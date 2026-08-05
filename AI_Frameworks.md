# AI Frameworks

> There are ~15 frameworks competing for the same job. This phase is about knowing what each one actually does, which three you should learn, and when writing 200 lines yourself beats adopting any of them.

**After this phase you can:** pick a framework for a given problem and defend the choice, avoid the abstraction traps, and recognise when the answer is "no framework."

---

## 1. The Question to Ask First

```
 WHAT ARE YOU ACTUALLY BUILDING?

 ┌────────────────────────────────────────────────────────────────────────┐
 │ One LLM call with structured output                                    │
 │   → Provider SDK + Instructor (or plain Pydantic). NO framework.       │
 ├────────────────────────────────────────────────────────────────────────┤
 │ RAG over your own documents                                            │
 │   → LlamaIndex, or write it yourself. It's ~300 lines.                 │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Multi-step agent with branching, state, and human approval             │
 │   → LangGraph. This is where a framework genuinely earns its weight.   │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Many providers, cost tracking, key management, fallback                │
 │   → LiteLLM (as a proxy, not just a library).                          │
 ├────────────────────────────────────────────────────────────────────────┤
 │ Exposing your tools/data to Claude Desktop, Cursor, or any MCP client  │
 │   → MCP server. It's a protocol, not a framework — different category. │
 └────────────────────────────────────────────────────────────────────────┘
```

**The honest position:** a RAG pipeline is retrieve → rerank → prompt → call → parse. That is a few hundred lines you fully understand. A framework buys you connectors and speed of prototyping; it costs you debuggability, a dependency with a fast-moving API, and an abstraction between you and the prompt actually sent.

**The rule:** use a framework when it manages *state and control flow you'd otherwise hand-roll* (LangGraph, MCP). Be sceptical when it merely wraps an HTTP call you already understand.

**Always be able to see the final prompt.** Any framework that makes this hard is disqualifying — you will need it during your first quality incident.

---

## 2. The Landscape

```
 ORCHESTRATION            DATA / RAG           STRUCTURE            ACCESS
 ─────────────            ──────────           ─────────            ──────
 LangChain                LlamaIndex           Instructor           OpenAI SDK
 LangGraph  ★             Haystack             PydanticAI  ★        Anthropic SDK
 AutoGen                  (LangChain retr.)    Outlines             Google GenAI SDK
 CrewAI                                                             LiteLLM  ★
 Semantic Kernel
 Agno                     OPTIMISATION         PROTOCOL
                          DSPy                 MCP  ★

 ★ = learn these
```

---

## 3. LangChain

**What it is.** The largest integration surface in the ecosystem — hundreds of connectors for models, vector stores, loaders, and tools — plus LCEL (LangChain Expression Language) for composing them with a `|` operator.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (ChatPromptTemplate.from_template("Summarise: {text}")
         | ChatOpenAI(model="gpt-4o-mini")
         | StrOutputParser())
result = await chain.ainvoke({"text": document})
```

**Use it for:** rapid prototyping, and when you need an integration that already exists (a niche loader, an unusual vector store).

**Be careful because:**
- The abstraction stack can be deep. Debugging "why did the model get *that* prompt?" means reading framework internals.
- The API has changed substantially across versions; much online material is stale.
- The `langchain` / `langchain-core` / `langchain-community` / provider-package split confuses dependency management.

**The pragmatic pattern:** use LangChain's *loaders and splitters* (genuinely useful, well-maintained) and write your own orchestration.

---

## 4. LangGraph ★

**What it is.** Stateful orchestration as an explicit graph: nodes are functions, edges are transitions (including conditional ones), and state is checkpointed after every node.

**This is the framework that earns its place**, because it solves things you would otherwise build badly:

```
 ✔ CYCLES        agent loops, retry-until-valid, reflection
 ✔ CHECKPOINTING resume after a crash or a deploy
 ✔ HUMAN-IN-LOOP interrupt, wait for approval (possibly for days), resume
 ✔ STREAMING     per-node output, so you can stream reasoning and tool calls
 ✔ TIME TRAVEL   rewind to a prior state and take a different branch
```

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver

class State(TypedDict):
    messages: Annotated[list, add_messages]
    documents: list[str]
    attempts: int

def retrieve(state): ...
def generate(state): ...
def should_retry(state) -> str:
    return "retrieve" if state["attempts"] < 3 and not grounded(state) else END

graph = StateGraph(State)
graph.add_node("retrieve", retrieve)
graph.add_node("generate", generate)
graph.set_entry_point("retrieve")
graph.add_edge("retrieve", "generate")
graph.add_conditional_edges("generate", should_retry)

app = graph.compile(
    checkpointer=PostgresSaver(conn),        # durable state
    interrupt_before=["execute_tool"])        # human approval gate
```

**Use it for:** anything with loops, branching, or a run that must survive a restart.
**Don't use it for:** a linear chain. A function is clearer.

**Note the boundary with Phase 20:** for runs measured in *hours or days* with external side effects, a durable workflow engine (Temporal) is a better fit. LangGraph is excellent for minutes-scale agent runs.

---

## 5. LlamaIndex

**What it is.** The RAG-focused framework. Document connectors, ingestion pipelines, index types, and query engines.

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core.node_parser import SentenceSplitter

docs = SimpleDirectoryReader("./data").load_data()
index = VectorStoreIndex.from_documents(
    docs, transformations=[SentenceSplitter(chunk_size=512, chunk_overlap=64)])
response = index.as_query_engine(similarity_top_k=5).query("What was Q3 revenue?")
```

**Genuinely strong at:**
- **Document connectors** (LlamaHub) — hundreds of sources, well-maintained.
- **Advanced retrieval strategies** as first-class objects: parent-document, auto-merging, recursive, sub-question decomposition. Implementing these yourself is real work (Phase 15).
- Ingestion pipelines with caching and deduplication.

**Watch out for:** the same "what prompt was actually sent?" opacity, and defaults tuned for demos rather than production (chunk sizes, top-k, no reranking).

**Verdict:** the best starting point if RAG *is* your product. Use its retrieval abstractions; consider owning the generation step yourself.

---

## 6. DSPy

**What it is.** Genuinely different from everything else here. You declare *signatures* (input → output behaviour) and DSPy **compiles** the prompts and few-shot examples by optimising against a metric on your training set.

```python
import dspy

class GenerateAnswer(dspy.Signature):
    """Answer questions using the provided context."""
    context = dspy.InputField(desc="relevant passages")
    question = dspy.InputField()
    answer = dspy.OutputField(desc="short, factual, grounded in the context")

class RAG(dspy.Module):
    def __init__(self):
        self.retrieve = dspy.Retrieve(k=5)
        self.generate = dspy.ChainOfThought(GenerateAnswer)
    def forward(self, question):
        ctx = self.retrieve(question).passages
        return self.generate(context=ctx, question=question)

# The point: prompts are OPTIMISED, not hand-written
compiled = dspy.BootstrapFewShot(metric=answer_exact_match).compile(
    RAG(), trainset=train_examples)
```

**Use it when:** you have a labelled dataset and a measurable metric, and you're currently hand-tuning prompts by intuition. DSPy replaces prompt-craft with optimisation.

**Don't use it when:** you have no eval set. Its entire value depends on a metric to optimise against.

**Even if you never adopt it, the idea is worth internalising:** prompts should be *optimised against a metric*, not hand-written and hoped over. That mindset belongs in Phase 22.

---

## 7. Haystack

**What it is.** A production-oriented pipeline framework from deepset with a strong classical-NLP heritage (it predates the LLM wave, which shows in good ways).

```python
from haystack import Pipeline
pipe = Pipeline()
pipe.add_component("embedder", SentenceTransformersTextEmbedder())
pipe.add_component("retriever", InMemoryEmbeddingRetriever(document_store))
pipe.add_component("ranker", TransformersSimilarityRanker())
pipe.add_component("generator", OpenAIGenerator())
pipe.connect("embedder.embedding", "retriever.query_embedding")
```

**Strengths:** explicit, inspectable pipelines with typed component connections; serialisable to YAML; mature evaluation tooling; strong hybrid search out of the box.

**Choose it over LangChain when** you want explicitness and production discipline over breadth of integrations. It's smaller but more predictable.

---

## 8. Multi-Agent Frameworks: AutoGen, CrewAI, Semantic Kernel, Agno

| Framework | Model | Best for |
|---|---|---|
| **AutoGen** (Microsoft) | agents converse in a group chat; a manager routes turns | research, code generation with a critic loop |
| **CrewAI** | role + goal + backstory; sequential or hierarchical processes | quick, readable multi-role prototypes |
| **Semantic Kernel** (Microsoft) | plugins + planners; .NET-first, Python second | enterprise .NET shops, Azure integration |
| **Agno** | lightweight, fast agent primitives; minimal abstraction | when you want agents without a heavy runtime |

**The honest assessment of multi-agent systems:**

```
 Multi-agent is fashionable and usually WRONG for a first version.

 ❌ Costs scale with agent count — every agent call is a full LLM call
 ❌ Latency compounds — 5 agents in sequence is 5× the wall clock
 ❌ Debugging is genuinely hard — failures emerge from interaction
 ❌ Agents talking to agents amplifies hallucination
 ❌ Non-determinism multiplies

 ✔ Genuinely useful when roles need DIFFERENT TOOLS, DIFFERENT PERMISSIONS,
   or an adversarial critic (generate → critique → revise)

 → Start with one agent and good tools. Add a second only when you can
   name the specific failure it fixes.
```

---

## 9. Provider SDKs: OpenAI, Anthropic, Google GenAI

**Learn at least one properly.** Every framework is a wrapper over these, and when something breaks you debug at this level.

```python
# OpenAI
from openai import AsyncOpenAI
client = AsyncOpenAI(api_key=..., base_url=...)   # base_url → any compatible server
r = await client.chat.completions.create(
    model="gpt-4o-mini", messages=[...], tools=[...], stream=True,
    stream_options={"include_usage": True})       # ⚠ or you get NO token counts

# Anthropic
from anthropic import AsyncAnthropic
client = AsyncAnthropic(api_key=...)
async with client.messages.stream(
    model="claude-sonnet-4-5", max_tokens=1024,   # ⚠ max_tokens is REQUIRED
    system="...",                                  # ⚠ system is a top-level param,
    messages=[...]) as stream:                     #    not a message role
    async for text in stream.text_stream: ...

# Google GenAI
from google import genai
client = genai.Client(api_key=...)
r = client.models.generate_content(model="gemini-2.0-flash", contents="...")
```

**Differences that cause real bugs:**

| | OpenAI | Anthropic | Google |
|---|---|---|---|
| System prompt | a message with `role: "system"` | a top-level `system` parameter | `system_instruction` |
| `max_tokens` | optional | **required** | optional |
| Stream end | `data: [DONE]` | `message_stop` event (**no** `[DONE]`) | iterator ends |
| Streamed usage | needs `stream_options` | split across two events | in the final chunk |
| Tool result role | `role: "tool"` | a `user` message with `tool_result` content | `function_response` |

**The OpenAI-compatible API is the de-facto standard.** vLLM, Ollama, Together, Groq, and most self-hosted servers speak it. Point the OpenAI SDK at a different `base_url` and everything works.

---

## 10. LiteLLM ★

**What it is.** One interface over 100+ providers — and, more importantly, a **proxy server** with keys, budgets, routing, and fallback.

```python
from litellm import acompletion
r = await acompletion(model="claude-sonnet-4-5", messages=[...])   # same call shape
r = await acompletion(model="gpt-4o-mini", messages=[...])
r = await acompletion(model="ollama/llama3", messages=[...])
```

**The proxy is the real product:**

```yaml
model_list:
  - model_name: fast
    litellm_params: {model: gpt-4o-mini, api_key: os.environ/OPENAI_API_KEY}
  - model_name: fast                          # same alias = automatic fallback
    litellm_params: {model: claude-haiku-4-5, api_key: os.environ/ANTHROPIC_KEY}
router_settings:
  routing_strategy: latency-based-routing
  num_retries: 2
  fallbacks: [{fast: ["smart"]}]
general_settings:
  master_key: sk-...
  database_url: postgres://...        # per-key budgets, spend tracking, rate limits
```

**Use it when:** you support multiple providers, need per-team virtual keys with budgets, or want centralised spend visibility without building it. It gives you Phase 21's model routing and cost control for near-zero effort.

**Trade-off:** another hop in the request path, and one more component to make highly available.

---

## 11. Instructor & PydanticAI ★

**Instructor** — structured outputs with automatic retry on validation failure.

```python
import instructor
from pydantic import BaseModel, Field

class Invoice(BaseModel):
    total: Decimal = Field(gt=0)
    date: date
    line_items: list[LineItem]

client = instructor.from_openai(AsyncOpenAI())
invoice = await client.chat.completions.create(
    model="gpt-4o-mini", response_model=Invoice,
    max_retries=3,                     # ← feeds validation errors BACK to the model
    messages=[{"role": "user", "content": document_text}])
# `invoice` is a validated Invoice, or it raised.
```

**This is the single highest-value small library in the list.** The retry-with-errors loop takes structured extraction from ~85% to ~99% success for a few lines of code.

**PydanticAI** — a type-safe agent framework from the Pydantic team.

```python
from pydantic_ai import Agent, RunContext

agent = Agent("openai:gpt-4o-mini", deps_type=Deps, result_type=Answer,
              system_prompt="Answer using the search tool.")

@agent.tool
async def search(ctx: RunContext[Deps], query: str) -> list[str]:
    return await ctx.deps.vector_store.search(query)

result = await agent.run("What was Q3 revenue?", deps=deps)
result.data          # a validated Answer instance
```

**Choose PydanticAI when** you want agents with real type safety, dependency injection, and a small surface area — and you already think in Pydantic. It fits a FastAPI codebase naturally.

---

## 12. Model Context Protocol (MCP) ★

**What it is.** An open protocol (from Anthropic, now broadly adopted) for connecting tools and data sources to LLM applications. **This is a standard, not a framework** — and it's the most strategically important item in this phase.

```
 BEFORE MCP                          WITH MCP
 Every app writes custom             Write ONE MCP server.
 integrations for every tool.        Every MCP client can use it:
 N apps × M tools = N×M work.        Claude Desktop, Cursor, your own agent…
                                     N + M instead of N×M.
```

**Three primitives:**

| Primitive | Meaning | Controlled by |
|---|---|---|
| **Tools** | functions the model can call | the model |
| **Resources** | data the client can read (files, records) | the application |
| **Prompts** | reusable templates the user can invoke | the user |

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("company-knowledge")

@mcp.tool()
async def search_documents(query: str, top_k: int = 5) -> list[dict]:
    """Search the company knowledge base.

    Use for questions about internal documents.
    Do NOT use for general world knowledge."""
    return await vector_store.search(query, top_k=top_k)

@mcp.resource("docs://{doc_id}")
async def get_document(doc_id: str) -> str:
    """Fetch the full text of a document."""
    return await repo.get_text(doc_id)

if __name__ == "__main__":
    mcp.run(transport="stdio")     # or "sse" / streamable HTTP for remote
```

**Transports:** `stdio` (local subprocess — simplest, most common), and HTTP/SSE (remote servers).

**Security, which the spec does not solve for you:**
```
 ⚠ An MCP server exposes real capability to a model influenced by untrusted data.
   • Authenticate remote servers; never expose one unauthenticated.
   • Authorize every tool call in YOUR code — the model proposes, you decide.
   • Scope tokens to the minimum; an agent's authority ⊆ the user's authority.
   • Destructive tools require human confirmation.
   • Audit-log every invocation with both principal and acting agent.
```

**Why it matters:** MCP is becoming the standard integration layer. Building your capabilities as an MCP server means they work with clients that don't exist yet.

---

## 13. Choosing: The Decision Table

| Your situation | Use |
|---|---|
| One call, structured output | Provider SDK + **Instructor** |
| RAG is the product | **LlamaIndex** (retrieval) + your own generation |
| Agent with loops, state, approval | **LangGraph** |
| Type-safe agents in a FastAPI codebase | **PydanticAI** |
| Multiple providers, budgets, fallback | **LiteLLM proxy** |
| Prompts tuned by hand, and you have evals | **DSPy** |
| Explicit, inspectable production pipelines | **Haystack** |
| Exposing tools to any LLM client | **MCP server** |
| Multi-agent with distinct roles and tools | **LangGraph** or CrewAI — after proving one agent isn't enough |
| Enterprise .NET | **Semantic Kernel** |
| You understand the problem completely | **No framework.** Write it. |

---

## Common Mistakes

```
❌ Adopting a framework before understanding the underlying API calls
❌ Choosing multi-agent for a problem one agent solves — cost, latency, and
   debuggability all get worse
❌ Using a framework that hides the final prompt from you
❌ Chaining framework abstractions until nobody can trace the control flow
❌ Believing a framework's defaults are production-tuned (they're demo-tuned)
❌ Pinning nothing — these libraries make breaking changes on minor versions
❌ Using DSPy without an eval set (its whole value is the metric)
❌ Exposing an MCP server without authentication or per-call authorization
❌ Framework-specific code all the way down to your domain logic —
   keep the framework at the edges so you can replace it
```

---

## Checklist

```
[ ] You can print the exact final prompt sent to the provider
[ ] Framework versions pinned; upgrades tested
[ ] Provider SDK understood well enough to debug without the framework
[ ] Framework confined to the edges; domain logic framework-free
[ ] Structured outputs via Instructor/PydanticAI with retry-on-validation
[ ] Multi-provider access and budgets handled (LiteLLM or your own router)
[ ] Agent state checkpointed if runs exceed a few seconds
[ ] MCP servers authenticated; every tool call authorized in your own code
[ ] A written reason for each framework you adopted
```

---

**Next:** Phase 14 — memory, the thing every agent framework claims to solve and mostly doesn't.
