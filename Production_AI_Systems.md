# Production AI Systems

> Everything from Phases 12–20 assembled into the shape a real product needs. This phase is the *system*: the request path, the safety layers around it, and the loops that keep it improving.

**After this phase you can:** design the full inference path with guardrails, routing, fallback, and cost control — and know which of those to build first.

---

## 1. The Request Path

```
 REQUEST
   │
   ├─ ① AUTH + RBAC ──────────────► 401/403        cost: ~0
   ├─ ② RATE LIMIT ───────────────► 429            cost: 1 Redis op
   ├─ ③ QUOTA / BUDGET ───────────► 402            cost: 1 Redis op
   ├─ ④ INPUT VALIDATION ─────────► 422            cost: µs
   ├─ ⑤ PII DETECTION ────────────► redact/reject  cost: ~5 ms
   ├─ ⑥ INJECTION / SAFETY ───────► 400            cost: ~50 ms + $
   │
   ├─ ⑦ CACHE LOOKUP ─────────────► HIT: return    cost: 1 Redis op
   │
   ├─ ⑧ SESSION + MEMORY ASSEMBLY                  cost: ~10 ms
   ├─ ⑨ RETRIEVAL (if RAG)                         cost: ~100 ms
   ├─ ⑩ PROMPT ASSEMBLY                            cost: ~5 ms
   │
   ├─ ⑪ MODEL ROUTING ────────────► pick a model
   ├─ ⑫ INFERENCE (+ retry, fallback, circuit breaker)  cost: 1–15 s, $$$
   │
   ├─ ⑬ OUTPUT VALIDATION ────────► retry or hedge
   ├─ ⑭ GROUNDEDNESS / SAFETY ────► block or flag
   ├─ ⑮ PII SCRUB (output)
   │
   ├─ ⑯ PERSIST: message, usage, cost, trace
   └─ ⑰ RESPONSE (streamed) + audit log + feedback hook

 ⚠ ORDER GATES CHEAPEST-FIRST. Running moderation (⑥) before the rate
   limit (②) means paying for moderation on requests you were going to
   reject. Under an abuse burst that is a real bill.
```

**Build order, if you're starting today:** ①②③④ then ⑫ with retry/fallback, then ⑦, then ⑬⑯. Everything else is refinement. A system with auth, limits, budgets, fallback, caching, and usage records is a real production system.

---

## 2. Prompt Pipeline

Prompts are code. Treat them accordingly.

```python
@dataclass(frozen=True)
class PromptTemplate:
    id: str                       # "rag_answer"
    version: str                  # "3.2"   ← goes in cache keys AND responses
    system: str
    user_template: str
    model_pins: dict[str, str]    # api_version → concrete dated model
    max_tokens: int
    temperature: float

def build_rag_prompt(tpl: PromptTemplate, *, query: str, chunks: list[Chunk],
                     history: list[Message], budget: int) -> list[Message]:
    """ORDER MATTERS FOR COST: stable content first so the provider's
    prefix cache engages (~10× cheaper on the cached prefix)."""
    messages = [{"role": "system", "content": tpl.system}]      # stable
    messages.append({"role": "system",
                     "content": render_references(chunks)})      # semi-stable
    messages += fit_to_budget(history, budget - used)            # variable
    messages.append({"role": "user", "content": query})          # variable
    return messages
```

```
 PROMPT DISCIPLINE
 ✔ Versioned, stored in the repo, reviewed like code
 ✔ Version in the cache key — otherwise a prompt change leaves every
   cached answer reflecting the old prompt, silently, for the TTL
 ✔ Version in a response header (X-Prompt-Version) for reproducibility
 ✔ Model PINNED to a dated version per API version. A floating alias
   silently becomes a different model: outputs shift, evals drift,
   downstream parsers break — with no deploy and no changelog entry.
 ✔ A/B tested and evaluated before rollout (§13, §14)
 ✔ Token budget enforced per section, measured not assumed
```

---

## 3. Guardrails

```
 INPUT                      EXECUTION                  OUTPUT
 ─────                      ─────────                  ──────
 PII detection              max steps / cost / time    schema validation
 injection scanning         tool authorization         groundedness check
 topic / scope check        loop detection             PII scrub
 rate + quota               approval gates             content policy
 size limits                circuit breakers           citation validation
```

```python
class Guardrails:
    async def check_input(self, text: str, ctx: RequestContext) -> InputVerdict:
        # Cheapest first — most rejections should cost nothing.
        if len(text) > ctx.plan.max_input_chars:
            return InputVerdict.reject("input_too_large")

        if findings := scan_injection_patterns(text):          # regex, ~0 ms
            log.warning("injection_pattern", patterns=findings, org=ctx.org.id)
            # Log and continue — patterns are evadable, so this is a SIGNAL,
            # not a gate. Authorization is what actually protects you.

        if pii := await detect_pii(text):                       # ~5 ms
            if ctx.org.pii_policy == "reject":
                return InputVerdict.reject("pii_detected", details=pii.types)
            text = redact(text, pii)                            # or redact

        if ctx.org.moderation_enabled:                          # ~50 ms + $
            if (mod := await moderate(text)).flagged:
                return InputVerdict.reject("content_policy", categories=mod.categories)

        return InputVerdict.allow(text)

    async def check_output(self, answer: str, chunks: list[Chunk],
                           ctx: RequestContext) -> OutputVerdict:
        if invalid := validate_citations(answer, chunks):
            return OutputVerdict.retry(f"invalid citations: {invalid}")
        if pii := await detect_pii(answer):
            answer = redact(answer, pii)                        # always scrub output
        if ctx.sample_for_groundedness():                       # sample in prod
            if (g := await check_grounded(answer, chunks)).score < 0.7:
                log.warning("low_groundedness", score=g.score)
                return OutputVerdict.hedge(answer, warning="low confidence")
        return OutputVerdict.allow(answer)
```

**Be honest about what guardrails achieve:** injection scanning and moderation reduce incidents. They do not prevent a determined attacker. **Authorization on tool execution is the control that actually holds** (Phase 17 §3).

---

## 4. PII Detection

```python
# Layer 1: deterministic patterns — fast, high precision, no cost
PII_PATTERNS = {
    "email":       r"\b[\w.+-]+@[\w-]+\.[\w.]+\b",
    "phone":       r"\b(?:\+?\d{1,3}[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b",
    "credit_card": r"\b(?:\d[ -]*?){13,16}\b",          # + Luhn check
    "ssn":         r"\b\d{3}-\d{2}-\d{4}\b",
    "api_key":     r"\b(sk-[A-Za-z0-9]{20,}|ghp_[A-Za-z0-9]{36})\b",
    "ip":          r"\b(?:\d{1,3}\.){3}\d{1,3}\b",
}

# Layer 2: NER for names, addresses, organisations (Presidio, spaCy) — ~20 ms
# Layer 3: an LLM classifier for context-dependent cases — slow, use sparingly
```

```
 WHERE PII APPEARS, AND WHAT TO DO
   User input          redact before sending to a provider (if policy requires)
   Retrieved chunks    the document may contain PII the user shouldn't see
   Model output        ALWAYS scrub — the model can echo training data or context
   Logs                redact in the logging pipeline, not at each call site
   Traces              same; APM tools capture bodies by default
   Evals               use synthetic data, or redacted production samples

 ⚠ REDACTION MUST SURVIVE CHUNK BOUNDARIES IN STREAMING.
   A credit card number can be split across two tokens. Buffer a sliding
   window (~200 chars) rather than scanning each chunk independently.

 ⚠ ZERO-RETENTION MODES exist at most providers and are a common enterprise
   contract requirement. That flag must be enforced in code, per tenant —
   not in a policy document.
```

---

## 5. Structured Outputs & Validation

*(Mechanics in Phase 12 §11.)* The production pattern:

```python
async def extract_validated(prompt: str, schema: type[BaseModel],
                            max_attempts: int = 3) -> BaseModel:
    """Retry with the validation errors fed back. This is the single
    highest-value reliability pattern in AI engineering:
    ~85% → ~99% success for ~20 lines."""
    messages = [{"role": "user", "content": prompt}]
    for attempt in range(max_attempts):
        raw = await llm.complete(messages, response_format=strict_schema(schema))
        try:
            result = schema.model_validate_json(raw)
            EXTRACTION_ATTEMPTS.observe(attempt + 1)
            return result
        except ValidationError as exc:
            if attempt == max_attempts - 1:
                EXTRACTION_FAILURES.inc()
                raise
            errors = "\n".join(f"- {'.'.join(map(str, e['loc']))}: {e['msg']}"
                               for e in exc.errors())
            messages += [{"role": "assistant", "content": raw},
                         {"role": "user", "content":
                          f"Validation errors:\n{errors}\nReturn corrected JSON only."}]
```

**Track average attempts as a metric.** If it drifts from 1.1 to 1.8, your schema has become harder for the model — usually after a provider model update. That's an early warning you get for free.

---

## 6. Model Routing

```python
class ModelRouter:
    """Route by task difficulty, plan, and cost. The single biggest
    cost lever after caching."""

    async def select(self, req: ChatRequest, ctx: RequestContext) -> str:
        # ① Explicit request, if permitted by the plan
        if req.model and req.model in ctx.plan.allowed_models:
            return req.model
        if req.model:
            raise PermissionDenied(f"Model '{req.model}' not on the {ctx.plan.name} plan")

        # ② Task-based routing — most requests don't need a frontier model
        task = classify_task(req)                    # cheap heuristic or classifier
        match task:
            case "extraction" | "classification" | "routing":
                return "small-fast"
            case "summarisation":
                return "small-fast" if req.input_tokens < 4000 else "mid"
            case "reasoning" | "code":
                return "reasoning" if ctx.plan.allows_reasoning else "large"
            case _:
                return ctx.plan.default_model

        # ③ Degraded mode — set by a feature flag during an incident
        # ④ Budget pressure — downgrade near the quota ceiling
```

```
 ROUTING SAVES MORE THAN ANY PROMPT OPTIMISATION.
   A frontier model is 20–50× the price of a small one. If 70% of your
   traffic is extraction, classification, and short summarisation, routing
   those to a small model cuts the bill by more than half — with no
   quality loss on those tasks.

 ⚠ VALIDATE THE ROUTING with evals per task type. A router that sends
   hard queries to a weak model is a quality regression that no latency
   or error metric will reveal.
```

---

## 7. Fallback

```python
class ResilientLLM:
    async def complete(self, req: ChatRequest) -> ChatResponse:
        errors = []
        for name, client in self._chain(req.model):
            if self._breakers[name].is_open:
                continue                                    # fail fast
            try:
                r = await client.complete(req)
                self._breakers[name].record_success()
                if errors:
                    FALLBACK_USED.labels(provider=name).inc()
                return r
            except (UpstreamError, UpstreamTimeout, RateLimited) as exc:
                self._breakers[name].record_failure()
                errors.append((name, exc))
                continue                                    # ✔ retryable → try next
            except AppError:
                raise           # ✘ 4xx: bad request, content filter, context length.
                                #   Every provider rejects it identically —
                                #   falling back pays 3× for the same failure.
        raise ServiceUnavailable("All providers unavailable")
```

```
 THE DEGRADATION LADDER
   primary model
     ↓ 5xx / 429
   SECONDARY PROVIDER          ← different outage domain. This is the key one:
     ↓                            a different vendor fails independently.
   cheaper model               ← lower quality, but an answer
     ↓
   RETRIEVAL-ONLY              ← "AI summarisation is temporarily unavailable;
     ↓                            here are the relevant passages"
   503 + Retry-After

 The retrieval-only tier is often the difference between a bad hour and
 an outage. Design it deliberately; it is not a fallback you improvise.

 ⚠ MID-STREAM YOU CANNOT FALL BACK. Once the first token is sent, the
   client has partial output. Fall back only BEFORE the first token;
   after it, emit an in-band error.
```

---

## 8. Caching

*(Full treatment: `BackendForAi.md` Ch 17.)* The AI-specific hierarchy and the traps:

```
 ① PROVIDER PREFIX CACHE   free. Just order the prompt stable-content-first.
                           ~10× cheaper on the cached prefix.
 ② EMBEDDING CACHE         deterministic, never invalidates, 40–70% hit rate.
 ③ RETRIEVAL CACHE         key = query hash + knowledge-base version.
 ④ EXACT RESPONSE CACHE    temperature 0 ONLY. Key must include:
                           model, messages, temperature, max_tokens,
                           PROMPT VERSION, and tool names.
 ⑤ SEMANTIC CACHE          ≥0.95 similarity, tenant-scoped, with exclusions.
```

```python
NEVER_SEMANTIC_CACHE = [
    r"\bmy\b", r"\bour\b",                                  # personalised
    r"\btoday\b", r"\bnow\b", r"\bcurrent\b", r"\blatest\b", # time-sensitive
    r"\b(balance|invoice|usage|quota|bill)\b",               # account-specific
]
```

```
 ⚠ THE SEMANTIC CACHE TRAP
   Embeddings do not encode negation. "Is X covered?" and "Is X NOT
   covered?" sit at ~0.97 similarity with opposite answers.

   OPTIMISE FOR PRECISION, NOT HIT RATE. A cache miss costs 4 seconds
   and a fraction of a cent. A false positive returns a confidently
   WRONG answer to a user. The asymmetry is enormous.

 ⚠ EVERY CACHE KEY NEEDS THE TENANT ID. A cache lookup bypasses every
   authorization check you wrote.
```

---

## 9. Session Management & Conversation Store

```python
# Postgres = source of truth. Append-only.
class Message(Base):
    id, conversation_id, org_id, user_id, role, content
    model, prompt_tokens, completion_tokens, cost_usd
    prompt_version, request_id, created_at
    # ⚠ store the MODEL and PROMPT VERSION per message — without them you
    #   cannot reproduce or explain an answer three months later

class Conversation(Base):
    id, org_id, user_id, title, created_at, updated_at, archived_at
    # title generated ASYNCHRONOUSLY after the first exchange

# Redis = hot path
#   session:{id}          active session state, 30-min TTL
#   stream:{id}           SSE event buffer for resumption, 1-hour TTL
#   memory:{conv_id}      assembled context cache, short TTL
```

```
 SESSION RULES
 ✔ Conversation state in Postgres, not in a process — any pod serves any request
 ✔ Assemble memory per request from the store (Phase 14), don't hold it in RAM
 ✔ Titles, embeddings, and summaries generated in background jobs
 ✔ Retention policy defined and ENFORCED — including vectors and summaries
 ✔ Export and delete endpoints (GDPR/CCPA), and deletion must actually delete
 ✔ Tenant-scoped repositories so an unscoped query isn't expressible
```

---

## 10. Cost Optimisation

**In rough order of return:**

```
 ① MODEL ROUTING              50–80% saving. Most traffic doesn't need a
                              frontier model.
 ② PROVIDER PREFIX CACHING    up to 90% on the cached prefix. Pure reordering.
 ③ EMBEDDING CACHE            near-100% saving on repeated text.
 ④ RERANK TO FEWER CHUNKS     send 5 instead of 20 → 4× fewer prompt tokens
                              AND better answers.
 ⑤ max_tokens CEILINGS        a runaway generation can be 10,000 tokens.
 ⑥ RESPONSE CACHE             for deterministic requests.
 ⑦ PROMPT COMPRESSION         trim boilerplate; measure before believing it.
 ⑧ BATCH APIs                 ~50% discount for non-urgent work
                              (evals, bulk enrichment, backfills).
 ⑨ SELF-HOSTING               only at steady high volume (Phase 18 §1).
```

```python
class CostGuard:
    """Three levels. Level ③ is the one that saves you."""

    async def check(self, org: Organization, estimated_usd: float) -> None:
        # ① per-request ceiling — catches a single pathological request
        if estimated_usd > org.plan.max_cost_per_request:
            raise ValidationError(f"Request would cost ${estimated_usd:.2f}")

        # ② per-org daily budget — atomic increment, refunded on rejection
        key = f"cost:{org.id}:{date.today()}"
        spent = float(await redis.incrbyfloat(key, estimated_usd))
        await redis.expire(key, 172800)
        if spent > org.plan.daily_budget_usd:
            await redis.incrbyfloat(key, -estimated_usd)
            raise QuotaExceeded(f"Daily budget ${org.plan.daily_budget_usd} exhausted")

        # ③ GLOBAL CIRCUIT BREAKER — the anomaly backstop
        if float(await redis.get("cost:global:today") or 0) > GLOBAL_DAILY_CAP:
            log.critical("global_budget_exceeded")
            await page_oncall("Global LLM budget exceeded")
            raise ServiceUnavailable("Service temporarily unavailable")
```

**Why level ③ matters most:** per-org limits assume the bug is in one org's usage. A prompt loop, retry storm, or leaked key affects *everything at once* — and only a global ceiling stops it before the invoice does.

---

## 11. Latency Optimisation

```
 A RAG REQUEST, DECOMPOSED
   query rewrite (if conversational)   150 ms   ← skip when unneeded
   embed query (cached 50%+)            20 ms
   hybrid retrieve (PARALLEL)           40 ms   ← TaskGroup, not sequential
   rerank 50 → 5                        50 ms
   prompt assembly                       5 ms
   ──────────────────────────────────────────
   pre-generation                      265 ms
   LLM TTFT                            600 ms
   ──────────────────────────────────────────
   FIRST TOKEN                        ~865 ms
```

```
 THE LEVERS, BY IMPACT
 ① STREAM                    8 s → 0.9 s PERCEIVED. Nothing else comes close.
 ② EMIT STATUS + CITATIONS   during the 265 ms of retrieval, so the screen
                             is never dead. Free perceived-quality win.
 ③ PARALLELISE               retrieval strategies, memory fetches, history —
                             TaskGroup, never sequential awaits.
 ④ CACHE                     embeddings, retrieval, prefixes.
 ⑤ SMALLER MODEL             often 3–5× faster TTFT for equal quality on
                             easy tasks.
 ⑥ FEWER OUTPUT TOKENS       output is sequential; ask for concision.
 ⑦ SHORTER PROMPTS           prefill is O(n²) — halving the prompt more than
                             halves TTFT.
 ⑧ PREFIX CACHING            skips prefill on the cached portion entirely.
```

**Measure TTFT separately from total latency.** They have different causes and different fixes, and a single latency histogram hides both.

---

## 12. RBAC & Audit Logs

*(Full treatment: `BackendForAi.md` Ch 6.)* The AI-specific dimensions:

```python
ROLE_PERMISSIONS = {
    Role.OWNER:  {"*"},
    Role.ADMIN:  {"documents:*", "models:*", "keys:*", "usage:read", "audit:read"},
    Role.MEMBER: {"documents:read", "documents:write", "chat:write", "usage:read"},
    Role.VIEWER: {"documents:read", "chat:read"},
}

# AI-SPECIFIC PERMISSION SURFACES
#   which MODELS a plan/role may use          (cost control + capability)
#   which KNOWLEDGE BASES are visible          (data access)
#   which TOOLS an agent may invoke            (blast radius)
#   whether PROMPTS are logged                 (privacy)
#   spend limits per key                        (financial)
```

```python
class AuditEvent(BaseModel):
    timestamp: datetime
    principal_id: str
    principal_type: Literal["user", "api_key", "agent", "system"]
    actor_id: str | None            # ← the AGENT acting on the user's behalf
    org_id: UUID
    action: str                     # "chat.complete", "tool.execute", "document.delete"
    resource_type: str
    resource_id: str | None
    effect: Literal["allow", "deny"]
    policy: str                     # WHICH rule decided — essential for debugging
    # AI-specific
    model: str | None
    prompt_version: str | None
    tokens: int | None
    cost_usd: Decimal | None
    request_id: str
```

```
 ✔ Append-only store the application cannot modify or delete
 ✔ Log DENIALS as loudly as successes — a denial spike is your earliest
   signal of an attack or a broken deploy
 ✔ Record both principal AND actor for agent actions (RFC 8693 `act`) —
   "why did this document get deleted?" needs both
 ✔ Retention matched to your compliance obligations
```

---

## 13. Feedback Loop

```python
class Feedback(BaseModel):
    message_id: UUID
    rating: Literal["up", "down"] | None
    reason: Literal["incorrect", "unhelpful", "unsafe", "slow", "other"] | None
    comment: str | None = Field(default=None, max_length=2000)
    corrected_answer: str | None = None      # gold — a labelled example
```

```
 THE LOOP THAT MAKES A PRODUCT IMPROVE

   user feedback ──► triage ──► categorise the failure
                                     │
      ┌──────────────┬───────────────┼───────────────┬──────────────┐
      ▼              ▼               ▼               ▼              ▼
   retrieval     prompt          model          missing         genuine
   miss          problem         limitation     content         ambiguity
      │              │               │               │              │
   tune chunking  revise prompt   route to a     ingest the     clarify in
   / reranking    (+ eval it)     stronger model document       the product
      │              │               │               │              │
      └──────────────┴───────────────┴───────────────┴──────────────┘
                                     ▼
                        ADD TO THE GOLDEN EVAL SET
                        so the fix is protected against regression

 ⚠ IMPLICIT SIGNALS ARE STRONGER THAN EXPLICIT ONES.
   Fewer than 1% of users click thumbs-down. Everyone abandons a bad answer.
   Track: stream abandonment rate, immediate rephrase rate, copy events,
   follow-up-question rate, and session length.
```

**The most valuable feedback artefact is `corrected_answer`** — a user-supplied gold label. Route those straight into the eval dataset.

---

## 14. A/B Testing

```python
class Experiment(BaseModel):
    id: str                        # "rag_prompt_v4"
    variants: dict[str, Variant]   # "control" | "treatment"
    allocation: dict[str, float]   # {"control": 0.9, "treatment": 0.1}
    unit: Literal["user", "org", "session"]
    metrics: list[str]

def assign(experiment: Experiment, unit_id: str) -> str:
    """Deterministic hash-based assignment: STICKY, and needs no storage.
    ⚠ Assign by USER or ORG, not per request — a user seeing two different
      prompt versions mid-conversation is a broken experience AND
      contaminates the measurement."""
    h = int(hashlib.sha256(f"{experiment.id}:{unit_id}".encode()).hexdigest()[:8], 16)
    point, cumulative = (h % 10_000) / 10_000, 0.0
    for variant, share in experiment.allocation.items():
        cumulative += share
        if point < cumulative:
            return variant
    return "control"
```

```
 WHAT TO A/B TEST                    METRICS TO COMPARE
   prompt versions                     thumbs up/down rate
   models                              abandonment rate
   retrieval strategies                follow-up-question rate (lower = better)
   chunk sizes                         TTFT and total latency
   temperature                         COST PER REQUEST      ← always include
   reranking on/off                    eval scores on the golden set

 ⚠ AI A/B TESTS NEED LARGE SAMPLES. Quality differences are small and
   variance is high. Run for weeks, not hours.

 ⚠ ALWAYS COMPARE COST. A treatment that improves satisfaction 2% while
   doubling spend is usually a bad trade — and only appears bad if you
   measured it.

 ✔ Combine with feature flags: deploy the code dark, enable per-org,
   watch, expand. A MODEL CHANGE IS A PRODUCT CHANGE and deserves the
   same care as a UI redesign.
```

---

## 15. Monitoring Dashboard

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │ HEALTH        error rate · p50/p95/p99 · TTFT · in-flight · streams  │
 ├──────────────────────────────────────────────────────────────────────┤
 │ QUALITY       thumbs up/down · abandonment · groundedness (sampled)  │
 │               retry rate · validation failure rate · eval scores      │
 ├──────────────────────────────────────────────────────────────────────┤
 │ COST          $/hour · $/request · $ by model · $ by top-10 orgs      │
 │               cache hit rate · savings from routing + caching         │
 ├──────────────────────────────────────────────────────────────────────┤
 │ CAPACITY      queue depth · KV-cache utilisation · DB pool            │
 │               provider rate-limit headroom · event-loop lag           │
 ├──────────────────────────────────────────────────────────────────────┤
 │ SAFETY        moderation blocks · injection detections · PII hits     │
 │               authz denials · approval timeouts                       │
 └──────────────────────────────────────────────────────────────────────┘
```

**The AI-specific alerts nothing else catches:**

```yaml
- alert: CostSpike                            # ← the most important AI alert
  expr: sum(rate(llm_cost_usd_total[1h])) * 24
        > 2 * sum(rate(llm_cost_usd_total[1h] offset 1d)) * 24
  for: 30m

- alert: QualityRegression
  expr: rate(feedback_negative_total[1h]) / rate(feedback_total[1h]) > 0.15
  for: 1h

- alert: AbandonmentSpike                     # users giving up
  expr: rate(sse_streams_abandoned_total[15m]) / rate(sse_streams_total[15m]) > 0.3
  for: 15m

- alert: ValidationFailureRate                # schema drift after a model update
  expr: rate(extraction_failures_total[1h]) / rate(extraction_total[1h]) > 0.05
  for: 30m

- alert: AllProvidersDown
  expr: min(circuit_breaker_open) == 1
  for: 1m
```

---

## Common Mistakes

```
❌ Gates ordered expensive-first — paying for moderation on rejected requests
❌ No global spend ceiling (only per-org limits)
❌ Floating model aliases — silent, undeployed behaviour changes
❌ Prompt version missing from cache keys → stale answers after a prompt change
❌ No model routing — paying frontier prices for classification
❌ Falling back on 4xx — paying 3× for the same failure
❌ Attempting fallback mid-stream after the first token
❌ Semantic cache tuned for hit rate → confidently wrong answers
❌ Cache keys without a tenant ID
❌ Usage recorded only on the success path → systematic under-billing
❌ No PII scrubbing on OUTPUT (models echo context and training data)
❌ Redaction that doesn't survive streaming chunk boundaries
❌ Model and prompt version not stored per message → answers unreproducible
❌ Relying only on thumbs-down (<1% of users) instead of implicit signals
❌ A/B assignment per request instead of per user → broken UX and contaminated data
❌ A/B tests that don't compare cost
❌ Guardrails treated as the защита against prompt injection instead of authorization
```

---

## Checklist

```
REQUEST PATH
[ ] Gates ordered cheapest-first: auth → limits → quota → validation → safety
[ ] Prompt versioned, in the repo, in cache keys and response headers
[ ] Models pinned to dated versions per API version
[ ] Token budget enforced per prompt section

RELIABILITY
[ ] Structured outputs with retry-on-validation (feed errors back)
[ ] Model routing by task, validated with per-task evals
[ ] Fallback on 5xx/429 only; degradation ladder to retrieval-only
[ ] Circuit breaker per provider
[ ] Fallback only before the first token; in-band errors after

COST
[ ] Three-level cost guard including a GLOBAL ceiling
[ ] max_tokens on every request
[ ] Prefix caching engaged (stable content first)
[ ] Embedding + retrieval + response caches
[ ] Cost per request logged, dashboarded, and alerted on

SAFETY
[ ] PII detected on input, retrieved content, and OUTPUT
[ ] Streaming redaction with a cross-chunk window
[ ] Zero-retention flag honoured per tenant, in code
[ ] Tool calls authorized independently of the model
[ ] RBAC covers models, knowledge bases, tools, and spend
[ ] Append-only audit log with principal AND actor

STATE
[ ] Conversation in Postgres; model + prompt version per message
[ ] Tenant-scoped repositories
[ ] Retention enforced across messages, vectors, and summaries
[ ] Export and delete endpoints that actually delete

IMPROVEMENT
[ ] Explicit and implicit feedback captured
[ ] Failures triaged into retrieval / prompt / model / content categories
[ ] Fixes added to the golden eval set
[ ] A/B by user or org, sticky, comparing quality AND cost
[ ] Model and prompt rollouts behind feature flags
```

---

**Next:** Phase 22 — how to know whether any of this is actually working.
