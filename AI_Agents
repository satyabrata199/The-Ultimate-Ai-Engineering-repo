# Phase 17 — AI Agents

> An agent is a loop: the model picks a tool, your code runs it, the result goes back, repeat until done. Everything else — planning, reflection, multi-agent, guardrails — is a variation on controlling that loop.

**After this phase you can:** build an agent that terminates, costs a predictable amount, can be interrupted and audited, and cannot be hijacked by a document it reads.

---

## 1. The Loop

```python
async def agent_loop(task: str, tools: dict, *, max_steps: int = 10,
                     max_cost_usd: float = 0.50) -> AgentResult:
    messages = [{"role": "system", "content": SYSTEM}, {"role": "user", "content": task}]
    cost = 0.0

    for step in range(max_steps):                      # ⚠ ALWAYS BOUNDED
        response = await llm.complete(messages, tools=tool_schemas(tools))
        cost += response.cost
        if cost > max_cost_usd:                        # ⚠ ALWAYS BUDGETED
            return AgentResult(status="budget_exceeded", steps=step)

        if not response.tool_calls:
            return AgentResult(status="complete", answer=response.content, steps=step)

        messages.append(response.message)
        for call in response.tool_calls:
            result = await execute_tool(call, tools)   # validate + authorize inside
            messages.append({"role": "tool", "tool_call_id": call.id,
                             "content": result})

    return AgentResult(status="max_steps_exceeded", steps=max_steps)
```

**That is genuinely the whole thing.** Frameworks add state persistence, streaming, and branching — but if you can't write this loop, you can't debug theirs.

**The three bounds are not optional.** An agent without a step cap, a cost cap, and a wall-clock timeout will eventually loop forever on some input, and you will find out from the invoice.

---

## 2. When You Actually Need an Agent

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │ ONE LLM CALL           the task is known and single-step             │
 │ FIXED PIPELINE         steps are known in advance, always the same   │
 │                        (RAG is a pipeline, not an agent)             │
 │ ROUTER                 pick one of N known paths, then execute       │
 │ AGENT                  the STEPS DEPEND ON INTERMEDIATE RESULTS      │
 └──────────────────────────────────────────────────────────────────────┘

 The test: "Can I draw the flowchart in advance?"
   Yes → build the flowchart. It's cheaper, faster, and debuggable.
   No  → you need an agent.
```

**Cost of the agent pattern, stated plainly:**

```
 Fixed pipeline:  1 LLM call,  ~1 s,   ~$0.001
 Agent (5 steps): 6 LLM calls, ~8 s,   ~$0.010    — and non-deterministic

 → 10× cost, 8× latency, and much harder to test.
   Use an agent when the flexibility is genuinely required.
```

---

## 3. Tool Use

**The most important part of an agent is its tools, not its prompt.**

```python
class SearchDocuments(BaseModel):
    """Search the user's uploaded documents for relevant passages.

    Use this when the user asks about the content of their own documents,
    reports, or contracts.

    Do NOT use this for general world knowledge, current events, or
    arithmetic — it can only answer from the indexed corpus."""

    query: str = Field(description="Natural-language search query")
    top_k: int = Field(default=5, ge=1, le=20,
                       description="Number of passages to retrieve")
```

**Tool design rules, in order of impact:**

```
 ① NEGATIVE GUIDANCE      "Do NOT use for..." reduces spurious calls more than
                          any amount of positive description. Models over-call.
 ② FEWER TOOLS            selection accuracy degrades past ~20–30 tools, and
                          every schema costs tokens on EVERY step.
                          Select a subset per task.
 ③ CONSTRAINED TYPES      Literal/enum/ge/le appear in the schema AND are
                          enforced on return. Two defences from one declaration.
 ④ ONE JOB PER TOOL       `search_documents` beats `do_document_stuff(action=...)`
 ⑤ USEFUL ERRORS          "No results for 'X'. Try broader terms." lets the
                          agent recover. "Error 500" makes it loop or give up.
 ⑥ IDEMPOTENT WHERE       agents retry. A non-idempotent tool called twice
    POSSIBLE              is a duplicate charge.
```

**Execution is where safety lives:**

```python
async def execute_tool(call: ToolCall, tools: dict, ctx: AgentContext) -> str:
    # ① Does the tool exist? (models hallucinate tool names)
    if call.name not in tools:
        return f"Error: unknown tool '{call.name}'. Available: {list(tools)}"

    # ② Validate arguments with the SAME schema that generated the definition
    try:
        args = TOOL_SCHEMAS[call.name].model_validate_json(call.arguments)
    except ValidationError as exc:
        return f"Error: invalid arguments — {exc.errors()[:3]}"   # agent can retry

    # ③ AUTHORIZE — the model proposes, your code decides.
    #    Effective authority = user permissions ∩ agent grant. Never a union.
    await authorize_tool_call(ctx.principal, ctx.grant, call.name, args)

    # ④ Destructive actions need a human
    if TOOL_META[call.name].destructive and not ctx.approved(call.id):
        raise RequiresApproval(call)

    # ⑤ Execute with a timeout — a hanging tool hangs the agent
    try:
        async with asyncio.timeout(TOOL_META[call.name].timeout_s):
            result = await tools[call.name](**args.model_dump(), ctx=ctx)
    except TimeoutError:
        return f"Error: {call.name} timed out. Try a narrower request."
    except Exception as exc:
        log.exception("tool_failed", tool=call.name)
        return f"Error: {call.name} failed — {type(exc).__name__}"

    # ⑥ Truncate — a 50 KB response destroys the context budget
    return truncate_for_context(result, max_tokens=2000)
```

**Return errors to the model as text, don't raise.** An agent that receives "no results, try broader terms" recovers. An exception ends the run.

---

## 4. Planning

Two approaches, with different trade-offs:

```
 ReAct (Reason + Act) — IMPLICIT PLANNING
   Think → Act → Observe → Think → ...
   ✔ Adapts to what it learns; simple; the default
   ✘ Can wander; no global view; hard to show the user a plan

 PLAN-AND-EXECUTE — EXPLICIT PLANNING
   Generate a full plan → execute each step → replan if a step fails
   ✔ Inspectable and approvable before anything runs
   ✔ Steps can run in PARALLEL when independent
   ✘ Plans made without information are often wrong
   ✘ Needs replanning logic, which is where the complexity goes
```

```python
class Plan(BaseModel):
    steps: list[PlanStep] = Field(max_length=10)      # ⚠ bound it

class PlanStep(BaseModel):
    id: int
    description: str
    tool: str
    depends_on: list[int] = Field(default_factory=list)   # → a task graph

async def plan_and_execute(task: str, ctx) -> AgentResult:
    plan = await llm.complete_structured(PLAN_PROMPT.format(task=task), schema=Plan)

    if requires_approval(plan):
        await ctx.request_approval(plan)               # human sees it FIRST

    results: dict[int, Any] = {}
    for layer in topological_layers(plan.steps):       # independent steps together
        outcomes = await asyncio.gather(
            *[execute_step(s, results, ctx) for s in layer], return_exceptions=True)
        for step, outcome in zip(layer, outcomes):
            if isinstance(outcome, Exception):
                plan = await replan(task, plan, step, outcome)   # bounded replans
                break
            results[step.id] = outcome
    return AgentResult(status="complete", results=results)
```

**Task graphs matter for latency.** A plan whose steps are independent should run them concurrently — three sequential 3-second tools is 9 seconds; in parallel it's 3.

**Hybrid is usually best:** a lightweight plan for structure and user visibility, ReAct within each step for adaptability.

---

## 5. Reflection

The agent critiques its own output and revises.

```python
async def with_reflection(task: str, max_rounds: int = 2) -> str:
    answer = await agent.run(task)
    for _ in range(max_rounds):
        critique = await llm.complete_structured(
            f"""Critique this answer against the task. Identify concrete
            problems: unsupported claims, missing requirements, errors.
            If it is good, say so.

            Task: {task}
            Answer: {answer}""", schema=Critique)
        if critique.is_acceptable:
            break
        answer = await agent.run(f"{task}\n\nRevise, addressing:\n{critique.issues}")
    return answer
```

```
 ✔ Genuinely helps: code (compile/test feedback), factual grounding, format
   compliance, and anything with an OBJECTIVE checker
 ✘ Weak for subjective quality — the model often "fixes" things that were fine,
   or declares its own output good regardless
 ✘ Doubles or triples cost and latency

 → The best reflection uses a REAL SIGNAL, not the model's opinion:
   run the tests, validate the schema, check the citations, execute the SQL.
   Self-critique with no external signal has sharply diminishing returns.
```

---

## 6. Memory in Agents

*(Full treatment: Phase 14.)* The agent-specific parts:

```
 SCRATCHPAD          within-run notes: tool results, what's been tried,
                     the current plan. ⚠ BOUND IT — an agent in a loop will
                     fill an unbounded scratchpad until the context overflows.

 STEP HISTORY        the message list itself. Grows fast — tool results are
                     usually the largest component. Compress older ones:
                     keep errors verbatim, summarise large successes.

 CROSS-RUN MEMORY    what worked before, user preferences, learned constraints.
                     Store as extracted FACTS, not transcripts.
```

```python
def compress_history(messages: list[Message], budget: int) -> list[Message]:
    """Keep: system, the original task, the last N steps verbatim.
    Compress: older tool results. Errors are never compressed — the agent
    needs them to avoid repeating a failing action."""
    head = [messages[0], messages[1]]                 # system + task
    tail = messages[-6:]                              # recent steps
    middle = messages[2:-6]
    if not middle:
        return head + tail
    summary = summarise_tool_results(middle, max_tokens=500)
    return head + [Message(role="system", content=f"Earlier steps:\n{summary}")] + tail
```

---

## 7. Multi-Agent

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │ THE HONEST ASSESSMENT                                                │
 │                                                                      │
 │ Multi-agent is the most over-applied pattern in the field.           │
 │                                                                      │
 │ ✘ Cost scales linearly with agent count                              │
 │ ✘ Latency compounds — 5 sequential agents = 5× wall clock            │
 │ ✘ Errors compound — each agent's mistakes feed the next              │
 │ ✘ Debugging is genuinely hard; failures emerge from interaction      │
 │ ✘ Agents "agreeing" with each other is not verification              │
 │                                                                      │
 │ ✔ Justified when agents need DIFFERENT TOOLS, DIFFERENT PERMISSIONS, │
 │   or when you want an ADVERSARIAL critic (generate → critique)       │
 │                                                                      │
 │ → Start with one agent and good tools. Add a second only when you    │
 │   can name the specific failure it fixes.                            │
 └──────────────────────────────────────────────────────────────────────┘
```

**Topologies:**

```
 SUPERVISOR (most useful)          PIPELINE
   supervisor decides who acts       A → B → C, fixed order
        │                            ✔ predictable, testable
   ┌────┼────┐                       ✘ no adaptivity — this is a pipeline,
   ▼    ▼    ▼                          not really a multi-agent system
  res  code  qa

 DEBATE / CRITIC                   GROUP CHAT
   generator ⇄ critic                everyone sees everything
   ✔ measurably improves quality     ✘ token cost explodes with participants
     on objective tasks              ✘ chaotic, hard to reproduce
```

**Where multi-agent genuinely earns its cost:** different *permission scopes*. A "reader" agent with read-only database access and a "writer" agent requiring approval is a real security boundary, not just an organisational metaphor.

---

## 8. Human in the Loop

```
 THREE PATTERNS

 ① APPROVAL GATE      pause before a destructive action, wait for a decision
 ② EDIT-AND-CONTINUE  human modifies the agent's proposal, then it proceeds
 ③ ESCALATION         low confidence or repeated failure → hand to a human
```

```python
async def request_approval(self, call: ToolCall, timeout_s: float = 300) -> bool:
    future = asyncio.get_running_loop().create_future()
    self._pending[call.id] = future
    await self._send(type="tool_request", id=call.id, name=call.name,
                     arguments=call.args, description=TOOL_META[call.name].description,
                     risk=TOOL_META[call.name].risk_level)
    try:
        async with asyncio.timeout(timeout_s):
            return await future
    except TimeoutError:
        self._pending.pop(call.id, None)
        return False                                   # ⚠ DEFAULT DENY
```

**Design rules:**
```
 ✔ Default DENY on timeout, always
 ✔ Show the human the ACTUAL arguments, not a paraphrase
 ✔ Approval ≠ authorization. Still run the authz check — a human approving
   something the user isn't permitted to do doesn't make it permitted.
 ✔ For waits longer than minutes, persist state (a durable workflow) rather
   than holding a connection
 ✔ Log both the principal and the approver
```

---

## 9. Guardrails

```
 INPUT GUARDRAILS (before the agent runs)
   ├─ prompt injection detection            (Phase 15 §13)
   ├─ PII detection and redaction
   ├─ off-topic / out-of-scope rejection
   └─ rate limits and budget checks         ← cheapest, run FIRST

 EXECUTION GUARDRAILS (during the loop)
   ├─ max steps, max cost, max wall clock   ← MANDATORY
   ├─ tool authorization on every call
   ├─ loop detection (same action repeated)
   └─ approval gates for destructive tools

 OUTPUT GUARDRAILS (before returning)
   ├─ schema validation
   ├─ groundedness / citation check
   ├─ PII leakage check
   └─ content policy
```

```python
class LoopDetector:
    """Agents get stuck repeating the same failing action. Detect and break."""

    def __init__(self, threshold: int = 3):
        self._seen: Counter[str] = Counter()
        self._threshold = threshold

    def check(self, call: ToolCall) -> str | None:
        key = f"{call.name}:{hashlib.sha256(call.arguments.encode()).hexdigest()[:12]}"
        self._seen[key] += 1
        if self._seen[key] >= self._threshold:
            return (f"You have called {call.name} with these exact arguments "
                    f"{self._seen[key]} times. It is not producing new information. "
                    f"Try a different approach or report what you found.")
        return None
```

**Order guardrails cheapest-first.** Running content moderation before a rate-limit check means paying for moderation on requests you were going to reject.

---

## 10. Workflows vs Agents

```
 ┌────────────────────────┬──────────────────────────────────────────────┐
 │ AGENT                  │ WORKFLOW                                     │
 ├────────────────────────┼──────────────────────────────────────────────┤
 │ model decides the steps│ you decide the steps                         │
 │ non-deterministic      │ deterministic                                │
 │ seconds to minutes     │ minutes to days                              │
 │ in-memory state        │ DURABLE state, survives deploys              │
 │ hard to test           │ testable                                     │
 │ LangGraph, PydanticAI  │ Temporal, Restate, or a task queue           │
 └────────────────────────┴──────────────────────────────────────────────┘

 THE RULE
   If a run must survive a deploy, or waits on a human for hours,
   or has more than ~3 sequential long steps → it is a WORKFLOW.
   Put the agent INSIDE a workflow step, not the other way round.
```

```python
# Durable workflow with an agent inside — the production shape for long runs
@workflow.defn
class ResearchWorkflow:
    @workflow.run
    async def run(self, request: ResearchRequest) -> Report:
        plan = await workflow.execute_activity(
            create_plan, request, start_to_close_timeout=timedelta(minutes=2))

        findings = []
        for step in plan.steps:
            result = await workflow.execute_activity(
                run_agent_step, step,                  # ← the agent lives here
                start_to_close_timeout=timedelta(minutes=10),
                retry_policy=RetryPolicy(maximum_attempts=3))
            findings.append(result)

            if result.needs_approval:
                # Sleeps for days, costs nothing, survives every deploy.
                approved = await workflow.wait_condition(
                    lambda: self._decision is not None, timeout=timedelta(days=2))
                if not approved:
                    return Report(status="rejected")

        return await workflow.execute_activity(synthesize, findings)
```

---

## 11. Scheduling

```
 TRIGGER TYPES
   on-demand     user asks → agent runs
   scheduled     cron: "summarise new documents each morning"
   event-driven  a webhook, a new upload, a threshold crossed
   recurring     "check this every hour until resolved"

 ⚠ SCHEDULED AGENTS ARE THE MOST DANGEROUS THING YOU WILL DEPLOY
   Nobody is watching. A loop bug runs 288 times a day.
   A prompt regression produces wrong output for a week before anyone notices.
```

```python
@cron("0 9 * * *")
async def daily_digest(ctx) -> None:
    for org in await orgs_with_digest_enabled():
        # ① Hard per-run budget
        if await spend_today(org.id) > org.plan.daily_budget:
            log.warning("digest_skipped_budget", org=org.id); continue
        # ② Timeout
        try:
            async with asyncio.timeout(300):
                result = await agent.run(DIGEST_TASK, ctx=AgentContext(
                    org=org, max_steps=8, max_cost_usd=0.20,
                    grant=DIGEST_GRANT))       # ③ read-only tools only
        except TimeoutError:
            log.error("digest_timeout", org=org.id); continue
        # ④ Output validated before it reaches anyone
        if issues := validate_digest(result):
            log.error("digest_invalid", org=org.id, issues=issues); continue
        await deliver(org, result)
```

**Scheduled agents need:** a hard budget, a timeout, a *reduced* tool grant (read-only where possible), output validation, and alerting on failure rate and cost. Treat them as unattended production jobs, because that is what they are.

---

## 12. MCP for Agents

*(Introduced in Phase 13.)* For agents specifically, MCP means your tools become portable:

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("company-tools")

@mcp.tool()
async def search_documents(query: str, top_k: int = 5) -> list[dict]:
    """Search the company knowledge base.

    Use for questions about internal documents.
    Do NOT use for general world knowledge."""
    return await store.search(query, top_k=top_k)
```

One server; usable by your agent, Claude Desktop, Cursor, and clients that don't exist yet.

```
 ⚠ MCP SECURITY IS YOUR RESPONSIBILITY
   • Authenticate remote servers (never expose one openly)
   • Authorize every call in YOUR code — MCP transports the request,
     it does not decide whether the caller may make it
   • Scope agent tokens to a SUBSET of the user's permissions
   • Audit-log principal AND acting agent (RFC 8693 `act` claim)
```

---

## 13. Observability

**Agents are the hardest thing to debug in AI engineering. Instrument from day one.**

```python
@dataclass
class AgentTrace:
    run_id: UUID
    task: str
    steps: list[StepTrace]        # per step: thought, tool, args, result, tokens, cost, ms
    total_cost_usd: float
    total_tokens: int
    duration_ms: int
    status: Literal["complete", "max_steps", "budget_exceeded", "error", "interrupted"]
    principal_id: str
    agent_grant_id: str
```

```
 METRICS THAT MATTER
   steps per run (p50/p95)      ← rising = degrading tool descriptions or prompts
   cost per run (p50/p95/max)   ← the max is what bankrupts you
   completion rate              ← fell? something changed upstream
   tool error rate by tool      ← which tool is failing the agent
   loop-detection triggers      ← a leading indicator of prompt regression
   approval rate / timeout rate
   time to first token          ← agents feel slow; stream the reasoning

 STREAM THE AGENT'S WORK
   A 45-second opaque wait feels broken. Streaming thinking → tool call →
   result → answer turns it into a legible process — and when the agent does
   something wrong, the user can see WHERE.
```

**Tools:** LangSmith, Langfuse, Braintrust, or OpenTelemetry spans per step. Any of them beats none.

---

## Common Mistakes

```
❌ No step limit, cost limit, or timeout → an infinite loop with a bill
❌ Multi-agent for a problem one agent solves
❌ Executing tool calls without independent authorization
❌ Raising exceptions from tools instead of returning recoverable error text
❌ Unbounded scratchpad / message history → context overflow mid-run
❌ Too many tools → the model picks the wrong one
❌ No "do not use for" guidance → constant spurious calls
❌ Self-critique with no external signal, mistaken for verification
❌ Building an agent when a fixed pipeline would do (10× cost for no gain)
❌ Long-running agents with in-memory state that a deploy destroys
❌ Scheduled agents with full permissions and no budget
❌ Approval treated as authorization
❌ No trace — an agent you can't replay is an agent you can't fix
❌ Truncating tool results without keeping errors verbatim
```

---

## Checklist

```
DESIGN
[ ] Confirmed an agent is needed (can't draw the flowchart in advance)
[ ] Tools: one job each, negative guidance, constrained types, useful errors
[ ] Tool count bounded (~20 max in context at once)
[ ] Fixed pipeline used wherever the steps are known

SAFETY
[ ] max_steps, max_cost, and wall-clock timeout on every run
[ ] Every tool call validated AND authorized in your own code
[ ] Agent grant ⊆ user permissions
[ ] Destructive tools require human approval; default deny on timeout
[ ] Loop detection with a recoverable message
[ ] Input, execution, and output guardrails, ordered cheapest-first

STATE
[ ] Scratchpad and history bounded in tokens
[ ] Tool results compressed; errors kept verbatim
[ ] Runs longer than minutes use durable workflow state

OPERATIONS
[ ] Full trace per run: steps, tools, args, tokens, cost, duration
[ ] Metrics: steps/run, cost/run, completion rate, tool error rate
[ ] Reasoning and tool use streamed to the user
[ ] Scheduled agents: reduced grant, hard budget, output validation, alerting
[ ] MCP servers authenticated; every call authorized
```

---

**Next:** Phase 18 — running the models yourself, where KV cache maths and batching decide your cost per token.
