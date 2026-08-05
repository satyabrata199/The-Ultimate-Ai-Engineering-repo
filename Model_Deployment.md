# Model Deployment

> Deploying an AI service is ordinary backend deployment plus five things that break the usual assumptions: minute-long cold starts, GPU scheduling, 40-second requests, long-lived streams, and cost per request measured in cents.

**Scope note.** The general mechanics — Dockerfiles, Kubernetes manifests, probes, graceful shutdown, CI/CD, secrets, the full observability stack — are covered in depth in **`BackendForAi.md` Ch 21**. This phase covers the **AI-specific delta** and assumes that baseline. Where a topic is purely standard, it's stated briefly with a pointer rather than repeated.

---

## 1. The Two-Tier Architecture

**The single most important deployment decision.**

```
 ❌ MODEL INSIDE THE API PROCESS
    4 workers × 8 GB model = 32 GB per pod
    Scaling the API means scaling GPUs
    Cold start: minutes
    One slow generation blocks a worker
    → the pod is stateful, expensive, and slow to scale

 ✔ TWO TIERS
    ┌──────────────────┐         ┌──────────────────────┐
    │  API pods        │  HTTP   │  Inference tier      │
    │  stateless       │────────►│  vLLM / TGI / SGLang │
    │  small (512 MB)  │         │  GPU, few pods       │
    │  scale on rps    │         │  scale on queue depth│
    │  start in 2 s    │         │  start in 3 min      │
    └──────────────────┘         └──────────────────────┘

    Each tier scales for its own reason. This is Twelve-Factor VI
    (stateless processes) applied to AI.
```

**Exception:** small CPU models (a 130 MB reranker, a tokenizer, spaCy) are fine in-process. The dividing question is *"do I want to scale my API and my inference for the same reason?"* If no, split them.

---

## 2. Containers

Standard multi-stage, non-root build (`BackendForAi.md` Ch 21 §21.4). The AI-specific parts:

```dockerfile
# ─── API tier: ordinary Python service, small and fast ───
FROM python:3.12-slim AS runtime
# ... standard non-root, multi-stage build ...
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--workers", "2", \
     "--proxy-headers", "--timeout-graceful-shutdown", "180"]
```

```dockerfile
# ─── Inference tier: GPU base image, weights NOT baked in ───
FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04
RUN apt-get update && apt-get install -y --no-install-recommends \
        python3.11 python3-pip && rm -rf /var/lib/apt/lists/*
RUN pip install vllm==0.6.3

# ⚠ DO NOT COPY MODEL WEIGHTS INTO THE IMAGE.
#   A 20 GB image takes 5–10 minutes to pull on every node.
#   Mount from a PVC, or download in an init container to a shared volume.
ENV HF_HOME=/models
VOLUME /models

CMD ["python3", "-m", "vllm.entrypoints.openai.api_server", \
     "--model", "/models/llama-3.1-8b-instruct", \
     "--gpu-memory-utilization", "0.90", \
     "--enable-prefix-caching", "--kv-cache-dtype", "fp8"]
```

**GPU container requirements:**
```
 • NVIDIA Container Toolkit on the host
 • A CUDA base image whose version matches the driver
 • `--gpus all` (Docker) or `nvidia.com/gpu: 1` (Kubernetes)
 • The NVIDIA device plugin installed in the cluster
 • Verify: docker run --gpus all <image> nvidia-smi
```

**Image size discipline:** a CUDA runtime image is ~2 GB before you add anything. Use `-runtime` not `-devel` (you don't need compilers at runtime), and keep weights out.

---

## 3. Docker Compose — Local Development

The value here is dev/prod parity: the same services, wired the same way.

```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://dev:dev@postgres:5432/app
      REDIS_URL: redis://redis:6379/0
      LLM_BASE_URL: http://vllm:8000/v1        # ← same shape as production
    depends_on:
      postgres: {condition: service_healthy}
      redis: {condition: service_started}

  postgres:
    image: pgvector/pgvector:pg16              # vector search in the same DB
    environment: {POSTGRES_PASSWORD: dev}
    healthcheck: {test: ["CMD-SHELL", "pg_isready -U postgres"], interval: 5s}
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis: {image: redis:7-alpine}

  qdrant:
    image: qdrant/qdrant:latest
    ports: ["6333:6333"]
    volumes: ["qdrant:/qdrant/storage"]

  vllm:                                        # optional — most devs use Ollama
    image: vllm/vllm-openai:latest
    command: ["--model", "meta-llama/Llama-3.2-1B-Instruct",
              "--max-model-len", "4096"]
    volumes: ["~/.cache/huggingface:/root/.cache/huggingface"]
    deploy:
      resources: {reservations: {devices: [{driver: nvidia, count: 1,
                                            capabilities: [gpu]}]}}
volumes: {pgdata: , qdrant: }
```

**Practical note:** most developers don't have a GPU. Point `LLM_BASE_URL` at Ollama locally and at vLLM in staging — both speak the OpenAI API, so nothing else changes.

---

## 4. Kubernetes — The AI-Specific Parts

Standard Deployment/Service/HPA/PDB as per `BackendForAi.md` Ch 21 §21.9. What differs:

```yaml
# ─── INFERENCE TIER ───
spec:
  template:
    spec:
      nodeSelector: {cloud.google.com/gke-accelerator: nvidia-l4}
      tolerations: [{key: nvidia.com/gpu, operator: Exists, effect: NoSchedule}]
      terminationGracePeriodSeconds: 300       # let generations finish
      initContainers:
        - name: fetch-weights                  # keeps weights out of the image
          image: amazon/aws-cli
          command: ["sh","-c","aws s3 sync s3://models/llama-3.1-8b /models/"]
          volumeMounts: [{name: models, mountPath: /models}]
      containers:
        - name: vllm
          resources:
            limits: {nvidia.com/gpu: 1}        # ⚠ GPUs are limits-only,
                                                #   never shared or fractional
            requests: {cpu: "4", memory: "32Gi"}
          startupProbe:                        # ⚠ MODEL LOAD TAKES MINUTES
            httpGet: {path: /health, port: 8000}
            failureThreshold: 60
            periodSeconds: 10                  # allows 10 minutes
          readinessProbe:
            httpGet: {path: /health, port: 8000}
            periodSeconds: 5
          livenessProbe:                       # generous — a busy GPU is slow
            httpGet: {path: /health, port: 8000}
            periodSeconds: 30
            failureThreshold: 5
          volumeMounts: [{name: models, mountPath: /models}]
      volumes: [{name: models, emptyDir: {sizeLimit: 50Gi}}]
```

```
 THE FOUR GPU RULES

 ① A GPU is not shared. One pod owns the device. No fractional requests.
    (MIG partitioning on A100/H100 is the exception, and it's a cluster-level
     configuration, not a per-pod one.)

 ② Cold start is minutes. Startup probes must allow for it, and
    scale-to-zero is usually wrong — keep a warm floor.

 ③ GPU nodes are expensive and often scarce. Use node affinity, taints, and
    a separate node pool. Never let CPU workloads schedule onto them.

 ④ Spot/preemptible GPUs are 60–90% cheaper but vanish with ~30 s notice.
    Fine for batch. For interactive serving, only behind on-demand capacity
    with graceful drain.
```

---

## 5. Autoscaling

**The AI-specific correction: standard signals don't work.**

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │ TIER          │ WRONG SIGNAL      │ RIGHT SIGNAL                     │
 ├───────────────┼───────────────────┼──────────────────────────────────┤
 │ API pods      │ CPU (sits at 10%  │ in-flight requests, or concurrent│
 │               │ while every worker│ streams                          │
 │               │ waits on the LLM) │                                  │
 ├───────────────┼───────────────────┼──────────────────────────────────┤
 │ Inference     │ GPU utilisation   │ pending/queued requests          │
 │               │ (lags badly; 100% │ (vLLM: num_requests_waiting)     │
 │               │ tells you nothing │                                  │
 │               │ about queue depth)│                                  │
 ├───────────────┼───────────────────┼──────────────────────────────────┤
 │ Workers       │ CPU               │ queue LATENCY (not depth —       │
 │               │                   │ depth ignores job duration)      │
 └───────────────┴───────────────────┴──────────────────────────────────┘
```

```yaml
# API tier — scale on concurrency
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 4
  maxReplicas: 60
  metrics:
    - type: Pods
      pods:
        metric: {name: http_requests_in_flight}
        target: {type: AverageValue, averageValue: "40"}
  behavior:
    scaleUp:   {stabilizationWindowSeconds: 30}     # react fast
    scaleDown: {stabilizationWindowSeconds: 300}    # descend slowly
---
# Inference tier — scale on queue, with a warm floor
spec:
  minReplicas: 2                                    # ⚠ NEVER 0 with 3-min starts
  maxReplicas: 8
  metrics:
    - type: Pods
      pods:
        metric: {name: vllm_num_requests_waiting}
        target: {type: AverageValue, averageValue: "5"}
  behavior:
    scaleUp:   {stabilizationWindowSeconds: 60,
                policies: [{type: Pods, value: 2, periodSeconds: 120}]}
    scaleDown: {stabilizationWindowSeconds: 900}    # GPUs are slow to replace
```

**KEDA** is often better than the raw HPA for queue-driven scaling (Redis/Kafka/SQS triggers, and genuine scale-to-zero for batch workers).

---

## 6. Health Checks

Three probes, three jobs — the semantics matter more here because start-up is slow.

```python
@router.get("/health/live")
async def liveness():
    """Is the PROCESS alive? Touches NOTHING.
    Failing this RESTARTS the pod — a DB blip must never cause a
    fleet-wide restart storm."""
    return {"status": "alive"}

@router.get("/health/ready")
async def readiness(request: Request, response: Response):
    """Can this pod SERVE? Failing removes it from the LB, no restart."""
    if not request.app.state.ready:
        response.status_code = 503
        return {"status": "draining"}
    checks = {}
    async def probe(name, coro):
        try:
            async with asyncio.timeout(2.0):        # a probe must never hang
                await coro
            checks[name] = "ok"
        except Exception as exc:
            checks[name] = f"error: {type(exc).__name__}"
    await asyncio.gather(
        probe("database", db_ping()),
        probe("redis", redis.ping()),
        probe("inference", http.get(f"{LLM_BASE_URL}/models", timeout=2)),
    )
    if any(v != "ok" for v in checks.values()):
        response.status_code = 503
    return {"status": "ready", "version": VERSION, "checks": checks}

@router.get("/health/startup")
async def startup(request: Request, response: Response):
    """For slow starts. K8s disables liveness/readiness until this passes,
    so a 3-minute model load isn't killed as unhealthy."""
    if not request.app.state.ready:
        response.status_code = 503
    return {"status": "started" if request.app.state.ready else "loading"}
```

**Warm up before declaring ready.** For an inference pod, run one real generation at startup — the first request otherwise pays CUDA graph compilation, and that request belongs to a user.

---

## 7. Reverse Proxy: NGINX and Traefik

The AI-specific requirement is **per-path configuration** — streaming, WebSockets, and JSON endpoints need different rules.

```nginx
upstream api { least_conn; server api:8000; keepalive 64; }

server {
    listen 443 ssl http2;                  # HTTP/2: escapes the browser's
                                            # 6-connections-per-origin cap

    client_max_body_size 50m;               # ⚠ default is 1m
    proxy_request_buffering off;            # stream uploads through

    location / {
        proxy_pass http://api;
        proxy_read_timeout 120s;
    }

    location ~ ^/v1/.*(stream|sse) {        # ← streaming needs DIFFERENT rules
        proxy_pass http://api;
        proxy_buffering off;                # THE critical directive
        proxy_cache off;
        gzip off;                           # compression defeats streaming
        proxy_read_timeout 600s;            # long generations
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    location /ws/ {
        proxy_pass http://api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
    }

    location /metrics { allow 10.0.0.0/8; deny all; proxy_pass http://api; }
}
```

```yaml
# Traefik equivalent — same concerns, declarative
labels:
  - "traefik.http.routers.stream.rule=PathRegexp(`/v1/.*stream`)"
  - "traefik.http.services.api.loadbalancer.responseforwarding.flushinterval=1ms"
  #                                                            ↑ = no buffering
  - "traefik.http.middlewares.slow.buffering.maxResponseBodyBytes=0"
```

**The default that catches everyone:** `proxy_buffering on` in NGINX. Streaming works perfectly on localhost and arrives all at once in production. `X-Accel-Buffering: no` as a response header is the application-side escape hatch when you can't edit the proxy config.

---

## 8. Secrets

Standard practice (`BackendForAi.md` Ch 21 §21.3). The AI-specific note:

```
 PROVIDER API KEYS ARE MONEY.

 ✔ Mounted secret files or a secret manager — not environment variables
   (visible in `ps`, /proc, crash dumps, and child processes)
 ✔ Separate keys per environment; never share prod keys with staging
 ✔ Per-key spend limits at the provider where supported
 ✔ Rotation rehearsed — support two valid keys during a rotation window
 ✔ A leaked key on GitHub is drained by automated scanners within minutes.
   Rotate; deleting the commit does nothing (git history is distributed).
```

---

## 9. CI/CD

Standard pipeline: test → build → scan → sign → migrate → canary → production, with automatic rollback (`BackendForAi.md` Ch 21 §21.11). The AI-specific additions:

```yaml
jobs:
  test:
    steps:
      - run: pytest tests/unit tests/integration -n auto
      # ⚠ NO REAL LLM CALLS IN CI — slow, costly, non-deterministic,
      #   and broken by the provider's bad afternoon. Stub them.

  build-api:                                # small, fast, every commit
    steps: [...standard multi-stage build...]

  build-inference:                          # large, slow, only on model changes
    if: contains(github.event.head_commit.modified, 'inference/')
    steps: [...CUDA image build...]

  evals:                                    # ⚠ NIGHTLY, NOT ON EVERY COMMIT
    schedule: "0 2 * * *"
    steps:
      - run: python -m evals.run --dataset golden --output metrics.json
      - run: python -m evals.compare --baseline main --threshold 0.05
      # SCORED, not pass/fail. Alerts on regression; doesn't block merges.

  deploy-canary:
    steps:
      - run: kubectl set image deployment/api-canary api=...:${{ github.sha }}
      - run: python scripts/verify_canary.py --duration 600 --error-budget 0.005
      # Watches error rate, p99, AND cost per request vs the stable deployment
```

**The AI-specific canary check is cost.** A prompt change that quietly doubles token usage passes every functional test and every latency check. Compare cost-per-request between canary and stable before promoting.

---

## 10. Observability

The three pillars, with the AI-specific fields that make them useful.

### Logging

```python
# One structured line per request. To stdout, never to files.
{"event": "request_completed", "timestamp": "2026-08-05T10:22:31Z",
 "service": "ai-platform", "version": "a1b2c3d",
 "request_id": "req_01HQ3M", "trace_id": "4bf92f35...",
 "route": "/v1/chat/completions", "status": 200, "duration_ms": 4213,
 "org_id": "org_789", "user_id": "usr_123",
 # ── the AI-specific fields ──
 "model": "gpt-4o-mini", "prompt_version": "rag-v3.2",
 "prompt_tokens": 1240, "completion_tokens": 380,
 "cost_usd": 0.000414, "cache_hit": false, "ttft_ms": 612,
 "finish_reason": "stop", "retrieved_chunks": 5}
```

**Cost per request in the log line makes "which customer is expensive?" a log query** instead of a data-warehouse project. Add it on day one.

**Prompt logging — with care:** you need prompts for debugging and evals, but they contain user data. Redact PII, sample rather than logging everything (prompts are large), store in a system with short retention and access control, and honour per-tenant no-retention flags. Enterprise contracts routinely require zero prompt retention; that has to be enforced in code.

**Loki** (or your log backend) works unchanged here — the discipline is in what you put in the line, not the tooling.

### Metrics

```python
# Standard RED
REQUESTS = Counter("http_requests_total", "", ["method", "route", "status"])
LATENCY  = Histogram("http_request_duration_seconds", "", ["method", "route"],
                     buckets=(.05,.1,.25,.5,1,2.5,5,10,30,60,120))
                     # ⚠ default buckets top out at 10s — useless for LLM calls

# AI-specific
LLM_TTFT   = Histogram("llm_ttft_seconds", "", ["model"],
                       buckets=(.1,.25,.5,.75,1,1.5,2,3,5,10,30))
LLM_TOKENS = Counter("llm_tokens_total", "", ["model", "direction"])
LLM_COST   = Counter("llm_cost_usd_total", "", ["model"])   # ⚠ org_id label =
                                                             # cardinality risk
LLM_ERRORS = Counter("llm_errors_total", "", ["provider", "error_code"])
CIRCUIT    = Gauge("circuit_breaker_open", "", ["provider"])
CACHE_HITS = Counter("cache_hits_total", "", ["type"])      # embedding|llm|semantic
STREAMS    = Gauge("sse_streams_active", "")
ABANDONED  = Counter("sse_streams_abandoned_total", "")

# Saturation
EVENT_LOOP_LAG = Histogram("event_loop_lag_seconds", "",
                           buckets=(.001,.005,.01,.025,.05,.1,.25,.5,1,5))
```

```
 THE FOUR GOLDEN SIGNALS, AI EDITION
   LATENCY      p50/p95/p99 — and TTFT tracked SEPARATELY from total
   TRAFFIC      requests/s, concurrent streams, tokens/s
   ERRORS       5xx rate, provider failure rate, circuit state
   SATURATION   event-loop lag, DB pool, queue depth, GPU/KV-cache utilisation

 + THE FIFTH, WHICH ONLY AI SYSTEMS HAVE: COST
```

### Tracing

```python
FastAPIInstrumentor.instrument_app(app, excluded_urls="/health.*,/metrics")
HTTPXClientInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument(engine=engine.sync_engine)

# Custom spans where the time actually goes
with tracer.start_as_current_span("rag.query") as span:
    with tracer.start_as_current_span("rag.embed"):    ...
    with tracer.start_as_current_span("rag.search") as s:
        s.set_attribute("rag.hits", len(hits))
    with tracer.start_as_current_span("llm.generate") as s:
        s.set_attributes({"llm.model": model, "llm.prompt_tokens": pt,
                          "llm.completion_tokens": ct, "llm.cost_usd": cost,
                          "llm.cached": cached})
```

**Sample at 10% for the baseline, but force 100% sampling on errors** — a 10% sample means 90% of your incidents have no trace.

**LLM-specific tracing tools** (Langfuse, LangSmith, Braintrust) capture prompts, completions, and token counts per span in a form built for debugging quality rather than latency. Use one alongside OpenTelemetry, not instead of it — they answer different questions.

### Alerts

```yaml
- alert: HighErrorRate
  expr: sum(rate(http_requests_total{status=~"5.."}[5m]))
        / sum(rate(http_requests_total[5m])) > 0.01
  for: 5m
  annotations: {runbook: "https://runbooks/high-error-rate"}

- alert: EventLoopBlocked                    # the most diagnostic async alert
  expr: histogram_quantile(0.99, rate(event_loop_lag_seconds_bucket[5m])) > 0.1
  for: 5m

- alert: AllProvidersDown
  expr: min(circuit_breaker_open) == 1
  for: 1m

- alert: CostSpike                           # ⚠ AI-SPECIFIC AND MANDATORY
  expr: sum(rate(llm_cost_usd_total[1h])) * 24
        > 2 * sum(rate(llm_cost_usd_total[1h] offset 1d)) * 24
  for: 30m
  annotations: {summary: "Projected daily LLM spend is 2× yesterday"}

- alert: InferenceQueueGrowing
  expr: vllm_num_requests_waiting > 20
  for: 5m
```

**The cost alert is the one nothing else catches.** A prompt loop, retry storm, or leaked key produces no errors, no latency change, and a five-figure invoice. It has paid for itself at many companies.

**Every paging alert needs a runbook link.** An alert with no action to take should be deleted — alert fatigue is how real incidents get ignored.

---

## Common Mistakes

```
❌ Model loaded inside the API process
❌ Model weights baked into the container image (5–10 min pulls)
❌ No startup probe → K8s kills the pod during model loading
❌ Scale-to-zero on GPU pods with minute-long cold starts
❌ Autoscaling on CPU (API) or GPU utilisation (inference) instead of queue
❌ No warmup request before readiness → the first user pays compilation
❌ proxy_buffering on → streaming works locally, breaks in production
❌ client_max_body_size left at NGINX's 1 MB default
❌ Latency histogram buckets topping out at 10 s
❌ No cost metric, no cost alert, no cost in the canary check
❌ Real LLM calls in CI
❌ Evals as pass/fail CI gates (flaky pipelines teams learn to ignore)
❌ Provider keys in environment variables rather than mounted secrets
❌ Logging full prompts with no redaction, sampling, or retention policy
❌ terminationGracePeriodSeconds shorter than the longest generation
```

---

## Checklist

```
ARCHITECTURE
[ ] API and inference in separate tiers, scaled independently
[ ] Model weights mounted, not baked into images
[ ] Small CPU models in-process only

CONTAINERS
[ ] Multi-stage, non-root, SHA-tagged, scanned, signed
[ ] CUDA runtime (not devel) base for GPU images
[ ] NVIDIA toolkit verified with nvidia-smi in-container

KUBERNETES
[ ] GPU node pool with taints; nodeSelector + tolerations
[ ] Startup probe with a generous failure threshold
[ ] Warmup generation before readiness
[ ] terminationGracePeriodSeconds > longest generation
[ ] Autoscale: API on in-flight requests, inference on queue depth
[ ] Warm floor > 0 on GPU pods

PROXY
[ ] Per-path rules: streaming (no buffering), WebSocket (upgrade), JSON
[ ] HTTP/2 enabled; least-connections for long-lived connections
[ ] client_max_body_size raised; /metrics network-restricted

CI/CD
[ ] No real LLM calls in CI; evals nightly and scored
[ ] Canary compares error rate, p99, AND cost per request
[ ] Automatic rollback on canary failure

OBSERVABILITY
[ ] Structured logs with request_id, model, prompt_version, tokens, cost
[ ] Prompt logging redacted, sampled, retention-bounded, tenant-flag-aware
[ ] Metrics: RED + TTFT + tokens + cost + cache + streams + loop lag
[ ] Histogram buckets sized for LLM latencies (to 120 s)
[ ] Traces with 10% sampling and forced sampling on errors
[ ] LLM-specific trace store (Langfuse/LangSmith) alongside OTel
[ ] Alerts: errors, loop lag, all-providers-down, COST SPIKE, queue depth
[ ] Every paging alert has a runbook
```

---

**Next:** Phase 20 — the distributed-systems patterns underneath, and which of them AI workloads actually stress.
