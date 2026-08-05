# Local & Self-Hosted LLMs

> Running the model yourself. The decision is almost never about capability — it's about data residency, cost at volume, and latency floor. This phase is the arithmetic and the tooling.

**After this phase you can:** decide whether self-hosting makes sense, size GPU memory correctly, pick a serving stack, and understand why batching is the difference between $10 and $0.30 per million tokens.

---

## 1. Should You Self-Host?

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ SELF-HOST WHEN                    │ USE AN API WHEN                      │
 ├───────────────────────────────────┼──────────────────────────────────────┤
 │ Data cannot leave your network    │ You're pre-product-market-fit        │
 │ (regulatory, contractual)         │                                      │
 │ Volume is high AND steady         │ Traffic is spiky or low              │
 │ (>~50M tokens/day)                │                                      │
 │ You need a fine-tuned model       │ You want frontier quality            │
 │ Latency floor matters (no network)│ You have no GPU ops capability       │
 │ You need guaranteed capacity      │ Time to market matters most          │
 └───────────────────────────────────┴──────────────────────────────────────┘
```

**The honest cost comparison:**

```
 An A100 80GB costs ~$1.50–2.50/hour on demand ≈ $1,300–1,800/month, 24/7.

 Serving Llama-3-8B on it with vLLM: ~2,500 output tokens/s at good batch
 utilisation → ~6.5 BILLION tokens/month if you keep it saturated.

   Self-hosted:  ~$0.25 per 1M output tokens   IF FULLY UTILISED
   GPT-4o-mini:  ~$0.60 per 1M output tokens

 ⚠ THE CATCH: at 10% utilisation your effective cost is ~$2.50/1M — 4× worse
   than the API, plus the engineer maintaining it.

 → SELF-HOSTING WINS ON *STEADY HIGH VOLUME*, NOT ON PRICE PER TOKEN.
   Spiky traffic on your own GPUs is the worst of both worlds.
```

**Also weigh:** an 8B open model is not GPT-4o. Compare on *your* evals, not on leaderboards.

---

## 2. GPU Memory — The Arithmetic

**Do this before you buy or rent anything.**

```
 TOTAL VRAM = weights + KV cache + activations + overhead

 ① WEIGHTS = params × bytes_per_param
      fp16/bf16  2 bytes    int8  1 byte    int4  0.5 bytes

      7B  model:  14 GB (fp16)   7 GB (int8)   3.5 GB (int4)
      13B model:  26 GB          13 GB         6.5 GB
      70B model: 140 GB          70 GB         35 GB

 ② KV CACHE (per token, per sequence) — see Phase 12 §4
      = 2 × layers × kv_heads × head_dim × bytes

      Llama-3-8B (32 layers, 8 KV heads via GQA, head_dim 128), fp16:
        131 KB per token
        × 8,000 tokens              =  1.0 GB per sequence
        × 32 concurrent sequences   = 33 GB       ← MORE THAN THE WEIGHTS

 ③ ACTIVATIONS + CUDA overhead: ~2–4 GB. Budget it.

 ④ FRAGMENTATION: without PagedAttention, 20–40% waste. With it, ~4%.
```

```python
def size_gpu(params_b: float, layers: int, kv_heads: int, head_dim: int,
             *, weight_bits: int = 16, kv_bits: int = 16,
             context: int = 8192, concurrent: int = 32) -> dict:
    weights = params_b * 1e9 * (weight_bits / 8)
    kv_per_token = 2 * layers * kv_heads * head_dim * (kv_bits / 8)
    kv = kv_per_token * context * concurrent
    overhead = 3e9
    total = weights + kv + overhead
    return {"weights_gb": weights/1e9, "kv_cache_gb": kv/1e9,
            "total_gb": total/1e9,
            "fits_a100_80": total < 80e9, "fits_l4_24": total < 24e9}

# Llama-3-8B, fp16 weights, 8k context, 32 concurrent:
#   weights 16 GB + KV 33 GB + 3 GB = 52 GB  → needs an A100 80GB
# Same model, int8 weights + fp8 KV:
#   weights 8 GB + KV 16.5 GB + 3 GB = 27.5 GB → fits an L40S 48GB comfortably
```

**The lesson: KV cache, not weights, decides your concurrency.** Halving context or quantising the KV cache buys more capacity than quantising weights.

**Multi-GPU:** tensor parallelism splits each layer across GPUs (low latency, needs fast interconnect — NVLink); pipeline parallelism splits layers across GPUs (higher latency, tolerates slower links). vLLM: `--tensor-parallel-size 4`.

---

## 3. Quantization

Reducing numeric precision to save memory and bandwidth.

| Method | Bits | Memory | Quality loss | Where |
|---|---|---|---|---|
| **fp16 / bf16** | 16 | baseline | none | the default |
| **fp8** | 8 | 2× less | very small | H100+ (native support) |
| **int8 (SmoothQuant)** | 8 | 2× less | small | GPU serving |
| **AWQ** | 4 | 4× less | small | **GPU serving — best 4-bit** |
| **GPTQ** | 4 | 4× less | small–moderate | GPU serving |
| **GGUF Q4_K_M** | ~4.5 | 3.5× less | small | **llama.cpp / CPU / Apple** |
| **GGUF Q8_0** | 8 | 2× less | negligible | llama.cpp, quality-first |

```
 GGUF NAMING (llama.cpp), decoded
   Q4_K_M  = 4-bit, K-quant, Medium  ← the sweet spot, use this
   Q5_K_M  = 5-bit, better quality, ~20% more memory
   Q8_0    = 8-bit, essentially lossless, 2× the memory of Q4
   Q2_K    = 2-bit — noticeably degraded. Avoid except in emergencies.

 RULE OF THUMB
   A LARGER model at Q4 usually beats a SMALLER model at fp16
   for the same memory budget.
     13B @ Q4 (~7 GB)  >  7B @ fp16 (~14 GB)   ← and uses half the memory
```

**Quality impact, honestly:** 8-bit is effectively free. 4-bit costs a small, measurable amount — often 1–3% on benchmarks, more on tasks needing precise reasoning or long-context recall. **Measure on your own evals**; the loss is task-dependent and benchmark averages hide it.

**KV cache quantization** (fp8/int8) is often the better lever — it doesn't touch weights and directly buys concurrency.

---

## 4. Serving Stacks

| Stack | Type | Best for | Throughput |
|---|---|---|---|
| **vLLM** | GPU server | **production serving — the default** | excellent |
| **SGLang** | GPU server | complex prompts, heavy prefix reuse | excellent |
| **TensorRT-LLM** | GPU server | maximum NVIDIA performance | best |
| **HF TGI** | GPU server | HF ecosystem integration | very good |
| **llama.cpp** | CPU/GPU/Metal | CPU, Apple Silicon, edge | modest |
| **Ollama** | dev wrapper | **local development** | modest |
| **LM Studio** | desktop GUI | experimentation, non-engineers | modest |

### Ollama — development

```bash
ollama pull llama3.1:8b
ollama run llama3.1:8b
# OpenAI-compatible endpoint at http://localhost:11434/v1
```
```python
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
```

**Use for:** local development, demos, offline work. **Not for:** production serving — it lacks continuous batching, so throughput collapses under concurrency.

### vLLM — production

```bash
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --served-model-name llama3 \
  --tensor-parallel-size 1 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90 \
  --enable-prefix-caching \
  --quantization awq \
  --kv-cache-dtype fp8
```

```
 KEY FLAGS
   --gpu-memory-utilization  fraction of VRAM vLLM may use (0.90 typical).
                             Higher = more KV cache = more concurrency.
   --max-model-len           caps context. LOWER THIS if you don't need long
                             contexts — it directly buys concurrency.
   --enable-prefix-caching   reuses KV for shared prompt prefixes.
                             Huge win for RAG (same system prompt every time).
   --kv-cache-dtype fp8      halves KV memory. Usually the best single flag.
   --max-num-seqs            cap on concurrent sequences.
```

**What makes vLLM fast:** PagedAttention (KV cache in non-contiguous blocks, near-zero fragmentation) + continuous batching (§5). Together, often 10–20× the throughput of naive serving.

**It speaks the OpenAI API**, so the `openai` SDK works with only a `base_url` change — and so does every framework in Phase 13.

### llama.cpp — CPU and Apple Silicon

```bash
./llama-server -m models/llama-3.1-8b-Q4_K_M.gguf \
  -c 8192 -ngl 99 --host 0.0.0.0 --port 8080
#  -ngl 99 = offload all layers to GPU/Metal; -ngl 0 = pure CPU
```

**The only serious option for CPU-only or Apple Silicon.** Unified memory on Apple Silicon means an M-series machine with 64 GB can run models a 24 GB GPU cannot.

### SGLang and TensorRT-LLM

**SGLang** — RadixAttention gives automatic, aggressive prefix-cache sharing across requests. Notably better than vLLM when many requests share long prefixes (multi-turn chat, RAG with a fixed system prompt, few-shot prompts).

**TensorRT-LLM** — compiles a model into optimised CUDA kernels. Fastest available on NVIDIA, at the cost of a slow per-model, per-GPU compilation step and much less flexibility. Choose it when you have one stable model and throughput is the whole game.

---

## 5. Batching — Where Throughput Comes From

**This is the single most important concept in self-hosted serving.**

```
 THE PHYSICS
   Decode is MEMORY-BANDWIDTH-BOUND, not compute-bound. The GPU spends
   most of its time reading model weights from VRAM.

   Generating 1 token for 1 sequence:  read ALL weights, do a little maths
   Generating 1 token for 32 sequences: read ALL weights ONCE, do 32× the maths

   → Batch size 32 gives ~25× the throughput for ~1× the weight-reading cost.
     Batching is nearly FREE THROUGHPUT until you run out of KV cache.
```

```
 STATIC BATCHING (naive)                CONTINUOUS BATCHING (vLLM, TGI, SGLang)
 ─────────────────────                  ──────────────────────────────────────
 Wait for N requests, run them           Requests join and leave the batch
 together, all wait for the SLOWEST.     EVERY ITERATION.

 R1 ████░░░░░░░░ (done, GPU IDLE)        R1 ████                (done, slot freed)
 R2 ████████████                         R2 ████████████
 R3 ██████░░░░░░ (done, GPU IDLE)        R3 ██████  R5 ██████   (R5 took the slot)
 R4 ████████░░░░ (done, GPU IDLE)        R4 ████████ R6 ████
    ↑ massive waste                         ↑ near-full utilisation

 → 2–5× more throughput, and much lower latency for short requests
   that would otherwise wait behind long ones.
```

**Consequences for how you operate it:**

```
 ✔ Concurrency is your throughput lever. One request at a time wastes ~95%
   of the GPU. Send many concurrent requests.
 ✔ KV cache size determines max batch size. This is why --max-model-len and
   --kv-cache-dtype matter so much.
 ✔ Queue depth, not GPU utilisation, is the autoscaling signal. GPU util
   is a lagging, misleading indicator.
 ⚠ Latency degrades gracefully as the batch fills, then sharply when the
   KV cache is exhausted and requests queue. Watch for the knee.
```

---

## 6. Prefix Caching

Reuse the KV cache for a shared prompt prefix across requests.

```
 EVERY RAG REQUEST LOOKS LIKE:
   [system prompt: 500 tokens]  ← IDENTICAL every time
   [retrieved chunks: 2000]     ← varies
   [question: 50]               ← varies

 With prefix caching, the 500-token system prompt is computed ONCE and
 reused. With a shared few-shot block or a large fixed instruction set,
 the saving is much larger.

   vLLM:   --enable-prefix-caching
   SGLang: RadixAttention (automatic, and more aggressive — it shares
           arbitrary common prefixes across requests, not just exact ones)
```

**This is the same principle as provider prompt caching (Phase 12/17): put stable content first.** Order your prompt so the cacheable prefix is as long as possible.

---

## 7. Speculative Decoding

A small "draft" model proposes several tokens; the large model verifies them in one forward pass.

```
 NORMAL DECODE            SPECULATIVE DECODE
 1 forward pass           draft model proposes 4 tokens (cheap)
   = 1 token              target model verifies all 4 in ONE pass
                          → accepts the longest correct prefix
                          → 1 pass yields 1–4 tokens

 Typical speedup: 1.5–3× on latency, with IDENTICAL output distribution.
```

```bash
vllm serve meta-llama/Llama-3.1-70B-Instruct \
  --speculative-model meta-llama/Llama-3.2-1B-Instruct \
  --num-speculative-tokens 4
```

```
 ✔ Output is mathematically identical to the target model — this is not
   a quality trade-off
 ✔ Best for LOW-BATCH, latency-sensitive serving
 ✘ Consumes extra compute; at HIGH batch sizes the GPU is already saturated,
   so the gain shrinks or disappears
 ✘ Needs a well-matched draft model (same tokenizer, similar distribution)

 VARIANTS: n-gram / prompt lookup (no draft model — copies from the prompt;
 excellent for summarisation and code editing where output echoes input),
 Medusa and EAGLE (extra prediction heads on the target model).
```

---

## 8. Deployment Shape

```yaml
# Inference pods are a different species from API pods
resources:
  limits: {nvidia.com/gpu: 1}
nodeSelector: {cloud.google.com/gke-accelerator: nvidia-l4}
tolerations: [{key: nvidia.com/gpu, operator: Exists, effect: NoSchedule}]

startupProbe:                          # ⚠ model load takes MINUTES
  httpGet: {path: /health, port: 8000}
  failureThreshold: 60
  periodSeconds: 10                    # allows 10 minutes
terminationGracePeriodSeconds: 300     # let in-flight generations finish
```

```
 GPU REALITIES THAT BREAK NORMAL ASSUMPTIONS

 • GPUs are NOT time-shared like CPUs. One pod owns the device.
 • Cold start is minutes: pull a 20 GB image, load weights, warm up CUDA.
   → Scale-to-zero is usually WRONG. Keep a warm floor.
 • Autoscale on QUEUE DEPTH or pending requests, never GPU utilisation.
 • Spot/preemptible GPUs are 60–90% cheaper but vanish with ~30 s notice.
   → Fine for batch jobs. Dangerous for interactive serving unless you
     drain gracefully and have on-demand capacity behind them.
 • Model weights in the container image make it enormous and slow to pull.
   → Mount from a PVC or object storage, or use an init container.
 • Always run a warmup request at startup before passing readiness —
   the first real request otherwise pays CUDA graph compilation.
```

**Architecture:** keep inference **separate from your API**. Stateless FastAPI pods (small, fast to scale) call a GPU inference tier (few, expensive, slow to scale). Never load a large model inside your API process — Phase 19 and `BackendForAi.md` Ch 1 both cover why.

---

## 9. Monitoring

```
 THE METRICS THAT MATTER
   TTFT p50/p95                     prefill + queue wait
   Inter-token latency (ITL)        decode speed; ~20+ tok/s feels smooth
   Throughput (output tokens/s)     the utilisation number
   Running / waiting requests       ← YOUR AUTOSCALING SIGNAL
   KV cache utilisation %           at 100%, requests start queueing
   Preemptions                      vLLM evicting sequences = you're overloaded
   GPU memory used                  should be stable; growth = a leak
   Prefix cache hit rate            if low, your prompts aren't structured well
```

vLLM exposes Prometheus metrics at `/metrics`. Alert on: KV cache utilisation sustained >90%, rising preemptions, waiting-request queue growth, and TTFT p95 above your target.

---

## 10. Choosing a Model

```
 SIZE                    TYPICAL USE
 1–3B    classification, extraction, routing, drafting for speculation
 7–9B    general chat, RAG answering, summarisation  ← the workhorse tier
 13–14B  better reasoning, still single-GPU friendly
 30–40B  strong general capability, needs 1–2 good GPUs
 70B+    approaching frontier quality, multi-GPU, expensive

 CHECK BEFORE COMMITTING
   [ ] Licence — some "open" models restrict commercial use or revenue scale
   [ ] Context window and whether it holds quality at the far end
   [ ] Tool-calling support (many small models are weak here)
   [ ] Quantized versions available (AWQ for GPU, GGUF for CPU)
   [ ] Your OWN eval scores — leaderboards do not transfer to your task
```

**Start with the smallest model that passes your evals.** A 7B that meets the bar is 10× cheaper and 5× faster than a 70B that exceeds it.

---

## Common Mistakes

```
❌ Self-hosting for spiky traffic — worst of both worlds
❌ Sizing VRAM for weights only, ignoring the KV cache (usually larger)
❌ Ollama in production — no continuous batching, throughput collapses
❌ Sending one request at a time — wastes ~95% of the GPU
❌ Autoscaling on GPU utilisation instead of queue depth
❌ Scale-to-zero with minutes-long cold starts
❌ No startup probe → Kubernetes kills the pod during model loading
❌ Loading the model inside your API process
❌ max-model-len set far above what you need — silently costs concurrency
❌ Not enabling prefix caching when every prompt shares a system prompt
❌ Q2 quantization to fit a bigger model — the quality loss is real
❌ Trusting leaderboards instead of evaluating on your own data
❌ No warmup request before readiness → the first user pays compilation cost
❌ Spot GPUs for interactive serving with no drain handling
```

---

## Checklist

```
DECISION
[ ] Utilisation-adjusted cost compared against API pricing
[ ] Model evaluated on YOUR data, not leaderboards
[ ] Licence checked for commercial use

SIZING
[ ] VRAM computed: weights + KV cache + activations + overhead
[ ] max-model-len set to what you actually need
[ ] KV cache quantization (fp8) considered before buying more GPU
[ ] Weight quantization chosen (AWQ for GPU, Q4_K_M for llama.cpp)

SERVING
[ ] vLLM/SGLang/TGI — never Ollama — in production
[ ] Continuous batching confirmed active
[ ] Prefix caching enabled; prompts ordered stable-content-first
[ ] Speculative decoding evaluated if latency-bound at low batch
[ ] OpenAI-compatible endpoint so clients need only a base_url change

OPERATIONS
[ ] Startup probe with a generous failure threshold
[ ] Warmup request before readiness
[ ] Weights mounted, not baked into the image
[ ] Autoscale on queue depth / pending requests; warm floor > 0
[ ] Metrics: TTFT, ITL, throughput, KV utilisation, preemptions, queue depth
[ ] Graceful drain on SIGTERM; grace period > longest generation
[ ] Inference tier separate from the API tier
```

---

**Next:** Phase 19 — deploying all of this. Much of it is standard backend deployment; this phase covers the AI-specific delta.
