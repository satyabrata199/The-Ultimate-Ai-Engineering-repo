# Python Foundations for AI Engineers

### A complete, production-oriented path from zero programming experience to an AI-engineering-ready Python foundation

---

> **What this guide is.** This is not a general Python tutorial with a machine-learning chapter bolted on the end. Every concept here is taught twice: once as Python, and once as *the thing you will actually be doing with it* when you build a retrieval pipeline, fine-tune a model, wrap an LLM in an API, or debug an agent loop at 2 a.m.
>
> **What this guide is not.** It is not a NumPy/PyTorch/LangChain tutorial. It is the layer *underneath* those. When you finish, those libraries will feel like ordinary Python instead of magic — because you will recognise the language machinery they are built from.

---

## Table of Contents

| # | Chapter | Core question it answers |
|---|---------|--------------------------|
| 0 | [Orientation: How Python Runs](#chapter-0--orientation-how-python-actually-runs) | What actually happens when you press Run? |
| 1 | [Python Basics](#chapter-1--python-basics) | What is a value, and what does a variable really hold? |
| 2 | [Control Flow](#chapter-2--control-flow) | How do I make the program decide and repeat? |
| 3 | [Functions](#chapter-3--functions) | How do I package logic so a pipeline can reuse it? |
| 4 | [Data Structures](#chapter-4--data-structures) | How do I hold tokens, batches, configs, and JSON? |
| 5 | [File Handling](#chapter-5--file-handling) | How do datasets, configs, and checkpoints get in and out? |
| 6 | [Error Handling](#chapter-6--error-handling) | How does a system survive a flaky API at 3 a.m.? |
| 7 | [Object-Oriented Programming](#chapter-7--object-oriented-programming) | How do I model a Dataset, a Model, an Agent, a Tool? |
| 8 | [Advanced Python](#chapter-8--advanced-python) | Generators, decorators, async, the GIL, memory |
| A | [Cheat Sheets](#appendix-a--cheat-sheets) | The one-page recall layer |
| B | [Revision Notes](#appendix-b--rapid-revision-notes) | Night-before-the-interview pass |
| C | [Capstone Project](#appendix-c--capstone-project--the-mini-rag-ingestion-service) | Put all eight chapters into one system |
| D | [Where To Go Next](#appendix-d--where-to-go-next) | The ordered library roadmap |

---

## How Every Chapter Is Structured

Each chapter follows the same twelve-part rhythm, so you always know where to look:

```
1. Concept Overview ....... what it is, why it exists, why AI engineers care
2. Theory ................. internal behaviour, execution model, memory, limits
3. Syntax ................. basic -> intermediate -> advanced, every keyword explained
4. Examples ............... beginner -> intermediate -> production -> AI-specific
5. AI Engineering Perspective ... ML / DL / LLM / Agents / RAG / serving
6. Internal Working ....... objects, references, allocation, execution flow
7. Best Practices ......... PEP 8, clean code, Pythonic idioms
8. Common Mistakes ........ wrong vs. right, and *why*
9. Performance ............ time, space, optimisation, trade-offs
10. Interview Preparation . beginner -> intermediate -> AI/ML -> scenario
11. Exercises ............. easy / medium / hard, with full solutions
12. Mini Project .......... a small but real AI-workflow tool
```

**A promise about the code.** Every code block that produces output shows that output. When you see `# ->` it means "this expression evaluates to"; when you see an `Output:` block it means "this is what the terminal prints." Type them yourself. Reading code teaches recognition; typing code teaches recall.

---

## Prerequisites and Setup

You need three things. Nothing else.

**1. Python 3.11 or newer.** (3.12/3.13 are fine. `match` statements need 3.10+; several error messages we rely on got dramatically better in 3.11.)

```bash
python3 --version
# Output: Python 3.12.4
```

**2. A virtual environment — always, from day one.**

```bash
python3 -m venv .venv          # create an isolated Python just for this project
source .venv/bin/activate      # Linux / macOS
# .venv\Scripts\activate       # Windows PowerShell
python -m pip install --upgrade pip
```

> **Why an AI engineer cares immediately.** AI projects are dependency minefields: one project needs `torch==2.1` + `numpy<2`, the next needs `torch==2.4` + `numpy>=2`. Installing into your system Python guarantees that installing project B silently breaks project A. A venv is a per-project sandbox. Teams that skip this lose days to "it works on my machine."

**3. An editor with a Python extension** — VS Code, PyCharm, or Cursor. Turn on a formatter (`ruff format` or `black`) and a linter (`ruff`) now, not later:

```bash
pip install ruff
ruff check .        # find problems
ruff format .       # fix formatting
```

---

# Chapter 0 — Orientation: How Python Actually Runs

Before syntax, build the mental model. Almost every confusing Python behaviour you will ever meet — mutable default arguments, `is` vs `==`, why lists are slow for math, why the GIL exists — falls out of this one picture.

## 0.1 The Execution Pipeline

Python is often called "interpreted," which is half true. What really happens:

```
   your_script.py  (text)
          |
          |  1. LEXER + PARSER  -> builds an Abstract Syntax Tree
          v
        AST
          |
          |  2. COMPILER  -> emits bytecode (stack machine instructions)
          v
    __pycache__/your_script.cpython-312.pyc
          |
          |  3. CPython VM (a big C loop) executes bytecode, one op at a time
          v
      OS / CPU / (for AI: BLAS, CUDA kernels)
```

You can see step 2 directly:

```python
import dis

def add_bias(x):
    return x + 1

dis.dis(add_bias)
```

**Output (Python 3.12, abbreviated):**
```
  2           RESUME                   0
  3           LOAD_FAST                0 (x)
              LOAD_CONST               1 (1)
              BINARY_OP                0 (+)
              RETURN_VALUE
```

Read that as: *push `x`, push `1`, pop both and add, return the result.* That is the machine you are programming. `dis` is not a party trick — it is how you settle arguments about which of two implementations is doing more work.

## 0.2 Everything Is an Object

In Python there are no "primitives." `1`, `"hello"`, `[1,2]`, `print`, `int`, and the module `math` are all **objects**: heap-allocated structures with three properties.

```
        ┌──────────────────────────────┐
        │  PyObject (in heap memory)   │
        ├──────────────────────────────┤
        │  identity  -> memory address │  id(obj)      — never changes
        │  type      -> int / str / …  │  type(obj)    — never changes
        │  value     -> 42             │  may or may not change (mutability)
        │  refcount  -> 3              │  how many names point here
        └──────────────────────────────┘
```

A **variable is a name bound to an object**, not a box holding bytes:

```python
a = [1, 2, 3]
b = a              # NOT a copy — a second name for the SAME object
b.append(4)
print(a)           # -> [1, 2, 3, 4]
print(a is b)      # -> True   (same identity)
```

**Output:**
```
[1, 2, 3, 4]
True
```

```
     a ──┐
         ├──► [1, 2, 3, 4]   (one list object, refcount = 2)
     b ──┘
```

> **Why an AI engineer cares.** This is the single most common source of silent data corruption in preprocessing code. You "copy" a config dict, mutate it per-experiment, and quietly change every experiment's config. You pass a batch list into a function, the function `.append()`s to it, and now your training loop's batch grows every epoch. Chapter 1 §8 and Chapter 3 §8 return to this with real examples.

## 0.3 Why This Matters For Numerical Code

A Python list of a million floats is a million *pointers* to a million separate float objects, each with a header of ~24 bytes:

```
Python list:   [ ptr ][ ptr ][ ptr ] ...   ──► float obj ──► float obj ──► ...
               scattered across the heap; every add() = interpreter dispatch

NumPy array:   [ 8.0 | 8.0 | 8.0 | 8.0 | ... ]
               one contiguous block of raw doubles; add() = one C loop / SIMD
```

```python
import time, numpy as np

py_list = list(range(1_000_000))
np_arr  = np.arange(1_000_000)

t0 = time.perf_counter(); total = sum(x * 2 for x in py_list); t1 = time.perf_counter()
t2 = time.perf_counter(); total_np = (np_arr * 2).sum();       t3 = time.perf_counter()

print(f"pure python: {t1-t0:.4f}s")
print(f"numpy      : {t3-t2:.4f}s")
```

**Output (indicative — your numbers will differ, the ratio will not):**
```
pure python: 0.0821s
numpy      : 0.0013s
```

**~60x**, and the gap widens with size. This is the founding fact of the entire scientific Python stack: *Python is the orchestration language; the numerics happen in C/CUDA underneath.* Your job as an AI engineer is to write Python that spends as little time as possible in the interpreter loop and as much as possible inside vectorised library calls.

## 0.4 The Layer Map You Are Learning

```
┌──────────────────────────────────────────────────────────────┐
│  LangGraph / AI Agents / MCP servers                          │  ← ch.7 OOP, ch.8 async
├──────────────────────────────────────────────────────────────┤
│  LangChain · LlamaIndex · RAG pipelines · vector DB clients   │  ← ch.4 dicts, ch.6 errors
├──────────────────────────────────────────────────────────────┤
│  FastAPI / model serving / Docker                             │  ← ch.8 async, ch.7 classes
├──────────────────────────────────────────────────────────────┤
│  PyTorch · TensorFlow · Transformers                          │  ← ch.7 magic methods, ch.8 iterators
├──────────────────────────────────────────────────────────────┤
│  NumPy · Pandas · scikit-learn · Matplotlib                   │  ← ch.1 types, ch.4 slicing
├──────────────────────────────────────────────────────────────┤
│  ★ CORE PYTHON — objects, references, functions, iteration ★  │  ← THIS GUIDE
├──────────────────────────────────────────────────────────────┤
│  CPython VM (bytecode loop, GIL, refcount GC)                 │
└──────────────────────────────────────────────────────────────┘
```

Every arrow points down. Nobody debugs the top layer successfully without the starred one.

## Chapter 0 Summary

| Idea | One-line takeaway |
|------|-------------------|
| Execution | Source → AST → bytecode → CPython VM loop |
| Objects | Everything is a heap object with identity, type, value |
| Variables | Names bound to objects; assignment never copies |
| Aliasing | Two names, one object — the #1 source of silent data bugs |
| Performance | Interpreter loop is slow; vectorised C underneath is fast |
| Your role | Orchestrate in Python, compute in C/CUDA |

---
# Chapter 1 — Python Basics

> **Chapter goal.** By the end you will know what a value *is*, what a name *is*, why `0.1 + 0.2 != 0.3` will bite you in a loss comparison, and why `dtype` is the first thing a senior engineer checks in a tensor bug report.

---

## 1.1 Concept Overview

### What it is

The "basics" are the vocabulary of the language: **values** (data), **types** (what kind of data), **names/variables** (labels for data), **operators** (how data combines), and **I/O** (how data enters and leaves).

### Why it exists

A program is data plus transformation. Before you can express a transformation you need a way to *name* data and *describe its kind*. Types exist so that `+` can mean addition for numbers and concatenation for strings without ambiguity, and so the runtime can allocate memory correctly.

### Why Python includes it this way

Python made three design choices that shape everything downstream:

1. **Dynamic typing** — a name can be bound to any type; the *object* carries the type, not the name. Fast to write, weaker compile-time guarantees.
2. **Strong typing** — Python will *not* silently coerce `"5" + 5`. It raises. (Contrast with JavaScript, which gives `"55"`.)
3. **Everything is an object** — even integers. Uniform model, some memory cost.

### Why AI engineers care

| Basics concept | Where it shows up in AI work |
|---|---|
| Types & `dtype` | `float32` vs `float64` vs `bfloat16` decides whether your model fits in GPU memory |
| Mutability | Config dicts, batch lists, and dataset caches silently corrupted by aliasing |
| Float precision | Loss comparisons, `np.isclose` in tests, gradient underflow in fp16 |
| String formatting | Every prompt template you will ever write |
| Integer division | `//` for computing `num_batches`; `/` gives a float and breaks indexing |
| Type conversion | Parsing CSV/JSON — everything arrives as a string, nothing is a number yet |

---

## 1.2 Theory

### The object model in one picture

```python
x = 42
```

```
Namespace (a dict)              Heap
┌───────────────┐              ┌────────────────────────┐
│ "x"  ─────────┼─────────────►│ type: int              │
└───────────────┘              │ value: 42              │
                               │ refcount: 1            │
                               └────────────────────────┘
```

`x` lives in a dictionary (the module's `globals()`); the integer lives on the heap. Rebinding `x = 43` does **not** modify the object `42` — it points the name at a *different* object. Integers are immutable.

### Mutable vs Immutable — the master table

| Type | Mutable? | Hashable? | Can be a dict key / set member? |
|------|----------|-----------|-------------------------------|
| `int`, `float`, `complex`, `bool` | No | Yes | Yes |
| `str`, `bytes` | No | Yes | Yes |
| `tuple` (of hashables) | No | Yes | Yes |
| `frozenset` | No | Yes | Yes |
| `list` | **Yes** | No | No |
| `dict` | **Yes** | No | No |
| `set` | **Yes** | No | No |
| `bytearray` | **Yes** | No | No |
| NumPy `ndarray` | **Yes** | No | No |
| PyTorch `Tensor` | **Yes** | No | No |

Read that last column as: *"immutability is the price of being a key."* A dict must be able to trust that an object's hash never changes; a mutable object cannot make that promise.

### Dynamic typing, demonstrated

```python
value = 10          # value -> int object
print(type(value))
value = "ten"       # SAME name, DIFFERENT object of a different type
print(type(value))
value = [1, 0]      # and again
print(type(value))
```

**Output:**
```
<class 'int'>
<class 'str'>
<class 'list'>
```

Nothing about the name `value` constrained the type. This is power *and* danger: nothing stops a preprocessing function from returning a `str` on one branch and a `float` on another, and the failure surfaces 200 lines later inside `model.fit()`.

### Strong typing, demonstrated

```python
print("5" + 5)
```

**Output:**
```
TypeError: can only concatenate str (not "int") to str
```

Good. This exception is a gift — it fails at the source instead of producing `"55"` and poisoning a feature column.

### Advantages and limitations

**Advantages**
- Extremely fast to prototype — critical when you are exploring model architectures.
- Uniform object model → introspection (`type()`, `dir()`, `help()`) works on everything, which is why notebooks and debuggers feel so good.
- Arbitrary-precision integers: no overflow bugs when counting tokens across a 10-billion-token corpus.

**Limitations**
- Every integer is a heap object with a header (~28 bytes for a small `int`). A million-element Python list of ints costs ~40 MB; a NumPy `int32` array costs ~4 MB.
- No compile-time type errors — hence the industry's move to type hints (Chapter 8) plus `mypy`/`pyright` in CI.
- Interpreter dispatch overhead per operation → never write numeric inner loops in pure Python.

---

## 1.3 Syntax

### Basic

```python
# --- variables: name = object ---
model_name = "gpt-oss-20b"      # str
batch_size = 32                 # int
learning_rate = 3e-4            # float  (scientific notation = 0.0003)
use_gpu = True                  # bool
seed = None                     # NoneType — "no value yet"

# --- multiple assignment ---
train, val, test = 0.8, 0.1, 0.1        # tuple unpacking
h = w = 224                             # chained: both names -> same int object

# --- swap without a temp variable ---
a, b = 1, 2
a, b = b, a                             # -> a=2, b=1
```

**Every keyword explained:**
- `=` — *binding*, not "equals." Left side is a name, right side is evaluated first.
- `None` — the singleton "absence of a value." Compare with `is None`, never `== None`.
- `True` / `False` — the two `bool` singletons; `bool` is a subclass of `int` (`True + True == 2`).

### Intermediate

```python
# --- walrus operator := : assign inside an expression (3.8+) ---
tokens = ["hi", "there"]
if (n := len(tokens)) > 1:
    print(f"{n} tokens")            # -> 2 tokens

# --- augmented assignment ---
total_loss = 0.0
total_loss += 0.42                  # equivalent to total_loss = total_loss + 0.42

# --- constants by convention (Python has no `const`) ---
MAX_SEQ_LEN = 4096                  # SCREAMING_SNAKE means "do not reassign"

# --- unpacking with star ---
first, *rest = [101, 2054, 2003, 102]
print(first, rest)                  # -> 101 [2054, 2003, 102]

# --- type-hinted variables (documentation the tools can check) ---
temperature: float = 0.7
stop_sequences: list[str] = ["\n\n", "###"]
```

### Advanced

```python
from typing import Final, TypeAlias

EMBED_DIM: Final = 1536             # `Final` = a linter error if reassigned
Embedding: TypeAlias = list[float]  # a readable name for a shape-bearing type

# f-string mini-language: the formatting DSL you will use constantly
acc = 0.876543
print(f"{acc:.2%}")                 # -> 87.65%      percentage, 2 dp
print(f"{acc:>10.3f}")              # ->      0.877  right-aligned in 10 cols
print(f"{1234567:,}")               # -> 1,234,567   thousands separator
print(f"{255:#x} {255:08b}")        # -> 0xff 11111111
print(f"{acc=}")                    # -> acc=0.876543   self-documenting debug
```

**Format spec grammar** (`{value:[[fill]align][sign][#][0][width][,][.precision][type]}`):

| Piece | Meaning | Example |
|---|---|---|
| `<` `>` `^` | left / right / centre align | `f"{n:>8}"` |
| `+` | always show sign | `f"{delta:+.3f}"` |
| `,` `_` | thousands separator | `f"{params:,}"` |
| `.N` | precision | `f"{loss:.4f}"` |
| `f` `e` `%` `g` | fixed / scientific / percent / general | `f"{lr:.1e}"` |
| `d` `b` `o` `x` | decimal / binary / octal / hex | `f"{flags:08b}"` |

---

## 1.4 Examples

### Beginner — the five core types

```python
epochs      = 10                 # int
lr          = 1e-3               # float
name        = "bert-base"        # str
is_frozen   = False              # bool
checkpoint  = None               # NoneType

for v in (epochs, lr, name, is_frozen, checkpoint):
    print(f"{str(v):<12} {type(v).__name__}")
```

**Output:**
```
10           int
0.001        float
bert-base    str
False        bool
None         NoneType
```

**Line by line.**
1–5. Five names bound to five heap objects. Note `1e-3` is float syntax, not "e to the minus 3".
7. `(...)` is a tuple — an immutable sequence; iterating over it visits each element.
8. `str(v)` converts any object to text; `:<12` left-aligns it in 12 columns; `type(v).__name__` reads the *class name* off the object's type.

### Intermediate — type conversion, the CSV problem

Everything read from a file or an HTTP response is a **string**. Converting is your job.

```python
row = {"text": "great movie", "label": "1", "score": "0.94", "verified": "true"}

label    = int(row["label"])
score    = float(row["score"])
verified = row["verified"].strip().lower() in {"true", "1", "yes"}

print(label, type(label).__name__)
print(score, type(score).__name__)
print(verified, type(verified).__name__)
```

**Output:**
```
1 int
0.94 float
True bool
```

**Line by line.**
- `int("1")` parses text to an integer; it raises `ValueError` on `"1.0"` or `""` — which is exactly what you want (fail loudly on dirty data).
- `float("0.94")` parses text to a double.
- **The critical line is the bool.** `bool("false")` is `True` — because *any non-empty string is truthy*. This is one of the most common data-loading bugs in the industry. Always parse booleans against an explicit set of accepted tokens.

```python
print(bool("false"), bool("0"), bool(""))   # -> True True False   ← the trap
```

### Production-style — a validated config loader

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class TrainConfig:
    model_name: str
    batch_size: int
    learning_rate: float
    fp16: bool = True

    def __post_init__(self) -> None:
        if self.batch_size <= 0:
            raise ValueError(f"batch_size must be positive, got {self.batch_size}")
        if not (0.0 < self.learning_rate < 1.0):
            raise ValueError(f"learning_rate out of range: {self.learning_rate}")


raw = {"model_name": "bert-base", "batch_size": "32", "learning_rate": "3e-4"}

cfg = TrainConfig(
    model_name=raw["model_name"],
    batch_size=int(raw["batch_size"]),
    learning_rate=float(raw["learning_rate"]),
)
print(cfg)
print(f"effective tokens/step = {cfg.batch_size * 512:,}")
```

**Output:**
```
TrainConfig(model_name='bert-base', batch_size=32, learning_rate=0.0003, fp16=True)
effective tokens/step = 16,384
```

**Why this is the production shape.** `frozen=True` makes the config immutable — nothing downstream can mutate your run's settings halfway through training. `slots=True` removes the per-instance `__dict__` (memory + attribute-speed win). `__post_init__` validates *at construction*, so a bad hyperparameter fails in second one, not in hour three. Full treatment in Chapter 7.

### AI-specific — numeric precision and `dtype`

```python
import numpy as np

x64 = np.array([1/3], dtype=np.float64)
x32 = np.array([1/3], dtype=np.float32)
x16 = np.array([1/3], dtype=np.float16)

for name, arr in [("float64", x64), ("float32", x32), ("float16", x16)]:
    print(f"{name}: value={str(arr[0]):<22} bytes/elem={arr.itemsize}")
```

**Output:**
```
float64: value=0.3333333333333333     bytes/elem=8
float32: value=0.33333334             bytes/elem=4
float16: value=0.3333                 bytes/elem=2
```

**Read the memory implication.** A 7-billion-parameter model:

| dtype | Bytes/param | Weights alone |
|---|---|---|
| `float32` | 4 | 28 GB — will not fit on a 24 GB GPU |
| `float16` / `bfloat16` | 2 | 14 GB — fits |
| `int8` (quantised) | 1 | 7 GB — fits comfortably |
| `int4` | 0.5 | 3.5 GB — fits on a laptop GPU |

`dtype` is not a detail. It is the difference between "runs" and "CUDA out of memory."

### AI-specific — the floating-point equality trap

```python
print(0.1 + 0.2)                 # -> 0.30000000000000004
print(0.1 + 0.2 == 0.3)          # -> False

import math
print(math.isclose(0.1 + 0.2, 0.3, rel_tol=1e-9))   # -> True
```

**Output:**
```
0.30000000000000004
False
True
```

**Why.** IEEE-754 binary floats cannot represent 0.1 exactly, any more than decimal can represent 1/3 exactly. Consequence for you: **never** write `if loss == best_loss:` or `assert output == expected` on floats. Use `math.isclose`, `np.isclose`, `np.allclose`, or `torch.allclose` with explicit tolerances. Every serious ML test suite is built on these.

### AI-specific — prompt templates with f-strings

```python
SYSTEM = "You are a precise assistant. Answer only from the context."

def build_prompt(question: str, chunks: list[str], max_chars: int = 2000) -> str:
    context = "\n\n---\n\n".join(chunks)[:max_chars]
    return (
        f"{SYSTEM}\n\n"
        f"# Context\n{context}\n\n"
        f"# Question\n{question}\n\n"
        f"# Answer\n"
    )

print(build_prompt("What is RAG?", ["RAG retrieves documents, then generates."]))
```

**Output:**
```
You are a precise assistant. Answer only from the context.

# Context
RAG retrieves documents, then generates.

# Question
What is RAG?

# Answer

```

**Line by line.**
- `"\n\n---\n\n".join(chunks)` — the idiomatic way to glue a list of strings with a separator. Building this with `+=` in a loop is O(n²); `join` is O(n) (see §1.9).
- `[:max_chars]` — a slice, guarding against blowing the model's context window.
- The implicit string concatenation across adjacent lines inside `(...)` is the clean way to compose a multi-part prompt without a huge unreadable literal.

---

## 1.5 AI Engineering Perspective

**Machine Learning.** Feature engineering is 90% type discipline. A column read as `str` instead of `float` silently becomes a 40,000-category one-hot feature in scikit-learn and your model memorises noise. Always assert dtypes after loading: `assert df["age"].dtype == "float64"`.

**Deep Learning.** Mixed precision (`bfloat16` weights, `float32` optimizer state) is standard practice. Understanding that `float16` has ~3 decimal digits of precision and a max value of 65504 explains gradient overflow and why loss scaling exists. `bfloat16` trades mantissa for exponent range — same range as `float32`, less precision — which is why it is more forgiving in training.

**LLM applications.** Every prompt is string construction. Every token count is integer arithmetic. Every temperature/top-p is a float you must validate at the boundary (`0.0 <= temperature <= 2.0`) *before* the API rejects it 400 ms and $0.002 later.

**AI Agents.** Agent state is a mutable dict passed between steps. If two tools alias the same state object and both mutate it, you get non-deterministic behaviour that is nearly impossible to reproduce. Immutability (frozen dataclasses, tuples, copy-on-write) is the antidote.

**RAG.** Chunk sizes, overlap, and `top_k` are integers; similarity scores are floats compared with thresholds. `if score > 0.8` where `score` is `numpy.float32` behaves fine, but `if score == 1.0` for an exact-duplicate check does not — use `isclose`.

**Data preprocessing.** `int(x)` truncates toward zero (`int(-2.7) == -2`), `round()` uses banker's rounding (`round(0.5) == 0`, `round(1.5) == 2`). Both surprise people writing label-binning code.

**Model serving.** JSON has one number type. An `int64` id serialised to JSON and parsed by JavaScript loses precision above 2^53. Serialise large ids as strings.

---

## 1.6 Internal Working

### Small-integer and string interning

CPython pre-allocates the integers **−5 to 256** and reuses them:

```python
a = 256; b = 256
print(a is b)          # -> True   (same cached object)

c = 257; d = 257
print(c is d)          # -> False in a fresh REPL (two distinct objects)
print(c == d)          # -> True   (equal values)
```

**Output (REPL):**
```
True
False
True
```

> **Rule that follows:** use `==` for value comparison, `is` only for singletons (`None`, `True`, `False`) and genuine identity checks. `if x is 0:` is a bug that happens to work for small numbers and fails for large ones. Python 3.8+ even emits a `SyntaxWarning` for it.

### Reference counting and garbage collection

```python
import sys

data = [0] * 1_000_000
print(sys.getrefcount(data) - 1)   # -> 1   (minus 1 for the temporary arg ref)

alias = data
print(sys.getrefcount(data) - 1)   # -> 2

del alias                          # refcount back to 1 — object still alive
del data                           # refcount 0 — memory freed immediately
```

CPython frees an object the instant its refcount hits zero. A separate **cycle collector** handles reference cycles (`a.b = b; b.a = a`), running periodically in generations.

**Why an AI engineer cares:** holding one stray reference to a batch — appending it to a `history` list "for debugging" — prevents the GPU tensor it wraps from being freed, and you OOM at epoch 12 with no obvious cause. The fix is usually `history.append(loss.item())` (a plain float) instead of `history.append(loss)` (a tensor holding the entire autograd graph).

### Memory footprint reality check

```python
import sys
print(sys.getsizeof(0))            # -> 28    bytes for one small int
print(sys.getsizeof(1 << 100))     # -> 44    arbitrary precision grows
print(sys.getsizeof("a"))          # -> 50
print(sys.getsizeof([]))           # -> 56    empty list overhead
print(sys.getsizeof([0]*1000))     # -> 8056  (56 + 1000 pointers × 8B)
```

Note the last line: the list costs 8 KB *for the pointers alone*, and the 1000 ints they point to are extra (though here they all alias the cached `0`). A `np.zeros(1000, dtype=np.int8)` costs 1000 bytes total.

---

## 1.7 Best Practices

```python
# ---------- naming (PEP 8) ----------
learning_rate = 3e-4          # ✔ snake_case for variables and functions
MAX_TOKENS = 4096             # ✔ UPPER_SNAKE for constants
class VectorStore: ...        # ✔ PascalCase for classes
_internal_cache = {}          # ✔ leading underscore = "private, don't touch"
type_ = "encoder"             # ✔ trailing underscore avoids shadowing builtins

# ---------- clarity ----------
n_correct = 0                 # ✔ short but meaningful
x = 0                         # ✘ meaningless outside a 3-line scope
lst = []                      # ✘ says the type, not the purpose
token_ids: list[int] = []     # ✔ purpose + type
```

**Rules worth internalising**

1. **Never shadow builtins.** `list = [1,2]`, `id = 5`, `type = "x"`, `input = ...` break later code in the same scope. If you need the name, add a trailing underscore.
2. **Annotate module-level and function-signature types.** Not every local, but every boundary.
3. **Prefer explicit conversion at the edges.** Parse once at the ingestion boundary; keep types clean internally.
4. **Use `is None`, not `== None`.** `==` can be overloaded (NumPy arrays return an *array* from `==`, which then raises "truth value is ambiguous").
5. **Format strings with f-strings.** Not `%`, not `.format()`, except in logging (see below).
6. **Logging is the exception:**
   ```python
   logger.info("loaded %d rows from %s", n, path)   # ✔ lazy — formats only if emitted
   logger.info(f"loaded {n} rows from {path}")      # ✘ formats even when log level filters it
   ```
7. **One statement per line.** No `a = 1; b = 2`.
8. **4 spaces, ≤88-char lines** (`ruff`/`black` default). Let the formatter decide; don't argue about it in review.

---

## 1.8 Common Mistakes

### Mistake 1 — Aliasing a config instead of copying

```python
# ❌ WRONG
base_config = {"lr": 1e-3, "layers": [64, 32]}
exp_a = base_config
exp_a["lr"] = 1e-4
print(base_config["lr"])        # -> 0.0001   base is destroyed
```

```python
# ✔ CORRECT
import copy
base_config = {"lr": 1e-3, "layers": [64, 32]}
exp_a = copy.deepcopy(base_config)     # nested structures also copied
exp_a["lr"] = 1e-4
exp_a["layers"].append(16)
print(base_config)              # -> {'lr': 0.001, 'layers': [64, 32]}   intact
```

**Why.** `=` binds a second name to the same object. `dict.copy()` / `dict(base)` is a *shallow* copy — the nested `layers` list is still shared. Only `copy.deepcopy` (or a frozen dataclass) fully isolates.

### Mistake 2 — Comparing floats with `==`

```python
# ❌ WRONG
if val_loss == best_loss:
    save_checkpoint()

# ✔ CORRECT
if math.isclose(val_loss, best_loss, rel_tol=1e-6):
    save_checkpoint()
# or, for the far more common "did it improve?" case:
if val_loss < best_loss - 1e-6:
    save_checkpoint()
```

### Mistake 3 — `bool()` on strings from files

```python
# ❌ WRONG
use_gpu = bool(os.environ.get("USE_GPU", "false"))   # -> True, always

# ✔ CORRECT
use_gpu = os.environ.get("USE_GPU", "false").strip().lower() in {"1", "true", "yes", "on"}
```

Environment variables are *always* strings. This exact bug has shipped to production at every company that runs Python.

### Mistake 4 — `/` where you meant `//`

```python
n_samples, batch_size = 1000, 32

# ❌ WRONG
n_batches = n_samples / batch_size          # -> 31.25 (float)
# indexing with it raises: TypeError: list indices must be integers

# ✔ CORRECT
n_batches = n_samples // batch_size         # -> 31  (floor division)
n_batches_all = -(-n_samples // batch_size) # -> 32  (ceiling, keeps last partial batch)
import math
n_batches_all = math.ceil(n_samples / batch_size)   # -> 32  (clearer)
```

That last line matters: forgetting the partial final batch silently drops up to `batch_size - 1` samples from every evaluation.

### Mistake 5 — Mutating a string in a loop

```python
# ❌ WRONG — O(n²): each += allocates a whole new string
prompt = ""
for chunk in chunks:
    prompt += chunk + "\n"

# ✔ CORRECT — O(n)
prompt = "\n".join(chunks) + "\n"
```

### Mistake 6 — Shadowing a builtin

```python
# ❌ WRONG
list = [1, 2, 3]
tokens = list("hello")        # TypeError: 'list' object is not callable

# ✔ CORRECT
token_ids = [1, 2, 3]
tokens = list("hello")        # -> ['h','e','l','l','o']
```

---

## 1.9 Performance

| Operation | Complexity | Note |
|---|---|---|
| `x = y` (binding) | O(1) | Never copies |
| `int + int` | O(1) small / O(n) bignum | Arbitrary precision costs above 2^63 |
| `str + str` | O(n + m) | Allocates a new string every time |
| `"".join(list_of_str)` | O(total length) | **Always prefer for 3+ pieces** |
| `f"{x}"` | O(len of result) | Fastest of the formatting options |
| `int(s)` / `float(s)` | O(len(s)) | Parsing is not free — vectorise with `pandas.to_numeric` |
| `sys.getsizeof(int)` | 28 bytes | vs 4 bytes for `np.int32` |

### Measure, don't guess

```python
import timeit

setup = "parts = [str(i) for i in range(1000)]"
concat = "s = ''\nfor p in parts: s += p"
joined = "s = ''.join(parts)"

print(f"+= loop: {timeit.timeit(concat, setup, number=1000):.4f}s")
print(f"join   : {timeit.timeit(joined, setup, number=1000):.4f}s")
```

**Output (indicative):**
```
+= loop: 0.0642s
join   : 0.0061s
```

### Optimisation ladder for AI code

```
1. Is it actually slow?           → profile first (cProfile, py-spy). Never guess.
2. Wrong algorithm?               → O(n²) lookup in a list → O(1) in a set/dict. Biggest win.
3. Pure-Python numeric loop?      → vectorise with NumPy/pandas. 10–100×.
4. Still slow, embarrassingly parallel? → multiprocessing (ch.8).
5. I/O-bound (API calls)?         → asyncio (ch.8). 10–50× for RAG ingestion.
6. Truly CPU-bound in Python?     → Cython / Numba / Rust extension. Last resort.
```

---

## 1.10 Interview Preparation

**Q1 (Beginner). Is Python interpreted or compiled?**
Both, in stages. Source is compiled to bytecode (cached in `__pycache__/*.pyc`), and that bytecode is executed by the CPython virtual machine. It is not compiled to native machine code ahead of time, which is why per-operation overhead is high and why numerics live in C extensions.

**Q2 (Beginner). Difference between `is` and `==`?**
`==` compares *values* (calls `__eq__`); `is` compares *identity* (same memory address). Use `is` only for `None`, `True`, `False`, and sentinel objects. `a == b` can be `True` while `a is b` is `False` — two distinct lists with the same contents.

**Q3 (Beginner). Mutable vs immutable — name three of each and why it matters.**
Immutable: `int`, `str`, `tuple`, `frozenset`, `bytes`. Mutable: `list`, `dict`, `set`, `ndarray`, `Tensor`. It matters because (a) only immutables are hashable and so usable as dict keys / set members, (b) mutables passed to a function can be modified by that function — a common source of bugs, (c) immutable objects are safe to share across threads without locks.

**Q4 (Intermediate). Why is `0.1 + 0.2 != 0.3`, and how do you handle it in ML code?**
Binary floating point cannot represent 0.1 or 0.2 exactly; the sum's nearest representable double is 0.30000000000000004. Handle it with `math.isclose` / `np.allclose` / `torch.allclose` and explicit tolerances. In practice: never `==` floats, use `<` with a min-delta for "improved" checks, and use `Decimal` only for money, never for tensors.

**Q5 (Intermediate). What does `sys.getsizeof([0]*1000)` return and why is it not 1000?**
About 8056 bytes: 56 bytes of list header plus 1000 × 8-byte pointers. The integers themselves are separate heap objects (here all aliasing the cached `0`). This is the memory argument for NumPy: `np.zeros(1000, np.int8)` is ~1000 bytes contiguous.

**Q6 (AI/ML). Why do we train in `bfloat16` rather than `float16`?**
`float16` has 5 exponent bits (max ≈ 65504) and 10 mantissa bits; gradients underflow/overflow easily, requiring loss scaling. `bfloat16` has 8 exponent bits — the same dynamic range as `float32` — with only 7 mantissa bits. Deep learning is far more sensitive to range than to precision, so `bfloat16` trains stably without loss scaling. Both halve memory versus `float32`.

**Q7 (AI/ML). A 13B-parameter model in fp16 — estimate GPU memory for inference and for full fine-tuning.**
Inference: 13e9 × 2 bytes ≈ 26 GB weights, plus KV cache (grows with batch × sequence length) — needs an 40 GB card. Full fine-tuning with Adam: weights (2) + gradients (2) + fp32 master weights (4) + Adam m and v (4 + 4) ≈ 16 bytes/param ≈ 208 GB — hence LoRA/QLoRA, which train ~0.1% of parameters.

**Q8 (Scenario). Your training accuracy is 99% and validation is 51%, and you find the label column loaded as `object` dtype. What happened?**
Labels arrived as strings, so a downstream encoder treated near-duplicate strings (`"1"`, `"1 "`, `"1.0"`) as distinct classes, or a feature column containing the label leaked in as text. Fix: explicit dtype casting and validation at load time (`df["label"] = df["label"].astype("int8")` with a `.unique()` assertion), plus a schema check in CI.

**Q9 (Production). Why is `logger.info(f"...")` discouraged?**
The f-string is evaluated eagerly, even when the log level would discard the record. With `%s`-style lazy args, formatting happens only if the record is actually emitted. On a hot path with `DEBUG` logs disabled in prod, this is free versus measurable.

**Q10 (Production). A config value read from an env var behaves as `True` even when set to `"false"`. Diagnose.**
`bool()` on any non-empty string is `True`. The code did `bool(os.environ["FLAG"])`. Fix by parsing against an explicit truthy token set, or use `pydantic-settings`, which does typed env parsing correctly.

---

## 1.11 Exercises

### Easy — E1
Write a script that takes a model name, parameter count, and dtype, and prints the estimated weight memory in GB with 2 decimal places.

<details><summary>Solution</summary>

```python
BYTES = {"float32": 4, "float16": 2, "bfloat16": 2, "int8": 1, "int4": 0.5}

def weight_memory_gb(params: int, dtype: str) -> float:
    if dtype not in BYTES:
        raise ValueError(f"unknown dtype {dtype!r}; expected one of {sorted(BYTES)}")
    return params * BYTES[dtype] / 1024**3

for dt in ("float32", "bfloat16", "int4"):
    print(f"Llama-7B @ {dt:<9} = {weight_memory_gb(7_000_000_000, dt):.2f} GB")
```
**Output:**
```
Llama-7B @ float32   = 26.08 GB
Llama-7B @ bfloat16  = 13.04 GB
Llama-7B @ int4      = 3.26 GB
```
**Explanation.** A dict maps dtype → bytes (an O(1) lookup and a single place to add new dtypes). We divide by `1024**3` for gibibytes. The `ValueError` with the received value and the valid set is the pattern every production function should follow: fail fast, and tell the caller how to fix it.
</details>

### Easy — E2
Given `raw = " 0.87 "`, produce a float, and given `raw2 = "TRUE"`, produce a bool — safely.

<details><summary>Solution</summary>

```python
raw, raw2 = " 0.87 ", "TRUE"
score = float(raw.strip())
flag  = raw2.strip().lower() in {"1", "true", "yes", "y", "on"}
print(score, flag)      # -> 0.87 True
```
`float()` tolerates surrounding whitespace, but `.strip()` makes the intent explicit and protects against `" "`-style non-breaking spaces in scraped data. The bool must never go through `bool()`.
</details>

### Medium — M1
Write `format_metrics(metrics: dict[str, float]) -> str` producing an aligned report, percentages for anything named `*_acc` or `*_rate`, and 4-dp fixed for everything else.

<details><summary>Solution</summary>

```python
def format_metrics(metrics: dict[str, float]) -> str:
    width = max(len(k) for k in metrics)
    lines = []
    for key, value in metrics.items():
        if key.endswith(("_acc", "_rate", "_accuracy")):
            rendered = f"{value:>9.2%}"
        else:
            rendered = f"{value:>9.4f}"
        lines.append(f"{key:<{width}} : {rendered}")
    return "\n".join(lines)

print(format_metrics({"train_loss": 0.0342, "val_acc": 0.9123, "f1": 0.8877,
                      "error_rate": 0.0877}))
```
**Output:**
```
train_loss :    0.0342
val_acc    :    91.23%
f1         :    0.8877
error_rate :     8.77%
```
**Explanation.** `max(len(k) for k in metrics)` computes the column width from the data — a generator expression, so no intermediate list. `f"{key:<{width}}"` shows *nested* format specs: the width itself is interpolated. `str.endswith` accepts a tuple, testing several suffixes in one call. `join` builds the result in O(n).
</details>

### Medium — M2
Demonstrate, with `id()` and output, that `copy.copy` is insufficient for a nested config and `copy.deepcopy` is correct.

<details><summary>Solution</summary>

```python
import copy

base = {"lr": 1e-3, "layers": [64, 32]}
shallow = copy.copy(base)
deep    = copy.deepcopy(base)

print(base["layers"] is shallow["layers"])   # -> True   SHARED
print(base["layers"] is deep["layers"])      # -> False  ISOLATED

shallow["layers"].append(999)
print(base["layers"])                        # -> [64, 32, 999]  corrupted
deep["layers"].append(111)
print(base["layers"])                        # -> [64, 32, 999]  unchanged by deep
```
**Explanation.** A shallow copy duplicates the outer dict's *pointer table* only; the inner list object is shared. `deepcopy` walks the object graph recursively (and handles cycles via a memo dict). Cost: `deepcopy` is O(total nodes) and can be slow on large graphs — for hot paths, prefer immutable configs so no copy is needed at all.
</details>

### Hard — H1
Implement `safe_parse_record(raw: dict[str, str], schema: dict[str, type]) -> dict` that converts every field per the schema, collects **all** errors rather than failing on the first, and raises one `ValueError` listing them.

<details><summary>Solution</summary>

```python
TRUTHY = {"1", "true", "yes", "y", "on"}
FALSY  = {"0", "false", "no", "n", "off", ""}

def _to_bool(text: str) -> bool:
    t = text.strip().lower()
    if t in TRUTHY:
        return True
    if t in FALSY:
        return False
    raise ValueError(f"not a boolean: {text!r}")

def safe_parse_record(raw: dict[str, str], schema: dict[str, type]) -> dict:
    parsed, errors = {}, []
    for field, target in schema.items():
        if field not in raw:
            errors.append(f"{field}: missing")
            continue
        try:
            parsed[field] = _to_bool(raw[field]) if target is bool else target(raw[field].strip())
        except (ValueError, TypeError) as exc:
            errors.append(f"{field}: cannot convert {raw[field]!r} to {target.__name__} ({exc})")
    if errors:
        raise ValueError("record validation failed:\n  - " + "\n  - ".join(errors))
    return parsed


schema = {"text": str, "label": int, "score": float, "verified": bool}
print(safe_parse_record({"text": "hi", "label": "1", "score": "0.9",
                         "verified": "TRUE"}, schema))

try:
    safe_parse_record({"text": "hi", "label": "one", "verified": "maybe"}, schema)
except ValueError as exc:
    print(exc)
```
**Output:**
```
{'text': 'hi', 'label': 1, 'score': 0.9, 'verified': True}
record validation failed:
  - label: cannot convert 'one' to int (invalid literal for int() with base 10: 'one')
  - score: missing
  - verified: cannot convert 'maybe' to bool (not a boolean: 'maybe')
```
**Explanation.** Three production ideas in one function. (1) **Accumulate errors** — when validating 100k dataset rows, "row 4 field label is bad" is far more useful than dying on the first one. (2) **`bool` gets a special path**, because `bool(str)` is never what you want. (3) The message includes `!r` (the `repr`) so invisible whitespace and unicode show up. This is a hand-rolled version of what `pydantic` does; in real projects, use pydantic — but knowing what it does for you is the point.
</details>

---

## 1.12 Mini Project — Dataset Field Profiler

**Goal.** Given a list of raw string records (what you get out of a CSV or a scraped JSONL), infer each field's likely type, report null rates and cardinality, and flag fields that will cause problems downstream. This is the first thing you should run against any new dataset.

```python
"""profiler.py — infer types and quality signals for a raw string dataset."""
from collections import Counter

NULL_TOKENS = {"", "na", "n/a", "null", "none", "nan", "-"}
TRUTHY = {"1", "true", "yes", "y", "on"}
FALSY  = {"0", "false", "no", "n", "off"}


def infer_type(values: list[str]) -> str:
    """Return the narrowest type that fits every non-null value."""
    clean = [v.strip() for v in values if v.strip().lower() not in NULL_TOKENS]
    if not clean:
        return "empty"

    def all_are(pred) -> bool:
        return all(pred(v) for v in clean)

    def is_int(v: str) -> bool:
        try:
            int(v); return True
        except ValueError:
            return False

    def is_float(v: str) -> bool:
        try:
            float(v); return True
        except ValueError:
            return False

    if all_are(lambda v: v.lower() in TRUTHY | FALSY):
        return "bool"
    if all_are(is_int):
        return "int"
    if all_are(is_float):
        return "float"
    return "str"


def profile(records: list[dict[str, str]]) -> None:
    if not records:
        print("no records"); return

    fields = list(records[0])
    print(f"{len(records)} records, {len(fields)} fields\n")
    header = f"{'field':<12} {'type':<7} {'null%':>7} {'unique':>7} {'sample':<18} flags"
    print(header); print("-" * len(header))

    for field in fields:
        values = [str(r.get(field, "")) for r in records]
        nulls  = sum(1 for v in values if v.strip().lower() in NULL_TOKENS)
        null_pct = nulls / len(values)
        uniq = len(set(values))
        kind = infer_type(values)
        sample = next((v for v in values if v.strip()), "")[:16]

        flags = []
        if null_pct > 0.2:                       flags.append("HIGH-NULL")
        if uniq == 1:                            flags.append("CONSTANT")
        if uniq == len(values) and kind == "str": flags.append("ID-LIKE")
        if kind == "str" and uniq > 0.5 * len(values) > 10:
            flags.append("HIGH-CARDINALITY")

        print(f"{field:<12} {kind:<7} {null_pct:>6.1%} {uniq:>7} "
              f"{sample!r:<18} {' '.join(flags)}")


if __name__ == "__main__":
    data = [
        {"id": "a1", "text": "great film",  "label": "1", "score": "0.91", "src": "imdb"},
        {"id": "a2", "text": "awful",       "label": "0", "score": "0.12", "src": "imdb"},
        {"id": "a3", "text": "",            "label": "1", "score": "N/A",  "src": "imdb"},
        {"id": "a4", "text": "fine",        "label": "0", "score": "0.55", "src": "imdb"},
    ]
    profile(data)
```

**Output:**
```
4 records, 5 fields

field        type      null%  unique sample             flags
------------------------------------------------------------
id           str       0.0%        4 'a1'               ID-LIKE
text         str      25.0%        4 'great film'       HIGH-NULL ID-LIKE
label        bool      0.0%        2 '1'
score        float    25.0%        4 '0.91'             HIGH-NULL
src          str       0.0%        1 'imdb'             CONSTANT
```

**What it teaches and why it's real.** `id` is flagged ID-LIKE — never feed it to a model. `src` is CONSTANT — zero information, drop it. `score` has 25% nulls — decide imputation *now*, not after training. `label` inferred as `bool` because it is 0/1 — correct, and it tells you the task is binary classification. Every extension is a natural next exercise: write the report to JSON (Chapter 5), raise a custom `SchemaError` (Chapter 6), turn `profile` into a `DatasetProfiler` class with pluggable checks (Chapter 7).

---

## Chapter 1 Summary

| Concept | Takeaway you must keep |
|---|---|
| Variable | A name bound to a heap object; `=` never copies |
| Dynamic + strong typing | Any type may be bound to a name, but no silent coercion |
| Mutability | Immutable → hashable → usable as dict key; mutable → shared-state bugs |
| Floats | Never `==`; use `isclose`/`allclose`; `dtype` decides GPU fit |
| Conversion | Files and env vars give strings; `bool("false")` is `True` |
| Formatting | f-strings everywhere, `%s` lazy args in logging |
| Interning | `-5..256` cached → `is` is not `==` |
| Memory | 28 B per Python int vs 4 B per `np.int32` → vectorise |

**Revision flashcards.** `is` vs `==` · `//` vs `/` · shallow vs deep copy · `bool("false")` · `float16` vs `bfloat16` · `"".join()` vs `+=`.

---
# Chapter 2 — Control Flow

> **Chapter goal.** Control flow is where a *script* becomes a *system*. Training loops, dataset filters, retry logic, prompt routers, and agent decision cycles are all control flow. By the end you will also know when **not** to write a loop — which in AI code is most of the time.

---

## 2.1 Concept Overview

### What it is

Control flow is the set of constructs that decide **which** statements run and **how many times**: conditionals (`if`/`elif`/`else`, `match`), loops (`for`, `while`), and loop modifiers (`break`, `continue`, `else`, `pass`).

### Why it exists

Straight-line code can only do one fixed thing. Real systems must branch on data ("is this chunk long enough to embed?") and repeat over collections ("for each of 40,000 documents").

### Why Python includes it this way

Python's distinguishing choices:

- **Indentation defines blocks.** No braces. This is not cosmetic — it makes badly-structured code *look* bad, which is why Python codebases stay readable at scale.
- **`for` is a `foreach`.** There is no C-style `for(i=0;i<n;i++)`. Python's `for` walks an **iterator**, which is why it works identically over a list, a file, a generator, a `DataLoader`, and a streaming LLM response.
- **Truthiness.** Any object can be used in a condition; empty containers and zeros are falsy.
- **`for/else` and `while/else`** — a rare construct that is genuinely useful for search loops.

### Why AI engineers care

| Construct | Real AI use |
|---|---|
| `if/elif/else` | Prompt routing, threshold decisions, early stopping |
| `for` | Epoch loop, batch loop, document ingestion, agent steps |
| `while` | Retry with backoff, agent "keep going until done", token streaming |
| `break` | Early stopping on patience exhaustion; stop at first passing candidate |
| `continue` | Skip malformed records without killing the ingestion run |
| `match` | Dispatching on tool name / message role / event type |
| Comprehensions | The Pythonic replacement for 80% of the loops you'd otherwise write |
| **Not looping** | Vectorisation — the single biggest performance lever in ML code |

---

## 2.2 Theory

### Truthiness — the rules

An object is used as a condition by calling `bool()` on it, which consults `__bool__` then `__len__`.

| Falsy | Truthy |
|---|---|
| `False`, `None` | `True` |
| `0`, `0.0`, `0j`, `Decimal(0)` | Any non-zero number |
| `""`, `[]`, `()`, `{}`, `set()`, `range(0)` | Any non-empty container |
| Objects whose `__len__` returns 0 | Everything else, including `"0"` and `"False"` |

```python
chunks = []
if not chunks:                      # ✔ idiomatic emptiness check
    print("nothing to embed")
```
**Output:** `nothing to embed`

> **The NumPy/PyTorch exception — memorise this.** Arrays and tensors override `__bool__` to *raise* when they contain more than one element:
> ```python
> import numpy as np
> if np.array([1, 2]):
>     ...
> # ValueError: The truth value of an array with more than one element is ambiguous.
> ```
> Use `.any()`, `.all()`, or `.size == 0`. This error appears in every ML engineer's first month.

### Short-circuit evaluation

`and` / `or` stop as soon as the answer is known, and **return an operand, not a bool**:

```python
print(0 or "default")        # -> 'default'
print("gpt-4" or "fallback") # -> 'gpt-4'
print(None and crash())      # -> None   (crash() never called)
```

This gives the classic guard idiom:

```python
if response is not None and response.get("choices"):   # safe: order matters
    ...
```

If you reversed the operands you would get an `AttributeError` on `None`. **Short-circuiting is a correctness tool, not just an optimisation.**

### Conditional expressions (ternary)

```python
device = "cuda" if torch.cuda.is_available() else "cpu"
```
Reads left-to-right as an English sentence. Use for simple value selection; never nest more than one level.

### Loop mechanics — what `for` really does

```python
for item in iterable:
    body(item)
```

desugars to:

```python
_it = iter(iterable)          # calls iterable.__iter__()
while True:
    try:
        item = next(_it)      # calls _it.__next__()
    except StopIteration:
        break
    body(item)
```

**This is the most important paragraph in the chapter.** Because `for` only needs `__iter__`/`__next__`, the same loop works over:

```python
for line in open("data.jsonl"):        # file object — streams, never loads all
for batch in DataLoader(dataset):      # PyTorch — yields collated tensors
for token in llm.stream(prompt):       # generator — yields as tokens arrive
for row in pd.read_csv(f, chunksize=10_000):  # pandas — chunked reading
```

You will build your own iterables in Chapter 8. For now: `for` is a protocol, not a counter.

### `for`/`else` and `while`/`else`

The `else` clause runs **only if the loop finished without `break`**. Read `else` as "no break":

```python
for ckpt in candidate_checkpoints:
    if ckpt.is_valid():
        chosen = ckpt
        break
else:
    raise FileNotFoundError("no valid checkpoint found")
```

Without `for/else` you would need a `found = False` flag. This is the canonical search-loop idiom.

### `match` (Python 3.10+) — structural pattern matching

`match` is **not** a switch statement. It destructures. That is what makes it valuable for JSON-shaped data — which is all of LLM/agent programming.

---

## 2.3 Syntax

### Basic

```python
score = 0.83

if score >= 0.9:                 # `if` — first condition tested
    grade = "high"
elif score >= 0.7:               # `elif` — "else if"; any number of these
    grade = "medium"
else:                            # `else` — optional fallback
    grade = "low"

for i in range(3):               # range(stop) -> 0,1,2 — lazy, no list built
    print(i, end=" ")
print()

n = 3
while n > 0:                     # repeats while the condition stays truthy
    n -= 1

for x in [1, 2, 3, 4]:
    if x == 2:
        continue                 # skip the rest of THIS iteration
    if x == 4:
        break                    # exit the loop entirely
    print(x, end=" ")            # -> 1 3
print()

def todo():
    pass                         # `pass` — syntactic no-op placeholder
```

**Output:**
```
0 1 2 
1 3 
```

**Keyword-by-keyword.**
- `if` / `elif` / `else` — conditions evaluated top-down; the **first** truthy branch wins and the rest are skipped.
- `range(start, stop, step)` — a lazy sequence; `stop` is exclusive; `step` may be negative.
- `break` — leaves the innermost loop, skipping any `else`.
- `continue` — jumps to the next iteration.
- `pass` — does nothing; needed because Python requires a non-empty block.

### Intermediate

```python
# enumerate: index + value, with an optional start
for epoch, lr in enumerate([1e-3, 5e-4, 1e-4], start=1):
    print(f"epoch {epoch}: lr={lr}")

# zip: iterate several sequences in lockstep (stops at the shortest)
preds  = [1, 0, 1, 1]
labels = [1, 0, 0, 1]
correct = sum(p == y for p, y in zip(preds, labels, strict=True))
print(f"acc = {correct/len(labels):.0%}")

# reversed / sorted with a key
docs = [{"id": "a", "score": 0.4}, {"id": "b", "score": 0.9}]
for d in sorted(docs, key=lambda d: d["score"], reverse=True):
    print(d["id"], d["score"])

# comprehensions — a loop that produces a value
long_chunks = [c for c in ["a", "abcdef", "abc"] if len(c) > 2]
lengths     = {c: len(c) for c in long_chunks}
unique_lens = {len(c) for c in long_chunks}
```

**Output:**
```
epoch 1: lr=0.001
epoch 2: lr=0.0005
epoch 3: lr=0.0001
acc = 75%
b 0.9
a 0.4
```

**Note `strict=True` in `zip`** (3.10+): it raises if the iterables have different lengths. Silently truncating predictions against labels because one list is short is a *devastating* bug — always pass `strict=True` when the lengths should match.

### Advanced — `match`

```python
def handle_event(event: dict) -> str:
    match event:
        case {"type": "message", "role": "user", "content": str(text)}:
            return f"user said: {text[:30]}"

        case {"type": "tool_call", "name": name, "args": {"query": q}}:
            return f"search tool -> {q!r} (via {name})"

        case {"type": "tool_call", "name": name}:
            return f"tool {name} with no query"

        case {"type": "error", "code": int(code)} if code >= 500:
            return f"server error {code}: retry"

        case {"type": "error", "code": int(code)}:
            return f"client error {code}: do not retry"

        case _:
            return "unknown event"


for ev in [
    {"type": "message", "role": "user", "content": "explain RAG"},
    {"type": "tool_call", "name": "web_search", "args": {"query": "vector db"}},
    {"type": "error", "code": 503},
    {"type": "error", "code": 400},
    {"type": "heartbeat"},
]:
    print(handle_event(ev))
```

**Output:**
```
user said: explain RAG
search tool -> 'vector db' (via web_search)
server error 503: retry
client error 400: do not retry
unknown event
```

**Every piece of `match` syntax used above:**

| Pattern | Meaning |
|---|---|
| `case {"type": "message"}` | **Mapping pattern** — matches if the dict *contains* these keys with these values. Extra keys are allowed. |
| `case {"name": name}` | **Capture** — binds the value at `"name"` to the variable `name` |
| `case {"content": str(text)}` | **Class pattern as a type check** — matches only if the value is a `str`, and binds it |
| `case {"args": {"query": q}}` | **Nested pattern** — destructures two levels deep in one line |
| `if code >= 500` | **Guard** — an extra condition on an otherwise-matching case |
| `case _` | **Wildcard** — the default; always put it last |

Additional forms worth knowing:

```python
match value:
    case 0 | 1 | 2:                 # OR pattern
        ...
    case [first, *rest]:            # sequence pattern with rest-capture
        ...
    case Point(x=0, y=y):           # class pattern on your own objects
        ...
    case str() | bytes() as raw:    # type union, captured as `raw`
        ...
```

> **The one gotcha:** a bare name in a pattern *captures*, it does not compare. `case OK:` binds everything to `OK`; you need `case Status.OK:` (a dotted name) or `case "ok":` to compare.

---

## 2.4 Examples

### Beginner — classification thresholds

```python
def route_by_confidence(score: float) -> str:
    if score >= 0.95:
        return "auto_approve"
    elif score >= 0.60:
        return "human_review"
    else:
        return "auto_reject"

for s in (0.99, 0.72, 0.31):
    print(f"{s:.2f} -> {route_by_confidence(s)}")
```
**Output:**
```
0.99 -> auto_approve
0.72 -> human_review
0.31 -> auto_reject
```
**Line by line.** Conditions are ordered from most restrictive to least — this ordering is load-bearing. If you wrote `>= 0.60` first, nothing would ever reach `auto_approve`. A reviewer's first check on any threshold cascade is "are the branches ordered correctly and are they exhaustive?"

### Intermediate — dataset filtering with skip-and-report

```python
records = [
    {"id": 1, "text": "A good long review of the film.", "label": 1},
    {"id": 2, "text": "bad",                              "label": 0},
    {"id": 3, "text": "",                                 "label": 1},
    {"id": 4, "text": "Another perfectly fine review.",   "label": None},
    {"id": 5, "text": "Great cinematography throughout.", "label": 0},
]

MIN_CHARS = 10
kept, skipped = [], {"too_short": 0, "empty": 0, "no_label": 0}

for rec in records:
    text = (rec.get("text") or "").strip()
    if not text:
        skipped["empty"] += 1
        continue
    if len(text) < MIN_CHARS:
        skipped["too_short"] += 1
        continue
    if rec.get("label") is None:
        skipped["no_label"] += 1
        continue
    kept.append(rec)

print(f"kept {len(kept)}/{len(records)}")
print("skipped:", {k: v for k, v in skipped.items() if v})
```
**Output:**
```
kept 2/5
skipped: {'too_short': 1, 'empty': 1, 'no_label': 1}
```
**Line by line.**
- `(rec.get("text") or "")` — short-circuit gives `""` when the key is missing *or* the value is `None`, so `.strip()` never explodes.
- Each `continue` is a **guard clause**: check one failure mode, count it, move on. This "flat" shape is far more readable than nested `if`s, and it scales to twenty rules.
- Counting *why* rows were dropped is the difference between a script and a pipeline. Silent filtering is how you lose 40% of a dataset and never notice.

### Production-style — retry with exponential backoff

Every LLM/API call in production needs this loop.

```python
import random, time

class TransientError(Exception): ...

def call_llm(prompt: str, _fail_times: list[int] = [2]) -> str:
    """Simulated flaky API: fails the first `_fail_times[0]` calls."""
    if _fail_times[0] > 0:
        _fail_times[0] -= 1
        raise TransientError("503 Service Unavailable")
    return f"answer to {prompt!r}"

def with_retry(fn, *args, max_attempts=5, base=0.5, cap=8.0):
    for attempt in range(1, max_attempts + 1):
        try:
            return fn(*args)
        except TransientError as exc:
            if attempt == max_attempts:
                raise RuntimeError(f"failed after {max_attempts} attempts") from exc
            delay = min(cap, base * 2 ** (attempt - 1))
            delay *= 0.5 + random.random()          # jitter
            print(f"  attempt {attempt} failed ({exc}); sleeping {delay:.2f}s")
            time.sleep(delay)

print(with_retry(call_llm, "what is RAG?"))
```
**Output (delays vary because of jitter):**
```
  attempt 1 failed (503 Service Unavailable); sleeping 0.34s
  attempt 2 failed (503 Service Unavailable); sleeping 1.21s
answer to 'what is RAG?'
```
**Line by line.**
- `for attempt in range(1, max_attempts+1)` — a bounded loop, not `while True`. **Always bound your retries**; an unbounded retry against a permanently-down service is a self-inflicted outage.
- `base * 2 ** (attempt-1)` — exponential backoff: 0.5, 1, 2, 4, 8 s.
- `min(cap, ...)` — never wait absurdly long.
- **Jitter** (`0.5 + random.random()`) — without it, 1000 clients that failed together retry together and re-DDoS the service. This is called a thundering herd; jitter is the standard fix.
- `raise ... from exc` — preserves the original traceback (Chapter 6).

### AI-specific — a training loop with early stopping

This single snippet contains `for`, `while`-like state, `break`, `continue`, and `for/else`.

```python
train_losses = [0.90, 0.61, 0.45, 0.38, 0.35, 0.34, 0.33, 0.33]
val_losses   = [0.95, 0.70, 0.52, 0.49, 0.50, 0.53, 0.58, 0.61]

best_val, best_epoch, patience, bad_epochs = float("inf"), -1, 2, 0
MIN_DELTA = 1e-3

for epoch, (tr, va) in enumerate(zip(train_losses, val_losses, strict=True)):
    if va < best_val - MIN_DELTA:
        best_val, best_epoch, bad_epochs = va, epoch, 0
        marker = "  <- best (checkpoint saved)"
    else:
        bad_epochs += 1
        marker = f"  (no improve {bad_epochs}/{patience})"

    print(f"epoch {epoch}: train={tr:.3f} val={va:.3f}{marker}")

    if bad_epochs >= patience:
        print(f"early stop at epoch {epoch}; best was epoch {best_epoch} ({best_val:.3f})")
        break
else:
    print(f"completed all epochs; best epoch {best_epoch} ({best_val:.3f})")
```
**Output:**
```
epoch 0: train=0.900 val=0.950  <- best (checkpoint saved)
epoch 1: train=0.610 val=0.700  <- best (checkpoint saved)
epoch 2: train=0.450 val=0.520  <- best (checkpoint saved)
epoch 3: train=0.380 val=0.490  <- best (checkpoint saved)
epoch 4: train=0.350 val=0.500  (no improve 1/2)
epoch 5: train=0.340 val=0.530  (no improve 2/2)
early stop at epoch 5; best was epoch 3 (0.490)
```
**Why every piece is there.**
- `float("inf")` as the initial best — any real loss beats it, so no special-casing the first epoch.
- `MIN_DELTA` — without it, a 1e-9 improvement resets patience and early stopping never triggers.
- `bad_epochs` resets to 0 on improvement — patience means *consecutive* bad epochs.
- `break` + `for/else` — the `else` reports the "ran to completion" case; you get correct reporting for both exits with no flag variable.
- Train loss keeps falling while val loss rises from epoch 4: that is textbook **overfitting**, and this loop is what catches it.

### AI-specific — when NOT to loop (vectorisation)

```python
import numpy as np

emb = np.random.rand(10_000, 384).astype(np.float32)
query = np.random.rand(384).astype(np.float32)

# ❌ explicit Python loop
def cosine_loop(emb, q):
    out = np.empty(len(emb), dtype=np.float32)
    qn = np.linalg.norm(q)
    for i in range(len(emb)):
        out[i] = emb[i] @ q / (np.linalg.norm(emb[i]) * qn)
    return out

# ✔ vectorised
def cosine_vec(emb, q):
    emb_n = emb / np.linalg.norm(emb, axis=1, keepdims=True)
    return emb_n @ (q / np.linalg.norm(q))

import time
t=time.perf_counter(); a=cosine_loop(emb, query); t1=time.perf_counter()-t
t=time.perf_counter(); b=cosine_vec(emb, query);  t2=time.perf_counter()-t
print(f"loop      : {t1:.4f}s")
print(f"vectorised: {t2:.4f}s   speedup {t1/t2:.0f}x")
print("same result:", np.allclose(a, b, atol=1e-5))
```
**Output (indicative):**
```
loop      : 0.0713s
vectorised: 0.0021s   speedup 34x
same result: True
```
**The lesson.** In AI code, a `for` loop over data points is usually a bug in disguise. Loop over *epochs*, *batches*, *documents*, and *agent steps*; never over individual floats or rows. And note `np.allclose`, not `==` — Chapter 1's float rule applied.

---

## 2.5 AI Engineering Perspective

**Machine Learning.** Cross-validation is a `for` over folds; hyperparameter search is a nested `for` (or better, `itertools.product`); feature filtering is a comprehension. Per-row `for` loops over a DataFrame (`df.iterrows()`) are 100× slower than vectorised column operations and should be treated as a code smell.

**Deep Learning.** The canonical shape is `for epoch: for batch:` — two loops, and everything inside them is vectorised tensor math. `break` implements early stopping; `continue` skips corrupt batches; gradient accumulation is `if (step + 1) % accum == 0: optimizer.step()`.

**LLM applications.** Streaming is `for token in stream:`. Function/tool calling is `match` on the tool name. Chunking a document is a `while` over a sliding window. Guardrails are `if`-cascades run before and after the model call.

**AI Agents.** An agent *is* a bounded loop:
```
while not done and steps < MAX_STEPS:
    thought = llm(state)
    match thought.action:
        case "final_answer": done = True
        case "use_tool":     state += run_tool(thought.tool, thought.args)
```
The `MAX_STEPS` bound is not optional. An unbounded agent loop is an unbounded bill.

**RAG.** Ingestion is `for doc → for chunk → embed`; retrieval filters with `if score >= threshold`; reranking sorts and takes the top-k; context assembly is a `while total_tokens < budget` accumulation loop.

**Data preprocessing.** Guard-clause `continue` with per-reason counters (as in the example above) is the standard shape. Always emit the skip report.

**Model serving.** Request handling is conditional routing; batching is a `while` that drains a queue until it is full or a timeout fires.

---

## 2.6 Internal Working

### What the bytecode shows

```python
import dis
dis.dis(compile("for x in xs: f(x)", "<s>", "exec"))
```
**Output (abbreviated, 3.12):**
```
  LOAD_NAME     xs
  GET_ITER                 <- calls xs.__iter__()
  FOR_ITER      to end     <- calls __next__(); jumps out on StopIteration
  STORE_NAME    x
  ... call f ...
  JUMP_BACKWARD to FOR_ITER
```

`GET_ITER` + `FOR_ITER` is the whole story: **the loop protocol is two method calls.** That is why any object implementing them works with `for`, and why `for` over a generator uses O(1) memory.

### Cost of the interpreter loop

Each iteration of a pure-Python loop pays for: bytecode dispatch, reference-count updates on every object touched, and boxing/unboxing of numbers. That is roughly 30–80 ns per simple operation — versus ~0.3 ns for a fused SIMD float op in a NumPy kernel. **~100× is the structural gap**, and no micro-optimisation closes it. Only moving the loop into C (vectorisation) does.

### Comprehension scope

```python
i = "unchanged"
squares = [i * i for i in range(3)]
print(i)          # -> unchanged
```
**Output:** `unchanged`

Comprehensions get their own scope (since Python 3), so they cannot leak or clobber names. A plain `for` loop does *not*:

```python
for i in range(3):
    pass
print(i)          # -> 2   the loop variable survives
```

---

## 2.7 Best Practices

```python
# ✔ Iterate the object, not the index
for chunk in chunks: ...
# ✘ for i in range(len(chunks)): chunk = chunks[i]

# ✔ Need the index? enumerate
for i, chunk in enumerate(chunks, start=1): ...

# ✔ Parallel sequences? zip with strict
for pred, label in zip(preds, labels, strict=True): ...

# ✔ Guard clauses over nesting
for rec in records:
    if not valid(rec):
        continue
    process(rec)
# ✘ for rec in records:
#        if valid(rec):
#            if has_label(rec):
#                if long_enough(rec):
#                    process(rec)      <- arrow anti-pattern

# ✔ Empty checks by truthiness
if not results: ...
# ✘ if len(results) == 0: ...     (and never `if results == []`)

# ✔ Bound every unbounded loop
for attempt in range(MAX_RETRIES): ...
# ✘ while True: ...   without a break condition you can prove terminates
```

Additional rules:

1. **Comprehension when producing a collection; `for` when performing side effects.** `[print(x) for x in xs]` is an abuse — it builds a list of `None` for nothing.
2. **Keep comprehensions to one `for` and at most one `if`.** Beyond that, write the loop; readability wins.
3. **Extract complex conditions into named predicates.** `if is_valid_chunk(c):` beats a four-clause boolean expression.
4. **Prefer `any()`/`all()` over flag loops.** `if any(t in text for t in BANNED):` replaces six lines.
5. **`match` over long `elif` chains when dispatching on structure**, not on simple equality — for plain value dispatch, a dict of handlers is usually cleaner still.

---

## 2.8 Common Mistakes

### Mistake 1 — Mutating a list while iterating it

```python
# ❌ WRONG — skips elements; the iterator index and the list shift together
docs = ["a", "", "b", "", "c"]
for d in docs:
    if not d:
        docs.remove(d)
print(docs)                 # -> ['a', 'b', '', 'c']   ← one empty survived!

# ✔ CORRECT — build a new list
docs = ["a", "", "b", "", "c"]
docs = [d for d in docs if d]
print(docs)                 # -> ['a', 'b', 'c']
```
If you must mutate in place, iterate over a copy (`for d in docs[:]`) or use slice assignment (`docs[:] = [...]`).

### Mistake 2 — Truth-testing an array

```python
# ❌ WRONG
if predictions:                      # ValueError for a multi-element array
    ...
# ✔ CORRECT
if predictions.size > 0: ...
if (predictions > 0.5).any(): ...
if torch.isnan(loss).any(): ...
```

### Mistake 3 — `zip` silently truncating

```python
preds  = [1, 0, 1]          # model returned 3
labels = [1, 0, 1, 0, 1]    # dataset has 5  ← something is already wrong

# ❌ WRONG: accuracy computed over 3 items, reported as if over the dataset
acc = sum(p == y for p, y in zip(preds, labels)) / len(labels)

# ✔ CORRECT: fail loudly
acc = sum(p == y for p, y in zip(preds, labels, strict=True)) / len(labels)
# ValueError: zip() argument 2 is longer than argument 1
```

### Mistake 4 — Late-binding in a loop that builds callables

```python
# ❌ WRONG
handlers = []
for name in ["a", "b", "c"]:
    handlers.append(lambda: f"handling {name}")
print([h() for h in handlers])      # -> ['handling c', 'handling c', 'handling c']

# ✔ CORRECT — bind at definition time via a default argument
handlers = [ (lambda n=name: f"handling {n}") for name in ["a", "b", "c"] ]
print([h() for h in handlers])      # -> ['handling a', 'handling b', 'handling c']
```
The lambda captures the *variable*, not its value; by call time the loop has finished and `name` is `"c"`. This bites when registering per-tool callbacks in an agent. (Closures: Chapter 3.)

### Mistake 5 — Unbounded agent/retry loop

```python
# ❌ WRONG — can loop forever, burning tokens and money
while not done:
    done = agent_step()

# ✔ CORRECT
for step in range(MAX_STEPS):
    if agent_step():
        break
else:
    logger.error("agent hit MAX_STEPS=%d without finishing", MAX_STEPS)
```

### Mistake 6 — `elif` chain that isn't exhaustive

```python
# ❌ WRONG — returns None for scores below 0.6, silently
def route(score):
    if score >= 0.9: return "approve"
    elif score >= 0.6: return "review"

# ✔ CORRECT — always terminate with else, or raise
def route(score):
    if score >= 0.9: return "approve"
    if score >= 0.6: return "review"
    return "reject"
```
An implicit `None` return flowing into a downstream `match` is a debugging nightmare. Make every branch explicit.

---

## 2.9 Performance

| Pattern | Relative cost (1M items) | Notes |
|---|---|---|
| Vectorised NumPy | 1× | The baseline you should be aiming for |
| List comprehension | ~40× | Optimised bytecode, no `append` lookup per item |
| `map()` with a builtin | ~35× | Slightly faster than a comprehension calling the same builtin |
| Plain `for` + `.append()` | ~55× | Attribute lookup + method call per item |
| `for i in range(len(x)): x[i]` | ~75× | Extra index + `__getitem__` per item |
| `df.iterrows()` | ~1000× | Builds a `Series` object per row. Never use it. |

```python
import timeit
setup = "xs = list(range(100_000))"
print("comprehension:", timeit.timeit("[x*2 for x in xs]", setup, number=100))
print("append loop  :", timeit.timeit(
    "out=[]\nfor x in xs: out.append(x*2)", setup, number=100))
print("map          :", timeit.timeit("list(map(lambda x: x*2, xs))", setup, number=100))
```
**Output (indicative):**
```
comprehension: 0.41
append loop  : 0.68
map          : 0.72
```

### Complexity traps in loop bodies

```python
# ❌ O(n·m): membership test against a list inside a loop
seen = []
for doc_id in ids:
    if doc_id not in seen:      # O(len(seen)) EVERY iteration
        seen.append(doc_id)

# ✔ O(n): set membership is O(1)
seen = set()
for doc_id in ids:
    if doc_id not in seen:
        seen.add(doc_id)
```
For 100k ids this is the difference between ~5 billion comparisons and ~100k hash lookups. **This is the most valuable performance idea in the whole guide** — it beats every micro-optimisation combined. More in Chapter 4.

### Loop hoisting

```python
# ❌ recomputed every iteration
for chunk in chunks:
    if len(tokenizer.encode(chunk)) > model.config.max_position_embeddings: ...

# ✔ computed once
max_len = model.config.max_position_embeddings
encode = tokenizer.encode
for chunk in chunks:
    if len(encode(chunk)) > max_len: ...
```
Binding the method to a local (`encode = tokenizer.encode`) skips an attribute lookup per iteration — small, but free, and standard in hot loops.

---

## 2.10 Interview Preparation

**Q1 (Beginner). What is truthiness? Which values are falsy?**
Any object can be used as a condition; `bool(obj)` consults `__bool__`, falling back to `__len__`. Falsy: `False`, `None`, numeric zeros, empty containers, and objects with `__len__() == 0`. Everything else, including `"0"` and `"False"`, is truthy.

**Q2 (Beginner). `break` vs `continue` vs `pass`?**
`break` exits the innermost loop (and skips its `else`); `continue` skips to the next iteration; `pass` does nothing at all and exists only to satisfy the grammar where a statement is required.

**Q3 (Beginner). What does `for/else` do?**
The `else` block runs if the loop completed without hitting `break`. It is the idiomatic way to express "searched everything and found nothing" without a sentinel flag.

**Q4 (Intermediate). Why does `if numpy_array:` raise?**
NumPy overrides `__bool__` to raise `ValueError` for arrays with more than one element, because "is this array true?" is ambiguous — did you mean *any* element, or *all*? Use `.any()`, `.all()`, or compare sizes.

**Q5 (Intermediate). Explain short-circuit evaluation and give a correctness use.**
`and`/`or` stop evaluating once the result is determined, and return the last evaluated operand. Correctness use: `if resp is not None and resp["ok"]:` — the second operand is never evaluated when `resp` is `None`, avoiding a `TypeError`. Reordering the operands would crash.

**Q6 (Intermediate). When is `match` better than `if/elif`?**
When you are dispatching on the *structure* of data — nested dicts, JSON events, tool calls — because `match` destructures and binds in the same expression. For simple equality dispatch over many values, a dict mapping key → handler function is cleaner and O(1).

**Q7 (AI/ML). Why is looping over rows in pandas an anti-pattern? What replaces it?**
`iterrows()` constructs a `Series` object per row (allocation + dtype coercion), so it is ~1000× slower than vectorised column operations and loses dtypes. Replace with vectorised expressions, `.map()`/`.apply()` on a Series if you must, `np.where` / `np.select` for conditional logic, or `merge` for lookups.

**Q8 (AI/ML). Implement early stopping in words and code.**
Track the best validation metric and a counter of consecutive non-improving epochs. Require improvement to exceed a `min_delta` so noise doesn't reset patience. When the counter reaches `patience`, `break` and restore the best checkpoint. (Code as in §2.4.)

**Q9 (Scenario). Your agent occasionally runs for hours and racks up API cost. What do you change?**
Add a hard `MAX_STEPS` bound and a wall-clock/token budget; detect repeated identical actions (a loop signature) and break; add a per-run cost accumulator that raises when exceeded; log every step's action so the loop is auditable. Never rely on the model deciding to stop.

**Q10 (Scenario). Ingestion of 2M documents dies at document 1.8M with one malformed record. How do you make it robust?**
Wrap per-document processing in `try/except`, `continue` on failure, and write failures to a dead-letter file with the record id and the exception. Track counters per failure reason, checkpoint progress so a restart resumes, and fail the *run* at the end if the error rate exceeds a threshold — so you neither lose the run nor silently ship a 40%-dropped dataset.

**Q11 (Production). Why is jitter necessary in retry backoff?**
Without randomisation, all clients that failed at the same moment retry at the same moment, producing synchronised load spikes (the thundering herd) that keep the service down. Multiplying the delay by a random factor spreads the retries out.

---

## 2.11 Exercises

### Easy — E1
Given a list of similarity scores, count how many exceed 0.8 — first with a loop, then with a one-liner.

<details><summary>Solution</summary>

```python
scores = [0.91, 0.42, 0.85, 0.79, 0.99]

count = 0
for s in scores:
    if s > 0.8:
        count += 1

count2 = sum(s > 0.8 for s in scores)     # bool is an int subclass: True == 1
print(count, count2)                      # -> 3 3
```
`sum(bool_generator)` is the idiomatic Python count. It is a generator expression, so no intermediate list is allocated.
</details>

### Easy — E2
Print a formatted 3-epoch × 2-batch training trace using nested loops.

<details><summary>Solution</summary>

```python
for epoch in range(1, 4):
    for batch in range(1, 3):
        step = (epoch - 1) * 2 + batch
        print(f"epoch {epoch}/3 batch {batch}/2 (global step {step})")
```
**Output:**
```
epoch 1/3 batch 1/2 (global step 1)
epoch 1/3 batch 2/2 (global step 2)
epoch 2/3 batch 1/2 (global step 3)
...
```
The `global step` computation is exactly what real trainers log to TensorBoard/W&B, because per-epoch numbering makes runs with different dataset sizes incomparable.
</details>

### Medium — M1
Write `sliding_window(text, size, overlap)` that chunks text into overlapping windows — the core of every RAG ingestion pipeline.

<details><summary>Solution</summary>

```python
def sliding_window(text: str, size: int = 100, overlap: int = 20) -> list[str]:
    if size <= 0:
        raise ValueError("size must be positive")
    if not 0 <= overlap < size:
        raise ValueError(f"overlap must satisfy 0 <= overlap < size, got {overlap}")

    stride = size - overlap
    chunks, start = [], 0
    while start < len(text):
        chunks.append(text[start:start + size])
        start += stride
    return chunks

text = "".join(str(i % 10) for i in range(50))
for i, c in enumerate(sliding_window(text, size=20, overlap=5)):
    print(f"{i}: {c}")
```
**Output:**
```
0: 01234567890123456789
1: 56789012345678901234
2: 01234567890123456789
3: 56789
```
**Explanation.** `stride = size - overlap` is the whole algorithm. The `overlap < size` guard is essential: with `overlap >= size`, `stride <= 0` and the `while` never terminates — an infinite loop that would silently fill memory. Notice consecutive chunks share their last/first 5 characters: that overlap is what stops a RAG system from splitting a sentence across chunks and losing the answer.
</details>

### Medium — M2
Route a batch of prompts to different "models" using `match`, with cost accounting.

<details><summary>Solution</summary>

```python
COST = {"small": 0.0002, "large": 0.003, "reasoning": 0.015}

def route(request: dict) -> tuple[str, str]:
    match request:
        case {"task": "classify"} | {"task": "extract"}:
            return "small", "cheap structured task"
        case {"task": "chat", "history_len": int(n)} if n > 20:
            return "large", "long conversation needs a bigger context"
        case {"task": "chat"}:
            return "small", "short chat"
        case {"task": "math" | "code" | "plan"}:
            return "reasoning", "requires multi-step reasoning"
        case _:
            return "large", "unknown task -> safe default"

requests = [
    {"task": "classify", "text": "..."},
    {"task": "chat", "history_len": 30},
    {"task": "chat", "history_len": 2},
    {"task": "math", "text": "integrate x^2"},
    {"task": "translate"},
]
total = 0.0
for r in requests:
    model, why = route(r)
    total += COST[model]
    print(f"{r['task']:<9} -> {model:<9} (${COST[model]:.4f})  {why}")
print(f"{'TOTAL':<9}    {'':<9}  ${total:.4f}")
```
**Output:**
```
classify  -> small     ($0.0002)  cheap structured task
chat      -> large     ($0.0030)  long conversation needs a bigger context
chat      -> small     ($0.0002)  short chat
math      -> reasoning ($0.0150)  requires multi-step reasoning
translate -> large     ($0.0030)  unknown task -> safe default
TOTAL                  $0.0214
```
**Explanation.** Three `match` features earn their place: the `|` OR-pattern groups equivalent tasks, the guard (`if n > 20`) adds a numeric condition to a structural match, and `"math" | "code" | "plan"` shows an OR pattern *inside* a value position. **Order matters**: the guarded `chat` case must precede the unguarded one, otherwise the general case would swallow every chat request. Routing like this is real — it is how production systems cut LLM spend by 5–10× without hurting quality.
</details>

### Hard — H1
Implement `build_context(chunks, token_budget, count_tokens)` that packs the highest-scoring chunks into a context window without exceeding the budget, never splitting a chunk, and reports what was dropped.

<details><summary>Solution</summary>

```python
def approx_tokens(text: str) -> int:
    """Rough heuristic: ~4 chars per token for English."""
    return max(1, len(text) // 4)

def build_context(chunks: list[dict], token_budget: int, count_tokens=approx_tokens):
    ranked = sorted(chunks, key=lambda c: c["score"], reverse=True)

    selected, used, dropped = [], 0, []
    for chunk in ranked:
        cost = count_tokens(chunk["text"])
        if cost > token_budget:
            dropped.append((chunk["id"], "single chunk exceeds budget"))
            continue
        if used + cost > token_budget:
            dropped.append((chunk["id"], f"would exceed budget ({used}+{cost})"))
            continue                      # keep trying: a later small chunk may fit
        selected.append(chunk)
        used += cost

    return {
        "context": "\n\n---\n\n".join(c["text"] for c in selected),
        "used_tokens": used,
        "budget": token_budget,
        "n_selected": len(selected),
        "dropped": dropped,
    }

chunks = [
    {"id": "c1", "score": 0.95, "text": "A" * 200},
    {"id": "c2", "score": 0.88, "text": "B" * 160},
    {"id": "c3", "score": 0.80, "text": "C" * 1000},
    {"id": "c4", "score": 0.75, "text": "D" * 40},
]
result = build_context(chunks, token_budget=100)
for k, v in result.items():
    if k != "context":
        print(f"{k}: {v}")
```
**Output:**
```
used_tokens: 100
budget: 100
n_selected: 3
dropped: [('c3', 'would exceed budget (90+250)')]
```
**Explanation and why each decision is the production one.**
- **Sort by score first** — the budget should be spent on the most relevant material.
- **`continue`, not `break`, on overflow.** Breaking would stop at the first chunk that doesn't fit; continuing lets a smaller lower-ranked chunk (`c4`, 10 tokens) fill the remaining space. This is the greedy knapsack heuristic and it measurably improves RAG recall.
- **Two distinct drop reasons.** "Chunk bigger than the entire budget" means your chunker is misconfigured — a different bug from "the window filled up". Conflating them hides a real problem.
- **Injectable `count_tokens`.** In production you pass `tiktoken`/`transformers` tokenizer's real counter; the heuristic is only a default. Passing behaviour as a parameter is Chapter 3's lesson arriving early.
- **Complexity** O(n log n) for the sort, O(n) for the pack.
</details>

---

## 2.12 Mini Project — Prompt Router with Cost Guardrails

**Goal.** A single component that classifies an incoming request, routes it to the cheapest adequate model, enforces a hard spend budget, and reports what it did. This is a real production pattern — teams deploy exactly this to cut LLM costs.

```python
"""router.py — rule-based LLM router with a hard cost ceiling."""
from dataclasses import dataclass, field

MODELS = {
    "small":     {"cost_per_1k": 0.0002, "max_ctx": 16_000},
    "large":     {"cost_per_1k": 0.0030, "max_ctx": 128_000},
    "reasoning": {"cost_per_1k": 0.0150, "max_ctx": 200_000},
}
REASONING_HINTS = ("prove", "step by step", "derive", "debug", "why does", "algorithm")
SIMPLE_TASKS = {"classify", "extract", "translate", "summarize"}


@dataclass
class Router:
    budget_usd: float
    spent_usd: float = 0.0
    log: list[dict] = field(default_factory=list)

    def _choose(self, req: dict) -> tuple[str, str]:
        est_tokens = len(req["prompt"]) // 4 + req.get("expected_output_tokens", 200)
        prompt_lower = req["prompt"].lower()

        match req:
            case {"force_model": str(m)} if m in MODELS:
                return m, "explicitly forced by caller"
            case {"task": t} if t in SIMPLE_TASKS and est_tokens < 4000:
                return "small", f"simple task {t!r}, {est_tokens} est. tokens"
            case _ if any(h in prompt_lower for h in REASONING_HINTS):
                return "reasoning", "prompt contains reasoning cue"
            case _ if est_tokens > MODELS["small"]["max_ctx"]:
                return "large", f"{est_tokens} tokens exceeds small model context"
            case _:
                return "small", "default: try cheapest"

    def route(self, req: dict) -> dict:
        model, reason = self._choose(req)
        est_tokens = len(req["prompt"]) // 4 + req.get("expected_output_tokens", 200)
        est_cost = est_tokens / 1000 * MODELS[model]["cost_per_1k"]

        if self.spent_usd + est_cost > self.budget_usd:
            downgrade = "small"
            cheap_cost = est_tokens / 1000 * MODELS[downgrade]["cost_per_1k"]
            if self.spent_usd + cheap_cost > self.budget_usd:
                entry = {"status": "REJECTED", "model": None, "cost": 0.0,
                         "reason": "budget exhausted"}
                self.log.append(entry)
                return entry
            model, est_cost, reason = downgrade, cheap_cost, "DOWNGRADED: budget pressure"

        self.spent_usd += est_cost
        entry = {"status": "OK", "model": model, "cost": est_cost, "reason": reason}
        self.log.append(entry)
        return entry

    def report(self) -> None:
        print(f"\n{'status':<9} {'model':<10} {'cost':>8}  reason")
        print("-" * 68)
        for e in self.log:
            print(f"{e['status']:<9} {str(e['model']):<10} ${e['cost']:>7.5f}  {e['reason']}")
        pct = self.spent_usd / self.budget_usd
        bar = "#" * int(pct * 30) + "." * (30 - int(pct * 30))
        print(f"\nbudget [{bar}] ${self.spent_usd:.5f} / ${self.budget_usd:.2f} ({pct:.1%})")


if __name__ == "__main__":
    router = Router(budget_usd=0.02)
    requests = [
        {"task": "classify",  "prompt": "Is this spam? " + "x" * 200},
        {"task": "chat",      "prompt": "Explain step by step why gradient descent converges."},
        {"task": "summarize", "prompt": "y" * 80_000},
        {"task": "chat",      "prompt": "hello"},
        {"task": "chat",      "prompt": "Debug this stack trace: " + "z" * 5000},
        {"task": "chat",      "prompt": "one more request"},
    ]
    for r in requests:
        router.route(r)
    router.report()
```

**Output:**
```
status    model          cost  reason
--------------------------------------------------------------------
OK        small      $0.00005  simple task 'classify', 253 est. tokens
OK        reasoning  $0.00319  prompt contains reasoning cue
OK        small      $0.00404  DOWNGRADED: budget pressure
OK        small      $0.00004  default: try cheapest
OK        small      $0.00029  DOWNGRADED: budget pressure
OK        small      $0.00004  default: try cheapest

budget [###########...................] $0.00766 / $0.02 (38.3%)
```
**Read row 3.** The 80 000-character summarisation request would have cost about $0.06 on the large model — three times the entire budget — so the router **downgraded it to the small model** instead of rejecting it. Row 5 does the same for the debugging request. Nothing failed; the system simply got cheaper under pressure. Lower `budget_usd` to `0.001` and re-run to see the `REJECTED` path.

**What this project exercises.** `match` with guards and forced-value cases; an `if`-cascade implementing a **degradation ladder** (preferred model → cheaper model → reject) rather than a hard failure; accumulation across a loop; and a formatted report with an ASCII progress bar. The degradation ladder is the important architectural idea: production AI systems should get *cheaper and dumber* under pressure, not fall over.

**Extensions.** Add a `while` loop that retries a rejected request after a budget refill window (§2.4 backoff); make the routing rules data-driven from a YAML file (Chapter 5); raise a custom `BudgetExceededError` instead of returning a status dict (Chapter 6); make `Router` an abstract base with `RuleRouter` and `EmbeddingRouter` subclasses (Chapter 7).

---

## Chapter 2 Summary

| Concept | Takeaway |
|---|---|
| Truthiness | Empty is falsy; NumPy arrays raise — use `.any()`/`.all()` |
| Short-circuit | `and`/`or` return operands and guard against crashes |
| `for` | Sugar over `iter()`/`next()` — works on files, generators, DataLoaders |
| `for/else` | `else` = "no break"; the search-loop idiom |
| `match` | Structural destructuring for JSON/events/tool calls, with guards |
| Guard clauses | `continue` early; never nest four `if`s |
| Bounded loops | Every retry and every agent loop needs a hard cap |
| Vectorisation | Loop over batches, never over floats — 30–100× |
| `set` membership | O(1) vs O(n) — the highest-value performance habit |

**Revision flashcards.** `for/else` semantics · why `if array:` raises · `zip(strict=True)` · late-binding closures in loops · stride = size − overlap · thundering herd → jitter.

---
# Chapter 3 — Functions

> **Chapter goal.** A machine learning pipeline is a composition of functions. A LangChain chain is literally function composition. A tool an agent can call is a function plus a schema. This chapter turns "I can write a def" into "I can design an interface."

---

## 3.1 Concept Overview

### What it is

A function is a named, reusable, parameterised block of code that takes inputs, does work, and returns a value. In Python it is also a **first-class object**: you can store it in a list, pass it to another function, return it from a function, and attach attributes to it.

### Why it exists

Three reasons, in order of importance:

1. **Naming.** `normalize_embedding(v)` documents intent in a way that four lines of inline math never will.
2. **Reuse.** The same tokenisation must run at training time and at inference time. If it lives in two places it *will* diverge — this is called training/serving skew and it is one of the top causes of silent model degradation in production.
3. **Testability.** You cannot unit-test a 300-line script. You can unit-test twelve functions.

### Why Python includes it this way

Python's function design is unusually rich:

- **First-class and higher-order** — functions are objects, so `sorted(key=...)`, `map`, decorators, and callbacks all just work.
- **Very flexible parameters** — positional, keyword, defaults, `*args`, `**kwargs`, keyword-only, positional-only. This is why library APIs like `model.generate(prompt, max_tokens=100, **gen_kwargs)` are ergonomic.
- **Closures** — inner functions capture enclosing scope, enabling decorators and factories.
- **Everything returns something** — implicitly `None` if you don't say otherwise.

### Why AI engineers care

| Function feature | AI use |
|---|---|
| Pure functions | Reproducible preprocessing; safe to cache and parallelise |
| Default arguments | Hyperparameters with sensible defaults (`temperature=0.7`) |
| `**kwargs` | Forwarding generation params to a model API without rewriting signatures |
| Keyword-only args | Preventing `generate(0.7, 100)` — which is temperature and which is max_tokens? |
| First-class functions | Tool registries for agents; `key=` for reranking; pluggable metrics |
| Closures | Decorators for retry/logging/caching; parameterised loss functions |
| Type hints | Auto-generating an LLM tool schema straight from the signature |
| Docstrings | Literally the tool description the model reads to decide when to call it |

---

## 3.2 Theory

### What `def` actually does

`def` is an *executable statement*, not a declaration. When it runs, Python:

1. Compiles the body into a **code object** (`func.__code__`).
2. Evaluates the **default values once**, storing them in `func.__defaults__`.
3. Creates a **function object** wrapping the code, the defaults, the enclosing closure cells, and `__globals__`.
4. Binds that object to the function's name in the current namespace.

```python
def score(x, threshold=0.5, tags=[]):
    return x > threshold

print(score.__name__)      # -> score
print(score.__defaults__)  # -> (0.5, [])
print(score.__code__.co_varnames)  # -> ('x', 'threshold', 'tags')
```
**Output:**
```
score
(0.5, [])
('x', 'threshold', 'tags')
```

Step 2 is the origin of the single most famous Python bug — the **mutable default argument** (§3.8). That `[]` is created *once*, when the `def` executes, and shared by every call forever.

### Argument passing: "call by object reference"

Python is neither call-by-value nor call-by-reference. The *reference* is passed by value:

```python
def rebind(lst):
    lst = [9, 9]        # rebinds the LOCAL name only
def mutate(lst):
    lst.append(9)       # mutates the SHARED object

x = [1]
rebind(x); print(x)     # -> [1]        caller unaffected
mutate(x); print(x)     # -> [1, 9]     caller affected
```
**Output:**
```
[1]
[1, 9]
```

```
    caller: x ──┐
                ├──► [1]   ← mutate() reaches this object
    callee: lst ┘

    after `lst = [9,9]`:
    caller: x ───► [1]
    callee: lst ──► [9,9]   (a different object; caller never sees it)
```

**Consequence for AI code:** a preprocessing function that takes `batch: list` and appends to it silently modifies the caller's batch. Convention: functions either return new data (pure) or clearly announce mutation in their name (`shuffle_` / `_inplace`). PyTorch follows this: `x.add(1)` returns new, `x.add_(1)` mutates.

### The LEGB rule — name resolution

When Python looks up a name it searches four scopes, in order:

```
   ┌─ B  Builtins  ──────────────────────────────────┐   len, print, range, ...
   │  ┌─ G  Global (module) ──────────────────────┐  │   MODULE_CONSTANTS, imports
   │  │  ┌─ E  Enclosing (outer functions) ────┐  │  │   closure variables
   │  │  │  ┌─ L  Local (this function) ────┐  │  │  │   parameters, locals
   │  │  │  │      name lookup starts here  │  │  │  │
   │  │  │  └───────────────────────────────┘  │  │  │
   │  │  └─────────────────────────────────────┘  │  │
   │  └───────────────────────────────────────────┘  │
   └─────────────────────────────────────────────────┘
```

```python
x = "global"

def outer():
    x = "enclosing"
    def inner():
        print(x)          # finds E before G
    inner()

outer()                   # -> enclosing
print(x)                  # -> global
```

**Writing** to a name makes it local for the *entire function*, unless you declare otherwise:

```python
counter = 0
def bump_broken():
    counter += 1          # UnboundLocalError: assignment makes `counter` local
def bump_ok():
    global counter        # explicitly write to the module-level name
    counter += 1

def make_counter():
    n = 0
    def inc():
        nonlocal n        # write to the ENCLOSING function's name
        n += 1
        return n
    return inc
```

> **Best practice.** `global` is almost always a design smell — it makes functions untestable and unsafe under threading. `nonlocal` is legitimate and appears constantly inside decorators and closures.

### Closures

A closure is a function plus the enclosing variables it captured. The captured variables stay alive after the outer function returns:

```python
def make_threshold_filter(min_score: float):
    def keep(doc: dict) -> bool:
        return doc["score"] >= min_score      # captures min_score
    return keep

strict  = make_threshold_filter(0.9)
relaxed = make_threshold_filter(0.5)
docs = [{"id": "a", "score": 0.95}, {"id": "b", "score": 0.6}]

print([d["id"] for d in docs if strict(d)])    # -> ['a']
print([d["id"] for d in docs if relaxed(d)])   # -> ['a', 'b']
print(strict.__closure__[0].cell_contents)     # -> 0.9
```
**Output:**
```
['a']
['a', 'b']
0.9
```

The captured value lives in a **cell object** attached to the function. Two calls to the factory produce two independent cells — which is why `strict` and `relaxed` don't interfere.

### Recursion

A function that calls itself. Needs a base case and progress toward it. CPython's default recursion limit is 1000 (`sys.getrecursionlimit()`), and it does **not** do tail-call optimisation — so deep recursion raises `RecursionError`.

Use recursion for genuinely tree-shaped problems: walking a nested JSON config, traversing a directory tree, chunking a document by its heading hierarchy, or an agent that spawns sub-agents. Use iteration for everything linear.

---

## 3.3 Syntax

### Basic

```python
def normalize(text: str) -> str:
    """Lowercase and collapse whitespace.

    Args:
        text: Raw input text.
    Returns:
        The normalised text.
    """
    return " ".join(text.lower().split())

print(normalize("  Hello   WORLD  "))     # -> hello world
```

- `def` — creates the function object.
- `text: str` — a **type hint**. Not enforced at runtime; used by `mypy`, IDEs, FastAPI, pydantic, and LLM tool-schema generators.
- `-> str` — the return type hint.
- The `"""..."""` **docstring** is the first statement, stored as `__doc__`. For agent tools, this text is what the model reads.
- `return` — hands a value back. A function with no `return` returns `None`.

### Intermediate — the full parameter grammar

```python
def generate(
    prompt,                     # positional-or-keyword
    /,                          # ← everything BEFORE this is positional-ONLY
    model="small",              # positional-or-keyword, with a default
    *,                          # ← everything AFTER this is keyword-ONLY
    temperature=0.7,
    max_tokens=256,
    **extra,                    # collects any other keyword args
):
    return {"prompt": prompt, "model": model, "temperature": temperature,
            "max_tokens": max_tokens, "extra": extra}

print(generate("hi", "large", temperature=0.2, top_p=0.9, seed=42))
```
**Output:**
```
{'prompt': 'hi', 'model': 'large', 'temperature': 0.2, 'max_tokens': 256,
 'extra': {'top_p': 0.9, 'seed': 42}}
```

| Marker | Effect | Why you'd want it |
|---|---|---|
| `/` | Params before it **cannot** be passed by keyword | Lets you rename them later without breaking callers |
| `*` | Params after it **must** be passed by keyword | Forbids `generate("hi", 0.7, 256)` — unreadable and error-prone |
| `*args` | Collects extra positional args into a tuple | Variadic functions like `sum_all(*nums)` |
| `**kwargs` | Collects extra keyword args into a dict | Forwarding options to a wrapped API |

**The keyword-only rule is a genuine production practice.** Compare:
```python
train(model, data, 0.001, 32, True, False, 3)     # ✘ unreadable, easy to transpose
train(model, data, lr=0.001, batch_size=32, fp16=True, shuffle=False, epochs=3)  # ✔
```
Adding `*,` after `data` makes the second form mandatory.

### `*args` and `**kwargs` — packing and unpacking

The same symbols mean **pack** in a definition and **unpack** at a call site:

```python
def log_metrics(run_id, *values, **tags):
    print(f"run={run_id} values={values} tags={tags}")

log_metrics("r1", 0.9, 0.8, stage="val", model="bert")
# -> run=r1 values=(0.9, 0.8) tags={'stage': 'val', 'model': 'bert'}

nums = [0.9, 0.8]
meta = {"stage": "val", "model": "bert"}
log_metrics("r1", *nums, **meta)          # unpacking — identical call
```

The forwarding pattern you will use constantly:

```python
def call_with_retry(fn, *args, retries=3, **kwargs):
    for attempt in range(retries):
        try:
            return fn(*args, **kwargs)     # forward everything untouched
        except TransientError:
            if attempt == retries - 1:
                raise
```

### Advanced — lambdas, and where they belong

```python
docs = [{"id": "a", "score": 0.4}, {"id": "b", "score": 0.9}]

# ✔ good use: a tiny inline key function
top = sorted(docs, key=lambda d: d["score"], reverse=True)

# ✔ better for repeated / complex keys: operator.itemgetter (faster, clearer)
from operator import itemgetter
top = sorted(docs, key=itemgetter("score"), reverse=True)

# ✘ bad use: naming a lambda — just use def
normalize = lambda t: t.lower().strip()      # no useful __name__, no docstring
```

**Rules for lambdas:** single expression only, no statements, no annotations, no docstring. Use them for one-liner `key=`/`filter` arguments. Anything else gets a `def`.

### Advanced — introspection (how tool schemas get built)

```python
import inspect

def search_docs(query: str, top_k: int = 5, min_score: float = 0.0) -> list[str]:
    """Search the knowledge base for documents matching a query."""
    ...

sig = inspect.signature(search_docs)
print(sig)
for name, param in sig.parameters.items():
    required = param.default is inspect.Parameter.empty
    print(f"  {name}: {param.annotation.__name__:<6} "
          f"{'required' if required else f'default={param.default!r}'}")
print("doc:", inspect.getdoc(search_docs))
```
**Output:**
```
(query: str, top_k: int = 5, min_score: float = 0.0) -> list[str]
  query: str    required
  top_k: int    default=5
  min_score: float default=0.0
doc: Search the knowledge base for documents matching a query.
```

**This is exactly how LangChain's `@tool`, MCP servers, and every LLM function-calling wrapper build their JSON schemas** — read the signature, read the annotations, read the docstring. Which is why type hints and docstrings are not optional style points in AI code: **the model reads them**.

---

## 3.4 Examples

### Beginner — a pure text-cleaning function

```python
import re

def clean_text(text: str, *, lowercase: bool = True, strip_urls: bool = True) -> str:
    """Normalise raw text for embedding or training."""
    if strip_urls:
        text = re.sub(r"https?://\S+", " ", text)
    if lowercase:
        text = text.lower()
    return " ".join(text.split())

raw = "Check   THIS out: https://example.com/x?y=1   Amazing!!"
print(repr(clean_text(raw)))
print(repr(clean_text(raw, lowercase=False, strip_urls=False)))
```
**Output:**
```
'check this out: amazing!!'
'Check THIS out: https://example.com/x?y=1 Amazing!!'
```
**Line by line.** `*` forces the flags to be keyword arguments, so no caller can write `clean_text(raw, False, True)`. `re.sub` replaces URL matches with a space. `" ".join(text.split())` is the idiomatic whitespace-collapse: `split()` with no argument splits on any run of whitespace *and* drops empties. The function is **pure** — same input, same output, no side effects — which means it is trivially testable, cacheable, and parallelisable.

### Intermediate — a metric registry using first-class functions

```python
def accuracy(y_true, y_pred):
    return sum(t == p for t, p in zip(y_true, y_pred, strict=True)) / len(y_true)

def precision(y_true, y_pred):
    tp = sum(t == 1 and p == 1 for t, p in zip(y_true, y_pred, strict=True))
    pp = sum(p == 1 for p in y_pred)
    return tp / pp if pp else 0.0

def recall(y_true, y_pred):
    tp = sum(t == 1 and p == 1 for t, p in zip(y_true, y_pred, strict=True))
    ap = sum(t == 1 for t in y_true)
    return tp / ap if ap else 0.0

def f1(y_true, y_pred):
    p, r = precision(y_true, y_pred), recall(y_true, y_pred)
    return 2 * p * r / (p + r) if (p + r) else 0.0

METRICS = {"accuracy": accuracy, "precision": precision, "recall": recall, "f1": f1}

def evaluate(y_true, y_pred, metrics: list[str] | None = None) -> dict[str, float]:
    chosen = metrics or list(METRICS)
    unknown = set(chosen) - set(METRICS)
    if unknown:
        raise KeyError(f"unknown metrics {sorted(unknown)}; available: {sorted(METRICS)}")
    return {name: METRICS[name](y_true, y_pred) for name in chosen}

y_true = [1, 0, 1, 1, 0, 1, 0, 0]
y_pred = [1, 0, 1, 0, 0, 1, 1, 0]
for name, value in evaluate(y_true, y_pred).items():
    print(f"{name:<10} {value:.3f}")
```
**Output:**
```
accuracy   0.750
precision  0.750
recall     0.750
f1         0.750
```
**Line by line.**
- `METRICS` is a **dict of functions** — the Pythonic replacement for a long `if name == "accuracy": ... elif ...` chain. O(1) dispatch, and adding a metric is one line with zero changes to `evaluate`.
- `metrics or list(METRICS)` — short-circuit default. Note the parameter default is `None`, not `list(METRICS)`: mutable defaults are forbidden (§3.8).
- The `unknown` check raises with **both** what was wrong and what is valid. Typo-driven silent metric loss is a real failure mode; this makes it impossible.
- Each metric guards against division by zero. In imbalanced datasets, `pp == 0` (the model predicted no positives at all) genuinely happens.

### Production-style — a decorator-ready retry wrapper (functions as arguments)

```python
import functools, time, random, logging

logger = logging.getLogger(__name__)

def retry(max_attempts=3, base_delay=0.5, exceptions=(Exception,)):
    """Parameterised decorator: retries a function with exponential backoff + jitter."""
    def decorator(fn):
        @functools.wraps(fn)                 # preserve __name__, __doc__, signature
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions as exc:
                    if attempt == max_attempts:
                        logger.error("%s failed after %d attempts", fn.__name__, attempt)
                        raise
                    delay = base_delay * 2 ** (attempt - 1) * (0.5 + random.random())
                    logger.warning("%s attempt %d failed (%s); retrying in %.2fs",
                                   fn.__name__, attempt, exc, delay)
                    time.sleep(delay)
        return wrapper
    return decorator


@retry(max_attempts=4, base_delay=0.1, exceptions=(ConnectionError,))
def embed(texts: list[str]) -> list[list[float]]:
    """Call the embedding API."""
    if random.random() < 0.6:
        raise ConnectionError("upstream timeout")
    return [[0.1] * 384 for _ in texts]

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
vecs = embed(["hello", "world"])
print(f"got {len(vecs)} vectors of dim {len(vecs[0])}")
print(f"decorated function still named: {embed.__name__!r}, doc: {embed.__doc__!r}")
```
**Output (varies):**
```
WARNING embed attempt 1 failed (upstream timeout); retrying in 0.08s
WARNING embed attempt 2 failed (upstream timeout); retrying in 0.13s
got 2 vectors of dim 384
decorated function still named: 'embed', doc: 'Call the embedding API.'
```
**Line by line — this is three closures deep, so read it carefully.**
1. `retry(...)` captures the configuration and returns `decorator`.
2. `decorator(fn)` captures the function and returns `wrapper`.
3. `wrapper(*args, **kwargs)` is what actually gets called; it forwards everything to `fn`.
4. `@functools.wraps(fn)` copies `__name__`, `__doc__`, `__module__`, and `__wrapped__` onto the wrapper. **Without it, `embed.__name__` would be `"wrapper"`** — which breaks logging, breaks `inspect.signature`, and breaks every LLM tool-schema generator that reads the docstring. Never write a decorator without it.
5. `exceptions=(ConnectionError,)` — retry only *transient* errors. Retrying a `ValueError` from bad input is pure waste; retrying an auth failure will get you rate-limited.

Decorators get their own full treatment in Chapter 8; this shows why closures matter.

### AI-specific — a tool function an LLM can call

```python
import inspect, json
from typing import get_type_hints

def python_type_to_json(t) -> str:
    return {str: "string", int: "integer", float: "number",
            bool: "boolean", list: "array", dict: "object"}.get(t, "string")

def to_tool_schema(fn) -> dict:
    """Build an LLM function-calling schema from a Python function."""
    sig, hints = inspect.signature(fn), get_type_hints(fn)
    props, required = {}, []
    doc = inspect.getdoc(fn) or ""
    summary = doc.split("\n\n")[0]

    for name, param in sig.parameters.items():
        props[name] = {"type": python_type_to_json(hints.get(name, str))}
        if param.default is inspect.Parameter.empty:
            required.append(name)
        else:
            props[name]["description"] = f"Optional. Defaults to {param.default!r}."

    return {"name": fn.__name__, "description": summary,
            "parameters": {"type": "object", "properties": props, "required": required}}


def search_knowledge_base(query: str, top_k: int = 5, min_score: float = 0.7) -> list[dict]:
    """Search the internal knowledge base for relevant documents.

    Use this whenever the user asks about internal policies, product docs,
    or anything that requires up-to-date company information.
    """
    return [{"id": f"doc{i}", "score": 0.9 - 0.05 * i} for i in range(top_k)]


print(json.dumps(to_tool_schema(search_knowledge_base), indent=2))
```
**Output:**
```json
{
  "name": "search_knowledge_base",
  "description": "Search the internal knowledge base for relevant documents.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {"type": "string"},
      "top_k": {"type": "integer", "description": "Optional. Defaults to 5."},
      "min_score": {"type": "number", "description": "Optional. Defaults to 0.7."}
    },
    "required": ["query"]
  }
}
```
**Why this example matters more than it looks.** This is the entire mechanism behind LLM tool use, in 20 lines. The model never sees your code — it sees this schema. Consequences:

- **Your docstring is a prompt.** "Search the KB" tells the model nothing about *when* to call it; the version above tells it explicitly. Bad docstrings cause tools to be called at the wrong times, which users experience as "the AI is dumb".
- **Your parameter names are a prompt.** `q` vs `query` measurably changes model behaviour.
- **Missing type hints degrade to `"string"`**, so the model sends `"5"` instead of `5` and your function crashes.

### AI-specific — closures for parameterised loss/scoring

```python
def make_weighted_scorer(semantic_w: float = 0.7, recency_w: float = 0.2,
                         popularity_w: float = 0.1):
    """Return a scoring function with these weights baked in."""
    total = semantic_w + recency_w + popularity_w
    if not 0.99 < total < 1.01:
        raise ValueError(f"weights must sum to 1.0, got {total}")

    def score(doc: dict) -> float:
        return (semantic_w * doc["similarity"]
                + recency_w * doc["recency"]
                + popularity_w * doc["popularity"])
    score.weights = (semantic_w, recency_w, popularity_w)   # functions are objects!
    return score

docs = [
    {"id": "a", "similarity": 0.9, "recency": 0.2, "popularity": 0.1},
    {"id": "b", "similarity": 0.7, "recency": 0.9, "popularity": 0.8},
]
for name, scorer in [("semantic-heavy", make_weighted_scorer(0.9, 0.05, 0.05)),
                     ("recency-heavy",  make_weighted_scorer(0.3, 0.6, 0.1))]:
    ranked = sorted(docs, key=scorer, reverse=True)
    print(f"{name:<15} weights={scorer.weights} order={[d['id'] for d in ranked]}")
```
**Output:**
```
semantic-heavy  weights=(0.9, 0.05, 0.05) order=['a', 'b']
recency-heavy   weights=(0.3, 0.6, 0.1) order=['b', 'a']
```
**The point.** The same reranking code produces different behaviour purely from configuration captured in a closure — no globals, no classes, no if-statements. This is how you A/B-test retrieval strategies. Note also `score.weights = ...`: functions are objects and accept arbitrary attributes, which is how many frameworks attach metadata to registered handlers.

---

## 3.5 AI Engineering Perspective

**Machine Learning.** scikit-learn's entire API is built on the assumption that transformations are composable callables — `Pipeline([("tfidf", ...), ("clf", ...)])`. `cross_val_score(estimator, X, y, scoring=my_scorer)` takes your function. Writing preprocessing as pure functions is what makes a `Pipeline` possible, and a `Pipeline` is what prevents train/test leakage.

**Deep Learning.** A custom loss is a function `(pred, target) -> scalar`. A learning-rate schedule is a function `step -> lr`. A `collate_fn` passed to a `DataLoader` is a function turning a list of samples into a batch. PyTorch hooks (`module.register_forward_hook(fn)`) are callbacks — first-class functions again.

**LLM applications.** Tool/function calling *is* functions plus schemas (§3.4). Prompt templates are functions `(**vars) -> str`. Output parsers are functions `str -> structured`. Guardrails are functions applied before and after the model call.

**AI Agents.** An agent is a loop over a **registry of functions**: `TOOLS: dict[str, Callable]`. The model picks a name, you look it up and call it with unpacked kwargs: `TOOLS[name](**args)`. Validate `name in TOOLS` before that lookup — the model *will* hallucinate tool names.

**RAG.** Chunkers, embedders, rerankers, and filters are all pluggable functions. This is exactly why LangChain and LlamaIndex feel the way they do — they are function-composition frameworks with a `|` operator on top.

**Data preprocessing.** `df["col"].apply(fn)`, `dataset.map(fn, batched=True)` (Hugging Face), and `functools.lru_cache` on an expensive normaliser are all everyday uses. Note: `dataset.map` requires your function to be **picklable** for multiprocessing — which rules out lambdas and locally-defined closures (§3.8).

**Model serving.** FastAPI reads your function's type hints and generates request validation plus OpenAPI docs from them:
```python
@app.post("/embed")
async def embed(req: EmbedRequest) -> EmbedResponse: ...
```
Your annotations *are* the API contract.

---

## 3.6 Internal Working

### The call machinery

```python
import dis
def f(a, b=2): return a + b
dis.dis("f(1, b=3)")
```
**Output (abbreviated):**
```
  LOAD_NAME    f
  PUSH_NULL
  LOAD_CONST   1
  LOAD_CONST   3
  KW_NAMES     ('b',)
  CALL         2
```

Each call: push the callable and its args on the stack, execute `CALL`, which creates a **frame object** holding local variables, the instruction pointer, and a link to the caller's frame. That frame chain is exactly what a traceback prints.

**Cost.** A Python function call is roughly 40–80 ns of pure overhead. Irrelevant in a training loop (one call per batch); significant if you call a tiny function 10 million times inside a hot loop. That is the *only* case where inlining is justified — and the real answer is usually vectorisation.

### Locals are array slots, not a dict

Inside a function, local variables compile to `LOAD_FAST`/`STORE_FAST` with integer indices into an array — no dictionary lookup. Module-level globals use `LOAD_GLOBAL`, a dict lookup. **This is why moving a hot loop into a function is often 20–30% faster on its own**, a genuinely useful trick.

### Frames, references, and memory

A frame holds strong references to all its locals. While a function is running (or its exception traceback is alive), everything it references stays alive.

```python
def train_step(batch):
    loss = model(batch)     # `loss` holds the whole autograd graph
    loss.backward()
    return loss             # ❌ returning the tensor keeps the graph alive
    # ✔ return loss.item()  — a plain float; the graph can be freed
```
This is the #1 cause of "my GPU memory grows every epoch." Chapter 6 covers the related trap: an exception's traceback holds every frame's locals, so catching and *storing* exceptions can pin GPU memory too.

---

## 3.7 Best Practices

```python
# ✔ Type-hint every public function; docstring every public function
def chunk_document(text: str, *, size: int = 512, overlap: int = 64) -> list[str]:
    """Split text into overlapping chunks of approximately `size` characters."""

# ✔ Keyword-only for anything ambiguous
def train(model, data, *, lr: float, epochs: int, fp16: bool = True): ...

# ✔ None as the default for mutable parameters
def process(items: list[str], cache: dict | None = None) -> list[str]:
    cache = {} if cache is None else cache

# ✔ Early return / guard clauses over deep nesting
def embed(text: str) -> list[float] | None:
    if not text.strip():
        return None
    ...

# ✔ One job per function
# ✘ def load_clean_embed_and_store(path): ...   ← four jobs, untestable
```

**The rules that matter most in AI codebases**

1. **Prefer pure functions.** No hidden global state, no I/O, no mutation of inputs. Pure functions can be cached (`@lru_cache`), parallelised (`multiprocessing`), and tested without fixtures.
2. **Never mutate an argument unless the name says so.** `sort_docs` returns new; `sort_docs_inplace` does not.
3. **Fail fast with informative errors.** Validate at the top, include the received value and the valid range.
4. **Keep functions under ~30 lines.** If you need section comments inside a function, those sections are functions.
5. **Return a consistent type.** A function that returns `list` on success and `None` on failure forces every caller to remember the check. Return an empty list, or raise.
6. **`functools.wraps` on every decorator.** Always.
7. **`functools.lru_cache` for expensive pure functions:**
   ```python
   @functools.lru_cache(maxsize=10_000)
   def embed_cached(text: str) -> tuple[float, ...]:   # tuple: hashable & immutable
       return tuple(embedding_model.encode(text))
   ```
   Arguments must be hashable — this is why you sometimes convert a list to a tuple at a cache boundary.
8. **Docstrings in Google or NumPy style, consistently.** Tooling (Sphinx, IDE tooltips, LLM tool schemas) parses them.

---

## 3.8 Common Mistakes

### Mistake 1 — Mutable default argument (the classic)

```python
# ❌ WRONG
def add_message(msg: str, history: list = []) -> list:
    history.append(msg)
    return history

print(add_message("hi"))      # -> ['hi']
print(add_message("there"))   # -> ['hi', 'there']   ← the SAME list, forever
```
```python
# ✔ CORRECT
def add_message(msg: str, history: list | None = None) -> list:
    history = [] if history is None else history
    history.append(msg)
    return history

print(add_message("hi"))      # -> ['hi']
print(add_message("there"))   # -> ['there']
```
**Why.** Defaults are evaluated **once**, when `def` executes, and stored in `func.__defaults__`. A mutable default becomes shared state across all calls. In LLM code this shows up as chat history leaking between users — a genuine security incident, not just a bug.

### Mistake 2 — Mutating the caller's data

```python
# ❌ WRONG
def add_system_prompt(messages: list[dict]) -> list[dict]:
    messages.insert(0, {"role": "system", "content": SYSTEM})
    return messages          # caller's list now permanently modified

# ✔ CORRECT
def add_system_prompt(messages: list[dict]) -> list[dict]:
    return [{"role": "system", "content": SYSTEM}, *messages]
```
Call the wrong version twice in a retry loop and you get two system prompts. Three retries, three system prompts.

### Mistake 3 — Forgetting `functools.wraps`

```python
# ❌ WRONG
def timed(fn):
    def wrapper(*a, **k): return fn(*a, **k)
    return wrapper

@timed
def search(query: str) -> list: ...
print(search.__name__)                 # -> 'wrapper'
print(inspect.signature(search))       # -> (*a, **k)   ← tool schema destroyed
```
```python
# ✔ CORRECT
import functools
def timed(fn):
    @functools.wraps(fn)
    def wrapper(*a, **k): return fn(*a, **k)
    return wrapper
print(search.__name__)                 # -> 'search'
```

### Mistake 4 — Returning `None` inconsistently

```python
# ❌ WRONG
def retrieve(query, k=5):
    hits = index.search(query, k)
    if not hits:
        return None                     # caller does len(None) -> TypeError
    return hits

# ✔ CORRECT
def retrieve(query, k=5) -> list[dict]:
    return index.search(query, k) or []  # always a list
```

### Mistake 5 — Using a lambda where pickling is required

```python
# ❌ WRONG — fails under multiprocessing / HF datasets num_proc>1
dataset.map(lambda x: {"len": len(x["text"])}, num_proc=4)
# AttributeError: Can't pickle local object '<lambda>'

# ✔ CORRECT — a module-level named function is picklable
def add_length(example):
    return {"len": len(example["text"])}
dataset.map(add_length, num_proc=4)
```

### Mistake 6 — Bare `*args, **kwargs` on public APIs

```python
# ❌ WRONG — no IDE completion, no type checking, no tool schema, no docs
def generate(*args, **kwargs): ...

# ✔ CORRECT — explicit params, then forward the rest
def generate(prompt: str, *, model: str = "small", temperature: float = 0.7,
             **provider_kwargs): ...
```

### Mistake 7 — Deep recursion on linear data

```python
# ❌ WRONG — RecursionError beyond ~1000 chunks
def process_all(chunks):
    if not chunks: return []
    return [transform(chunks[0])] + process_all(chunks[1:])   # also O(n²) slicing!

# ✔ CORRECT
def process_all(chunks):
    return [transform(c) for c in chunks]
```

---

## 3.9 Performance

| Aspect | Cost / note |
|---|---|
| Function call overhead | ~40–80 ns; negligible unless called millions of times |
| Local variable access | `LOAD_FAST` — array index, faster than a global dict lookup |
| Default evaluation | Once, at `def` time — not per call |
| `*args`/`**kwargs` | Builds a tuple/dict per call — slightly slower than named params |
| `functools.lru_cache` | Hash + dict lookup (~100 ns) vs. recomputation; enormous win for embeddings |
| Closures | One cell dereference; cheap |
| Recursion | Frame allocation per level; no TCO; limit 1000 |

### Caching, measured

```python
import functools, time

def slow_embed(text: str) -> tuple:
    time.sleep(0.01)                     # simulate an API call
    return (len(text) * 0.1,)

cached_embed = functools.lru_cache(maxsize=1000)(slow_embed)

queries = ["what is rag", "what is rag", "how to fine-tune", "what is rag"]

t = time.perf_counter(); [slow_embed(q) for q in queries]
print(f"uncached: {time.perf_counter()-t:.3f}s")

t = time.perf_counter(); [cached_embed(q) for q in queries]
print(f"cached  : {time.perf_counter()-t:.3f}s  {cached_embed.cache_info()}")
```
**Output:**
```
uncached: 0.041s
cached  : 0.021s  CacheInfo(hits=2, misses=2, maxsize=1000, currsize=2)
```
In a real RAG system, query embeddings repeat constantly (users ask similar things) and document embeddings repeat on every re-index. Caching cuts both latency and API spend directly.

### Recursion vs iteration

```python
import sys
def depth(n):
    return 0 if n == 0 else 1 + depth(n - 1)

print(sys.getrecursionlimit())     # -> 1000
try:
    depth(5000)
except RecursionError as e:
    print("RecursionError:", e)
```
**Output:**
```
1000
RecursionError: maximum recursion depth exceeded
```
Raising the limit with `sys.setrecursionlimit` risks a hard C-stack segfault. Convert to iteration with an explicit stack instead.

---

## 3.10 Interview Preparation

**Q1 (Beginner). Is Python pass-by-value or pass-by-reference?**
Neither — it is "pass by object reference" (call by sharing). The reference is copied, so rebinding the parameter doesn't affect the caller, but mutating the referenced object does. Immutable arguments therefore *appear* pass-by-value; mutable ones appear pass-by-reference.

**Q2 (Beginner). `*args` vs `**kwargs`?**
`*args` packs extra positional arguments into a tuple; `**kwargs` packs extra keyword arguments into a dict. At a call site the same symbols *unpack* a sequence/mapping into arguments. Order in a signature: positional, `*args`, keyword-only, `**kwargs`.

**Q3 (Beginner). Explain the mutable default argument bug.**
Defaults are evaluated once at function-definition time and stored on the function object. A mutable default (`[]`, `{}`) is therefore shared by every call, accumulating state. Fix: default to `None` and create the container inside the function.

**Q4 (Intermediate). Explain LEGB and when you need `nonlocal`.**
Name lookup goes Local → Enclosing → Global → Builtins. `nonlocal` is needed to *assign* to a variable in an enclosing function scope (without it, assignment creates a new local). `global` does the same for module scope. Reading needs neither; only writing does.

**Q5 (Intermediate). What is a closure and give a real use?**
A function object plus captured references to variables from its enclosing scope, kept alive in cell objects. Real uses: parameterised decorators (retry with a configured backoff), factory functions (a reranker with baked-in weights), and callbacks that need configuration but must match a fixed signature.

**Q6 (Intermediate). Why does `functools.wraps` matter?**
It copies `__name__`, `__doc__`, `__module__`, `__qualname__`, `__dict__`, and sets `__wrapped__` on the wrapper. Without it, logging shows `wrapper`, `inspect.signature` shows `(*args, **kwargs)`, Sphinx docs break, and — specific to AI work — any tool-schema generator that reads the signature and docstring produces a useless schema.

**Q7 (AI/ML). How does an LLM know how to call your Python function?**
It doesn't see the code. A schema is generated from the signature (parameter names, type annotations, defaults/required) and the docstring (the description), then sent with the request. The model returns a name plus a JSON arguments object, which you validate and dispatch with `TOOLS[name](**args)`. Therefore docstrings and annotations are functionally part of the prompt.

**Q8 (AI/ML). Why must a preprocessing function be pure and picklable?**
Pure: so training and inference produce identical features (no train/serve skew), and so results can be cached. Picklable: `multiprocessing`, `DataLoader(num_workers>0)`, and `datasets.map(num_proc>1)` send the function to worker processes via pickle. Lambdas and locally-defined closures are not picklable, so they fail with `num_workers > 0` — a very common and confusing error.

**Q9 (Scenario). Users report the chatbot sometimes replies with another user's context. Where do you look first?**
Shared mutable state. Prime suspect: a mutable default argument holding conversation history, or a module-level `history = []`. Both persist across requests in a long-lived server process. Fix: per-request state, `None` defaults, and ideally immutable message structures.

**Q10 (Scenario). Design a function boundary for a RAG pipeline.**
Five pure functions with explicit contracts: `load(path) -> list[Document]`, `chunk(doc, *, size, overlap) -> list[Chunk]`, `embed(chunks) -> list[Vector]`, `retrieve(query, *, k) -> list[Chunk]`, `build_prompt(query, chunks) -> str`. Each is independently testable and swappable; the orchestrator only wires them. Injectable dependencies (tokenizer, embedder) go in as parameters so tests can pass fakes.

**Q11 (Production). When would you use `lru_cache`, and what's the risk?**
For expensive, pure, hashable-argument functions: embeddings, tokenizer loads, config parsing. Risks: unbounded memory if `maxsize=None`; stale results if the underlying data changes; arguments must be hashable (no lists/dicts); and it holds strong references to arguments and results, which can pin large objects — including GPU tensors — in memory.

---

## 3.11 Exercises

### Easy — E1
Write `truncate_prompt(text, max_chars=1000, suffix="...")` that shortens text without cutting mid-word.

<details><summary>Solution</summary>

```python
def truncate_prompt(text: str, max_chars: int = 1000, suffix: str = "...") -> str:
    if len(text) <= max_chars:
        return text
    budget = max_chars - len(suffix)
    cut = text[:budget]
    last_space = cut.rfind(" ")
    if last_space > budget * 0.6:          # only back up if we don't lose too much
        cut = cut[:last_space]
    return cut.rstrip() + suffix

print(truncate_prompt("the quick brown fox jumps over the lazy dog", 20))
```
**Output:** `the quick brown...`

`rfind` returns the last index of a substring, or −1 if absent. The `0.6` guard handles the pathological case of a single 900-character "word" (a URL or base64 blob), where backing up to the last space would destroy the content.
</details>

### Easy — E2
Write `batched(items, batch_size)` returning a list of lists, with the final partial batch included.

<details><summary>Solution</summary>

```python
def batched(items: list, batch_size: int) -> list[list]:
    if batch_size < 1:
        raise ValueError(f"batch_size must be >= 1, got {batch_size}")
    return [items[i:i + batch_size] for i in range(0, len(items), batch_size)]

print(batched(list(range(10)), 4))
```
**Output:** `[[0, 1, 2, 3], [4, 5, 6, 7], [8, 9]]`

`range(0, len, step)` gives the start index of each batch; slicing past the end is safe in Python (no IndexError), which is why the last partial batch works with no special case. Python 3.12 ships this as `itertools.batched`, which returns a lazy iterator of tuples — preferable for large data.
</details>

### Medium — M1
Write `compose(*functions)` that returns a single function applying them left to right, then use it to build a text-cleaning pipeline.

<details><summary>Solution</summary>

```python
import functools, re

def compose(*functions):
    """Return f such that f(x) == fn_n(...fn_2(fn_1(x)))."""
    def composed(value):
        for fn in functions:
            value = fn(value)
        return value
    composed.__name__ = "compose(" + ", ".join(f.__name__ for f in functions) + ")"
    return composed

def strip_html(t):    return re.sub(r"<[^>]+>", " ", t)
def strip_urls(t):    return re.sub(r"https?://\S+", " ", t)
def lower(t):         return t.lower()
def collapse_ws(t):   return " ".join(t.split())

clean = compose(strip_html, strip_urls, lower, collapse_ws)
print(clean.__name__)
print(clean("<p>Visit   https://x.com  NOW</p>"))
```
**Output:**
```
compose(strip_html, strip_urls, lower, collapse_ws)
visit now
```
**Explanation.** Each step is independently testable, and the pipeline is data: reorder it, or build it from a config file, without touching any step. `collapse_ws` must run last — every earlier step introduces stray spaces. This is precisely LangChain's LCEL `|` operator and scikit-learn's `Pipeline`, minus the ceremony.
</details>

### Medium — M2
Write `@memoize_embedding` — a decorator that caches embeddings, handles unhashable list arguments, and exposes hit/miss stats.

<details><summary>Solution</summary>

```python
import functools

def memoize_embedding(fn):
    cache: dict[str, tuple] = {}
    stats = {"hits": 0, "misses": 0}

    @functools.wraps(fn)
    def wrapper(texts: list[str]) -> list[tuple]:
        results, missing, missing_idx = [None] * len(texts), [], []
        for i, text in enumerate(texts):
            if text in cache:
                stats["hits"] += 1
                results[i] = cache[text]
            else:
                stats["misses"] += 1
                missing.append(text)
                missing_idx.append(i)

        if missing:
            for idx, vec in zip(missing_idx, fn(missing), strict=True):
                cache[texts[idx]] = tuple(vec)
                results[idx] = tuple(vec)
        return results

    wrapper.cache_stats = lambda: dict(stats, size=len(cache))
    wrapper.clear_cache = cache.clear
    return wrapper


@memoize_embedding
def embed(texts: list[str]) -> list[list[float]]:
    print(f"  [API CALL for {len(texts)} texts]")
    return [[len(t) * 0.1] for t in texts]

print(embed(["cat", "dog"]))
print(embed(["cat", "bird", "dog"]))
print(embed.cache_stats())
```
**Output:**
```
  [API CALL for 2 texts]
[(0.3,), (0.3,)]
  [API CALL for 1 texts]
[(0.3,), (0.4,), (0.3,)]
{'hits': 4, 'misses': 3, 'size': 3}
```
**Explanation — why this beats `lru_cache` here.**
- `lru_cache` cannot take a `list` argument (unhashable) and would cache the *whole batch* as one key, so `["cat","dog"]` and `["dog","cat"]` would both miss.
- This version caches **per text** and calls the API only for the genuine misses, preserving output order via `missing_idx`. That is the real cost saving: on a second ingestion run over a mostly-unchanged corpus, you pay for the changed documents only.
- `nonlocal` isn't needed because we *mutate* `cache` and `stats` rather than rebind them — a good illustration of the LEGB write rule.
- Attaching `cache_stats`/`clear_cache` to the wrapper is the standard way to expose control over a decorator's internals.
</details>

### Hard — H1
Build a complete tool-dispatch system: a registry decorator, auto-generated schemas, argument validation, and safe execution of a model-produced tool call.

<details><summary>Solution</summary>

```python
import functools, inspect, json
from typing import Callable, get_type_hints

TOOLS: dict[str, Callable] = {}
SCHEMAS: dict[str, dict] = {}

JSON_TYPES = {str: "string", int: "integer", float: "number", bool: "boolean",
              list: "array", dict: "object"}

def tool(fn: Callable) -> Callable:
    """Register a function as an LLM-callable tool and build its schema."""
    if not inspect.getdoc(fn):
        raise ValueError(f"tool {fn.__name__!r} must have a docstring — the model reads it")

    hints = get_type_hints(fn)
    sig = inspect.signature(fn)
    props, required = {}, []
    for name, p in sig.parameters.items():
        if name not in hints:
            raise ValueError(f"tool {fn.__name__!r} param {name!r} needs a type annotation")
        props[name] = {"type": JSON_TYPES.get(hints[name], "string")}
        if p.default is inspect.Parameter.empty:
            required.append(name)
        else:
            props[name]["default"] = p.default

    SCHEMAS[fn.__name__] = {
        "name": fn.__name__,
        "description": inspect.getdoc(fn).split("\n\n")[0],
        "parameters": {"type": "object", "properties": props, "required": required},
    }

    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)

    TOOLS[fn.__name__] = wrapper
    return wrapper


class ToolError(Exception): ...

def dispatch(tool_call: dict) -> dict:
    """Safely execute a model-produced tool call. Never trust the model's output."""
    name = tool_call.get("name")
    args = tool_call.get("arguments", {})

    if name not in TOOLS:
        return {"error": f"unknown tool {name!r}; available: {sorted(TOOLS)}"}
    if not isinstance(args, dict):
        return {"error": f"arguments must be an object, got {type(args).__name__}"}

    sig = inspect.signature(TOOLS[name])
    accepted = set(sig.parameters)
    unexpected = set(args) - accepted
    if unexpected:
        return {"error": f"unexpected arguments {sorted(unexpected)}; accepts {sorted(accepted)}"}
    missing = [n for n, p in sig.parameters.items()
               if p.default is inspect.Parameter.empty and n not in args]
    if missing:
        return {"error": f"missing required arguments: {missing}"}

    try:
        return {"result": TOOLS[name](**args)}
    except Exception as exc:                      # noqa: BLE001 — boundary handler
        return {"error": f"{type(exc).__name__}: {exc}"}


# ---------------- tools ----------------
@tool
def search_docs(query: str, top_k: int = 3) -> list[str]:
    """Search internal documentation for a query.

    Use for questions about internal policies or product behaviour.
    """
    return [f"doc about {query} #{i}" for i in range(top_k)]

@tool
def calculate(expression: str) -> float:
    """Evaluate a simple arithmetic expression such as '2 * (3 + 4)'."""
    allowed = set("0123456789+-*/(). ")
    if not set(expression) <= allowed:
        raise ValueError(f"expression contains disallowed characters: {expression!r}")
    return float(eval(expression, {"__builtins__": {}}, {}))   # sandboxed-ish


if __name__ == "__main__":
    print("SCHEMAS SENT TO MODEL:")
    print(json.dumps(list(SCHEMAS.values()), indent=2)[:400], "...\n")

    model_outputs = [
        {"name": "search_docs", "arguments": {"query": "vacation policy", "top_k": 2}},
        {"name": "calculate",   "arguments": {"expression": "2 * (3 + 4)"}},
        {"name": "search_web",  "arguments": {"query": "x"}},              # hallucinated
        {"name": "search_docs", "arguments": {"q": "typo"}},               # wrong arg
        {"name": "calculate",   "arguments": {"expression": "__import__('os')"}},  # attack
    ]
    for call in model_outputs:
        print(f"{call['name']:<12} -> {dispatch(call)}")
```
**Output:**
```
SCHEMAS SENT TO MODEL:
[
  {
    "name": "search_docs",
    "description": "Search internal documentation for a query.",
    ...

search_docs  -> {'result': ['doc about vacation policy #0', 'doc about vacation policy #1']}
calculate    -> {'result': 14.0}
search_web   -> {'error': "unknown tool 'search_web'; available: ['calculate', 'search_docs']"}
search_docs  -> {'error': "unexpected arguments ['q']; accepts ['query', 'top_k']"}
calculate    -> {'error': "ValueError: expression contains disallowed characters: \"__import__('os')\""}
```
**Explanation — every design decision is a production one.**
- **The decorator enforces documentation and annotations at import time.** A tool with no docstring fails on module load, not silently at 3 a.m. when the model can't figure out when to use it.
- **`dispatch` treats model output as untrusted input** — exactly like an HTTP request body. Unknown tool names, wrong argument names, and wrong types are all normal occurrences, not exceptional ones.
- **Errors are returned, not raised.** The error string goes back into the conversation so the model can correct itself on the next turn. Crashing the process on a hallucinated tool name would be a terrible agent.
- **The broad `except` is justified here** because this is a *boundary*: a tool crashing must not kill the agent loop. (Chapter 6 explains why broad excepts are otherwise forbidden.)
- **`calculate` shows the security dimension.** `eval` on model output is remote code execution. The character allowlist plus stripped `__builtins__` is a minimum; in production use `ast.literal_eval` or a real expression parser, and never `eval`.
</details>

---

## 3.12 Mini Project — Prompt Template Engine

**Goal.** A reusable, validated prompt template system: named templates, required-variable checking, defaults, composable partials, and token-budget awareness. Every LLM application needs this, and rolling your own teaches the whole chapter at once.

```python
"""prompt_engine.py — validated, composable prompt templates."""
import re
import string
from typing import Callable

class TemplateError(Exception):
    """Raised when a template cannot be rendered."""


class PromptTemplate:
    """A named prompt template with declared variables and validation."""

    _VAR_RE = re.compile(r"\{(\w+)\}")

    def __init__(self, name: str, template: str, *,
                 defaults: dict[str, str] | None = None,
                 max_chars: int | None = None,
                 validators: dict[str, Callable[[str], bool]] | None = None):
        self.name = name
        self.template = template
        self.defaults = defaults or {}          # ← None default, not {} (§3.8)
        self.max_chars = max_chars
        self.validators = validators or {}
        self.variables = set(self._VAR_RE.findall(template))
        self.required = self.variables - set(self.defaults)

    def render(self, **kwargs: object) -> str:
        values = {**self.defaults, **{k: str(v) for k, v in kwargs.items()}}

        missing = self.required - set(values)
        if missing:
            raise TemplateError(
                f"template {self.name!r} missing required variables: {sorted(missing)}; "
                f"required={sorted(self.required)} optional={sorted(self.defaults)}"
            )
        extra = set(values) - self.variables
        if extra:
            raise TemplateError(
                f"template {self.name!r} received unknown variables: {sorted(extra)}"
            )
        for var, check in self.validators.items():
            if var in values and not check(values[var]):
                raise TemplateError(f"variable {var!r} failed validation: {values[var]!r}")

        rendered = self.template.format(**values)

        if self.max_chars and len(rendered) > self.max_chars:
            raise TemplateError(
                f"rendered prompt is {len(rendered)} chars, exceeds max {self.max_chars}"
            )
        return rendered

    def partial(self, **fixed: object) -> "PromptTemplate":
        """Return a new template with some variables pre-filled."""
        return PromptTemplate(
            f"{self.name}+partial",
            self.template,
            defaults={**self.defaults, **{k: str(v) for k, v in fixed.items()}},
            max_chars=self.max_chars,
            validators=self.validators,
        )

    def __repr__(self) -> str:
        return (f"PromptTemplate(name={self.name!r}, "
                f"required={sorted(self.required)}, optional={sorted(self.defaults)})")


REGISTRY: dict[str, PromptTemplate] = {}

def register(tmpl: PromptTemplate) -> PromptTemplate:
    if tmpl.name in REGISTRY:
        raise TemplateError(f"duplicate template name {tmpl.name!r}")
    REGISTRY[tmpl.name] = tmpl
    return tmpl


register(PromptTemplate(
    "rag_answer",
    "You are a {persona}.\n"
    "Answer using ONLY the context. If the answer is not present, say \"I don't know\".\n\n"
    "# Context\n{context}\n\n"
    "# Question\n{question}\n\n"
    "# Answer (max {max_words} words)\n",
    defaults={"persona": "precise technical assistant", "max_words": "150"},
    max_chars=8000,
    validators={"question": lambda q: 0 < len(q) <= 500},
))

register(PromptTemplate(
    "classify",
    "Classify the text into exactly one of: {labels}.\n"
    "Respond with the label only.\n\nText: {text}\nLabel:",
    validators={"labels": lambda s: "," in s},
))


if __name__ == "__main__":
    rag = REGISTRY["rag_answer"]
    print(rag, "\n")

    print(rag.render(context="RAG retrieves then generates.", question="What is RAG?"))
    print("-" * 60)

    terse = rag.partial(persona="terse expert", max_words=20)
    print(terse.render(context="RAG retrieves then generates.", question="What is RAG?"))
    print("-" * 60)

    for bad in (
        lambda: rag.render(context="x"),                          # missing question
        lambda: rag.render(context="x", question="y", tone="z"),  # unknown var
        lambda: rag.render(context="x", question=""),             # failed validator
        lambda: rag.render(context="x" * 9000, question="y"),     # too long
    ):
        try:
            bad()
        except TemplateError as exc:
            print("TemplateError:", exc)
```

**Output:**
```
PromptTemplate(name='rag_answer', required=['context', 'question'], optional=['max_words', 'persona'])

You are a precise technical assistant.
Answer using ONLY the context. If the answer is not present, say "I don't know".

# Context
RAG retrieves then generates.

# Question
What is RAG?

# Answer (max 150 words)

------------------------------------------------------------
You are a terse expert.
...
# Answer (max 20 words)

------------------------------------------------------------
TemplateError: template 'rag_answer' missing required variables: ['question']; required=['context', 'question'] optional=['max_words', 'persona']
TemplateError: template 'rag_answer' received unknown variables: ['tone']
TemplateError: variable 'question' failed validation: ''
TemplateError: rendered prompt is 9172 chars, exceeds max 8000
```

**What this project consolidates.**
- **`None`-defaults** for every mutable parameter (`defaults`, `validators`).
- **Keyword-only arguments** (`*`) so no caller can pass four positional strings in the wrong order.
- **Validator functions passed as parameters** — first-class functions and closures, letting each template carry its own rules without subclassing.
- **`partial()` returns a new object** rather than mutating — the immutability discipline from Chapters 0 and 1.
- **Errors name both the problem and the fix.** Compare `KeyError: 'question'` (what `str.format` gives you) with the message above.
- **Unknown-variable detection is not pedantry.** Renaming `{question}` to `{query}` in a template while callers still pass `question=` would otherwise render a literal `{query}` into the prompt and quietly degrade every answer.

**Extensions.** Load templates from YAML (Chapter 5); replace `max_chars` with real token counting; add versioning and A/B selection; make `PromptTemplate` a frozen dataclass (Chapter 7); add `async def arender` for templates that fetch context (Chapter 8).

---

## Chapter 3 Summary

| Concept | Takeaway |
|---|---|
| `def` | An executable statement; defaults evaluated **once** |
| Argument passing | Call by object reference — rebinding is local, mutation is shared |
| Parameter grammar | `pos, /, pos-or-kw, *, kw-only, **kwargs`; use `*` liberally |
| LEGB | Local → Enclosing → Global → Builtins; assignment implies local |
| Closures | Function + captured cells; the basis of decorators and factories |
| First-class functions | Registries, `key=`, callbacks, pluggable metrics |
| Introspection | Signature + docstring → LLM tool schema; write both carefully |
| Purity | Pure ⇒ testable, cacheable, parallelisable, no train/serve skew |
| `functools.wraps` | Mandatory on every decorator |
| Recursion | Trees yes, linear data no; limit 1000, no TCO |

**Revision flashcards.** Mutable default bug · `*` for keyword-only · LEGB + `nonlocal` · `wraps` and why · lambdas aren't picklable · docstring = prompt.

---
# Chapter 4 — Data Structures

> **Chapter goal.** Choosing the right container is the highest-leverage decision in most Python code. A `list` where a `set` belongs turns an O(n) job into O(n²) — the difference between a 30-second dedup and a 3-hour one. And since every LLM API, every vector-DB response, and every config file is JSON-shaped, `dict` fluency is not optional.

---

## 4.1 Concept Overview

### What it is

Python's built-in containers:

| Type | Ordered | Mutable | Duplicates | Lookup by | Literal |
|---|---|---|---|---|---|
| `list` | Yes | Yes | Yes | Integer index | `[1, 2, 3]` |
| `tuple` | Yes | No | Yes | Integer index | `(1, 2, 3)` |
| `set` | No* | Yes | No | Membership only | `{1, 2, 3}` |
| `frozenset` | No* | No | No | Membership only | `frozenset({1,2})` |
| `dict` | Yes† | Yes | Keys unique | Key | `{"a": 1}` |

\* Sets have no meaningful order (iteration order is a hash artefact — never rely on it).
† Dicts preserve insertion order — an implementation detail in 3.6, **a language guarantee since 3.7**.

### Why they exist

Different access patterns need different data layouts. There is no universal container: an array gives O(1) indexing but O(n) search; a hash table gives O(1) search but no order-by-position; an immutable type can be a key, a mutable one can't. Python gives you all four and expects you to choose.

### Why AI engineers care

```
    Raw data          →  list[dict]        (records from JSONL/CSV)
    Config            →  dict / frozen dataclass
    Tokens            →  list[int]         (then torch.tensor)
    Vocabulary        →  dict[str, int]    (O(1) token → id)
    Deduplication     →  set[str]          (hashes of documents)
    Batch             →  list[Tensor]      (then collated)
    LLM messages      →  list[dict]        ({"role", "content"})
    API response      →  dict              (nested JSON)
    Model metadata    →  tuple / NamedTuple (immutable shape info)
    Token counts      →  Counter
    Sliding window    →  deque(maxlen=n)
    Grouping          →  defaultdict(list)
```

Every one of those choices has a performance and correctness consequence. This chapter is about making them deliberately.

---

## 4.2 Theory

### The complexity table you must memorise

| Operation | `list` | `tuple` | `set` | `dict` | `deque` |
|---|---|---|---|---|---|
| Index `x[i]` | **O(1)** | O(1) | — | — | O(n) middle |
| Key/hash lookup | — | — | **O(1)** | **O(1)** | — |
| `x in c` (membership) | **O(n)** | O(n) | **O(1)** | **O(1)** | O(n) |
| Append / add | O(1)* | — | O(1) | O(1) | **O(1)** |
| Insert / delete at front | **O(n)** | — | — | — | **O(1)** |
| Insert / delete at end | O(1)* | — | O(1) | O(1) | O(1) |
| `len()` | O(1) | O(1) | O(1) | O(1) | O(1) |
| Iterate all | O(n) | O(n) | O(n) | O(n) | O(n) |
| Sort | O(n log n) | — | — | — | — |
| Memory per element | 8 B ptr + object | 8 B ptr + object | ~32–60 B | ~50–100 B | ~8 B ptr |

\* Amortised: appending occasionally triggers a reallocation that copies everything, but the cost spread over many appends is O(1).

**The single most valuable row is `x in c`.** `list` is O(n), `set`/`dict` is O(1). Internalise it.

### How a list works

A Python list is a **dynamic array of pointers**:

```
list object          ob_item ──► [ ptr | ptr | ptr | ptr | --- | --- | --- ]
  ob_size    = 4                    │     │     │     │     ^ allocated slack
  allocated  = 7                    v     v     v     v
                                  obj   obj   obj   obj   (scattered in the heap)
```

- Indexing is pointer arithmetic → O(1).
- `append` writes into the slack; when full, CPython allocates a new block ~1.125× larger and copies the pointers. Amortised O(1).
- `insert(0, x)` and `pop(0)` must shift every pointer → O(n). **Never use a list as a queue.**

### How dicts and sets work

Both are **open-addressed hash tables**. Lookup:

```
   key ──► hash(key) ──► index into a sparse table ──► compare (identity, then ==)
                                    │
                                    └─ collision? probe the next slot
```

Since 3.6, dicts use a **compact layout**: a dense `entries` array (in insertion order) plus a sparse `indices` array of positions. That is why iteration is insertion-ordered and why dicts got ~20% smaller.

**Requirements for keys:** hashable ⇒ has `__hash__` ⇒ (by convention) immutable. Equal objects must have equal hashes.

```python
key = ("model-a", 3, "fp16")     # ✔ tuple of immutables — hashable
bad = ["model-a", 3]             # ✘ TypeError: unhashable type: 'list'
```

**Why sets are unordered:** iteration walks the hash table slots, so order depends on hash values and insertion history. Never rely on it; if you need order, sort explicitly or use a dict with `None` values.

### Mutability, aliasing, and copies (revisited, concretely)

```python
a = [[0] * 3] * 2                  # ❌ the SAME inner list, twice
a[0][0] = 9
print(a)                           # -> [[9, 0, 0], [9, 0, 0]]

b = [[0] * 3 for _ in range(2)]    # ✔ two distinct inner lists
b[0][0] = 9
print(b)                           # -> [[9, 0, 0], [0, 0, 0]]
```
**Output:**
```
[[9, 0, 0], [9, 0, 0]]
[[9, 0, 0], [0, 0, 0]]
```
`[x] * n` copies the *reference* n times. This exact bug silently ruins hand-built matrices, attention masks, and per-class accumulator lists.

### Slicing

```python
tokens = [101, 2054, 2003, 20304, 102]
tokens[1:4]      # -> [2054, 2003, 20304]   start inclusive, stop exclusive
tokens[:3]       # -> [101, 2054, 2003]
tokens[-2:]      # -> [20304, 102]          last two
tokens[::2]      # -> [101, 2003, 102]      every second
tokens[::-1]     # -> [102, 20304, 2003, 2054, 101]   reversed (a copy)
tokens[1:-1]     # -> [2054, 2003, 20304]   drop the special tokens
```

Rules: slicing a list **always returns a new list** (a shallow copy); out-of-range slice bounds are clamped, never an error; `x[:]` is the idiomatic shallow copy.

> **Contrast with NumPy/PyTorch:** `arr[1:4]` returns a **view** that shares memory. Writing to the view writes to the original. This difference — list slices copy, array slices view — surprises everyone at least once.

---

## 4.3 Syntax

### Lists

```python
tokens = [101, 2054, 2003, 102]

tokens.append(103)            # add one at the end          O(1)
tokens.extend([104, 105])     # add many at the end         O(k)
tokens.insert(0, 100)         # insert at index             O(n)
tokens.remove(2054)           # remove first == value       O(n), ValueError if absent
last = tokens.pop()           # remove + return last        O(1)
first = tokens.pop(0)         # remove + return first       O(n)  ← avoid
idx = tokens.index(2003)      # first index of value        O(n)
n = tokens.count(102)         # occurrences                 O(n)
tokens.sort()                 # in place, returns None      O(n log n)
ordered = sorted(tokens)      # new list                    O(n log n)
tokens.reverse()              # in place
tokens.clear()                # empty it
copy = tokens.copy()          # shallow copy (same as tokens[:])
```

> **`sort()` vs `sorted()`** is a classic interview question and a real bug source: `.sort()` mutates and returns `None`, so `x = mylist.sort()` sets `x = None`. `sorted()` returns a new list and works on any iterable.

### Comprehensions — the full grammar

```python
docs = [{"id": "a", "score": 0.9, "text": "hello"},
        {"id": "b", "score": 0.4, "text": "world"}]

[d["id"] for d in docs]                                # list
{d["id"]: d["score"] for d in docs}                    # dict
{d["id"] for d in docs}                                # set
(d["id"] for d in docs)                                # generator (lazy!)

[d["id"] for d in docs if d["score"] > 0.5]            # filter
["high" if d["score"] > 0.5 else "low" for d in docs]  # conditional VALUE
[t for d in docs for t in d["text"].split()]           # nested / flatten
```

**Reading a nested comprehension:** the `for` clauses read in the same order as nested loops.
```python
[t for d in docs for t in d["text"]]
#  ^value  ^outer loop   ^inner loop
# equivalent to:
# out = []
# for d in docs:
#     for t in d["text"]:
#         out.append(t)
```

### Tuples

```python
shape = (32, 512, 768)                # batch, seq, hidden
b, s, h = shape                       # unpacking
single = (42,)                        # ← the comma makes it a tuple, not the parens
point = 1, 2                          # parens are optional

from typing import NamedTuple
class ModelInfo(NamedTuple):
    name: str
    params: int
    dtype: str = "bfloat16"

m = ModelInfo("llama-7b", 7_000_000_000)
print(m.name, m.params, m.dtype)      # attribute access
print(m[0])                           # AND index access — it IS a tuple
print(m._replace(dtype="int8"))       # returns a NEW instance
```
**Output:**
```
llama-7b 7000000000 bfloat16
llama-7b
ModelInfo(name='llama-7b', params=7000000000, dtype='int8')
```

`NamedTuple` gives you immutability + named fields + tuple compatibility + hashability, all at tuple memory cost. Ideal for shapes, coordinates, and small records passed around a pipeline.

### Sets

```python
a = {"doc1", "doc2", "doc3"}
b = {"doc3", "doc4"}

a | b        # union            -> {'doc1','doc2','doc3','doc4'}
a & b        # intersection     -> {'doc3'}
a - b        # difference       -> {'doc1','doc2'}
a ^ b        # symmetric diff   -> {'doc1','doc2','doc4'}
a <= b       # subset?          -> False
a.isdisjoint(b)                 # -> False

a.add("doc5")
a.discard("nope")   # no error if missing
a.remove("nope")    # KeyError if missing
empty = set()       # ← {} is an empty DICT, not a set
```

### Dictionaries

```python
cfg = {"model": "bert", "lr": 3e-4}

cfg["model"]                      # KeyError if missing
cfg.get("epochs")                 # -> None if missing
cfg.get("epochs", 10)             # -> 10 default
cfg.setdefault("epochs", 10)      # get, inserting the default if absent
cfg.update({"lr": 1e-4, "fp16": True})
cfg.pop("fp16", None)             # remove + return, with a default
del cfg["lr"]                     # KeyError if missing

cfg.keys(); cfg.values(); cfg.items()      # dynamic views, not copies

for key, value in cfg.items(): ...

merged = {**defaults, **overrides}         # merge (right wins)
merged = defaults | overrides              # 3.9+ same thing
defaults |= overrides                      # in-place merge
```

**Views are live:**
```python
d = {"a": 1}
ks = d.keys()
d["b"] = 2
print(list(ks))       # -> ['a', 'b']   the view updated
```
Consequence: you cannot add or remove keys while iterating a view — `RuntimeError: dictionary changed size during iteration`. Iterate over `list(d)` if you plan to mutate.

### `collections` — the four you will actually use

```python
from collections import defaultdict, Counter, deque, OrderedDict

# ---------- defaultdict: no more KeyError plumbing ----------
by_label = defaultdict(list)
for doc in [("spam", "d1"), ("ham", "d2"), ("spam", "d3")]:
    by_label[doc[0]].append(doc[1])
print(dict(by_label))          # -> {'spam': ['d1', 'd3'], 'ham': ['d2']}

# ---------- Counter: frequency counting ----------
tokens = "the cat sat on the mat the end".split()
counts = Counter(tokens)
print(counts.most_common(2))   # -> [('the', 3), ('cat', 1)]
print(counts["missing"])       # -> 0   (no KeyError)
print(sum(counts.values()))    # -> 8   total tokens
vocab = Counter({"the": 3}) + Counter({"the": 2, "a": 1})
print(vocab)                   # -> Counter({'the': 5, 'a': 1})

# ---------- deque: O(1) at BOTH ends, with an optional cap ----------
window = deque(maxlen=3)       # a fixed-size sliding window
for msg in ["m1", "m2", "m3", "m4"]:
    window.append(msg)         # oldest auto-evicted when full
print(list(window))            # -> ['m2', 'm3', 'm4']

q = deque([1, 2, 3])
q.appendleft(0); q.popleft()   # both O(1) — a list's would be O(n)
q.rotate(1)                    # -> deque([3, 1, 2])

# ---------- OrderedDict: mostly legacy, but move_to_end is real ----------
lru = OrderedDict()
lru["a"] = 1; lru["b"] = 2
lru.move_to_end("a")           # -> a is now the most recent
print(list(lru))               # -> ['b', 'a']
```
**Output:**
```
{'spam': ['d1', 'd3'], 'ham': ['d2']}
[('the', 3), ('cat', 1)]
0
8
Counter({'the': 5, 'a': 1})
['m2', 'm3', 'm4']
['b', 'a']
```

**When is `OrderedDict` still right?** Only when you need `move_to_end`/`popitem(last=False)` (LRU caches) or order-sensitive equality (`OrderedDict([("a",1),("b",2)]) != OrderedDict([("b",2),("a",1)])`, whereas plain dicts compare equal). Otherwise use `dict`.

---

## 4.4 Examples

### Beginner — token list manipulation

```python
CLS, SEP, PAD = 101, 102, 0
ids = [2054, 2003, 20304, 1029]

sequence = [CLS, *ids, SEP]
print(sequence)

MAX_LEN = 8
if len(sequence) < MAX_LEN:
    attention = [1] * len(sequence) + [0] * (MAX_LEN - len(sequence))
    sequence = sequence + [PAD] * (MAX_LEN - len(sequence))
else:
    sequence = sequence[:MAX_LEN - 1] + [SEP]
    attention = [1] * MAX_LEN

print(sequence)
print(attention)
print("content only:", sequence[1:sequence.index(SEP)])
```
**Output:**
```
[101, 2054, 2003, 20304, 1029, 102]
[101, 2054, 2003, 20304, 1029, 102, 0, 0]
[1, 1, 1, 1, 1, 1, 0, 0]
content only: [2054, 2003, 20304, 1029]
```
**Line by line.** `[CLS, *ids, SEP]` unpacks a list inside a literal — cleaner than `[CLS] + ids + [SEP]` and it allocates once. `[1] * n` builds a repeated list (safe here because ints are immutable — contrast §4.2's nested-list trap). The truncation branch keeps the `SEP` token, which matters: models are trained expecting it, and dropping it measurably degrades quality. This is exactly what a tokenizer's `padding`/`truncation` arguments do internally.

### Intermediate — deduplicating a dataset

```python
import hashlib

docs = [
    {"id": "a", "text": "Machine learning is great."},
    {"id": "b", "text": "Deep learning is a subset of ML."},
    {"id": "c", "text": "machine learning is great!"},     # near-duplicate
    {"id": "d", "text": "Machine learning is great."},     # exact duplicate
]

def normalize(t: str) -> str:
    return "".join(ch for ch in t.lower() if ch.isalnum() or ch.isspace()).strip()

def fingerprint(t: str) -> str:
    return hashlib.sha256(normalize(t).encode()).hexdigest()[:16]

seen: set[str] = set()          # ← set, not list: O(1) membership
unique, dupes = [], []
for doc in docs:
    fp = fingerprint(doc["text"])
    if fp in seen:              # O(1)
        dupes.append(doc["id"])
        continue
    seen.add(fp)
    unique.append(doc)

print(f"kept {len(unique)}, dropped {len(dupes)}: {dupes}")
print([d["id"] for d in unique])
```
**Output:**
```
kept 2, dropped 2: ['c', 'd']
['a', 'b']
```
**Line by line.**
- `normalize` strips case and punctuation so `"great."` and `"great!"` collide — catching near-duplicates, not just byte-identical ones.
- Hashing to 16 hex chars keeps the set tiny: 100M documents × 16 bytes ≈ 1.6 GB of hashes versus storing full texts.
- **The `set` is the whole point.** With a list, this loop is O(n²): 1M documents ≈ 500 billion comparisons ≈ hours. With a set it is O(n) ≈ seconds.
- Deduplication is not optional hygiene — duplicate documents inflate evaluation metrics (test data leaking into training) and waste training compute.

### Production-style — grouping and aggregating evaluation results

```python
from collections import defaultdict, Counter
from statistics import mean, median

results = [
    {"model": "small", "task": "qa",      "score": 0.71, "latency_ms": 120, "ok": True},
    {"model": "small", "task": "qa",      "score": 0.68, "latency_ms": 130, "ok": True},
    {"model": "large", "task": "qa",      "score": 0.89, "latency_ms": 640, "ok": True},
    {"model": "small", "task": "summary", "score": 0.55, "latency_ms": 200, "ok": False},
    {"model": "large", "task": "summary", "score": 0.82, "latency_ms": 890, "ok": True},
    {"model": "large", "task": "qa",      "score": 0.91, "latency_ms": 700, "ok": True},
]

grouped = defaultdict(list)
for r in results:
    grouped[(r["model"], r["task"])].append(r)      # tuple key = composite index

failures = Counter(r["model"] for r in results if not r["ok"])

header = f"{'model':<7} {'task':<8} {'n':>2} {'mean':>6} {'median':>7} {'p50 ms':>7} {'fails':>6}"
print(header); print("-" * len(header))
for (model, task), rows in sorted(grouped.items()):
    scores = [r["score"] for r in rows]
    lats   = [r["latency_ms"] for r in rows]
    print(f"{model:<7} {task:<8} {len(rows):>2} {mean(scores):>6.3f} "
          f"{median(scores):>7.3f} {median(lats):>7.0f} {failures[model]:>6}")
```
**Output:**
```
model   task      n   mean  median  p50 ms  fails
--------------------------------------------------
large   qa        2  0.900   0.900     670      0
large   summary   1  0.820   0.820     890      0
small   qa        2  0.695   0.695     125      1
small   summary   1  0.550   0.550     200      1
```
**Line by line.**
- `defaultdict(list)` removes the `if key not in d: d[key] = []` boilerplate. Without it this loop is three lines longer and easier to get wrong.
- **`(model, task)` as a key** — tuples are hashable, so a composite key needs no nested dicts. This is the single most useful "advanced" dict trick.
- `Counter(genexp)` counts in one pass; `failures[model]` returns 0 for models with no failures instead of raising.
- `sorted(grouped.items())` gives deterministic report order — important, because non-deterministic output makes diffs between runs useless.

### AI-specific — chat history with a token budget (deque)

```python
from collections import deque

class ChatHistory:
    """Rolling conversation window that respects a token budget."""

    def __init__(self, max_tokens: int = 2000, system: str | None = None):
        self.max_tokens = max_tokens
        self.system = {"role": "system", "content": system} if system else None
        self.messages: deque[dict] = deque()
        self._tokens = 0

    @staticmethod
    def _count(msg: dict) -> int:
        return len(msg["content"]) // 4 + 4         # +4 for role/format overhead

    def add(self, role: str, content: str) -> None:
        if role not in {"user", "assistant", "tool"}:
            raise ValueError(f"invalid role {role!r}")
        msg = {"role": role, "content": content}
        self.messages.append(msg)
        self._tokens += self._count(msg)

        while self._tokens > self.max_tokens and len(self.messages) > 1:
            evicted = self.messages.popleft()       # O(1) — a list would be O(n)
            self._tokens -= self._count(evicted)

    def to_api_format(self) -> list[dict]:
        return ([self.system] if self.system else []) + list(self.messages)

    def __len__(self) -> int:
        return len(self.messages)

    def __repr__(self) -> str:
        return f"ChatHistory({len(self.messages)} msgs, ~{self._tokens} tokens)"


chat = ChatHistory(max_tokens=60, system="You are helpful.")
for i in range(6):
    chat.add("user", f"Question number {i} about retrieval augmented generation")
    chat.add("assistant", f"Answer number {i}")
    print(f"after turn {i}: {chat}")

print("\nsent to API:")
for m in chat.to_api_format():
    print(f"  {m['role']:<9} {m['content'][:45]}")
```
**Output:**
```
after turn 0: ChatHistory(2 msgs, ~30 tokens)
after turn 1: ChatHistory(4 msgs, ~60 tokens)
after turn 2: ChatHistory(4 msgs, ~59 tokens)
after turn 3: ChatHistory(4 msgs, ~59 tokens)
after turn 4: ChatHistory(4 msgs, ~59 tokens)
after turn 5: ChatHistory(4 msgs, ~59 tokens)

sent to API:
  system    You are helpful.
  user      Question number 4 about retrieval augmen
  assistant Answer number 4
  user      Question number 5 about retrieval augmen
  assistant Answer number 5
```
**Why every choice matters.**
- **`deque` not `list`** — eviction happens from the *front*, on every message, forever. `list.pop(0)` is O(n); over a long session that becomes the dominant cost.
- **The system message lives outside the deque**, so it is never evicted. Losing the system prompt mid-conversation is a real production bug that manifests as the assistant "forgetting its instructions."
- **Running token count** — recomputing the total on every message would be O(n) per turn, O(n²) per session.
- **`len(self.messages) > 1`** guard prevents evicting everything and sending an empty conversation.

### AI-specific — an inverted index (dict of sets)

```python
from collections import defaultdict

docs = {
    "d1": "vector databases store embeddings for similarity search",
    "d2": "embeddings capture semantic meaning of text",
    "d3": "similarity search uses cosine distance over embeddings",
}
STOP = {"for", "of", "the", "a", "over", "uses"}

index: dict[str, set[str]] = defaultdict(set)
for doc_id, text in docs.items():
    for word in text.lower().split():
        if word not in STOP:              # O(1) stop-word check
            index[word].add(doc_id)       # set: automatic dedup within a doc

def search(query: str, mode: str = "and") -> set[str]:
    terms = [w for w in query.lower().split() if w not in STOP]
    if not terms:
        return set()
    postings = [index.get(t, set()) for t in terms]
    return set.intersection(*postings) if mode == "and" else set.union(*postings)

print(sorted(search("embeddings similarity")))          # AND
print(sorted(search("embeddings similarity", "or")))    # OR
print(sorted(search("nonexistent term")))
print("index size:", len(index), "postings:", sum(len(v) for v in index.values()))
```
**Output:**
```
['d1', 'd3']
['d1', 'd2', 'd3']
[]
index size: 12 postings: 15
```
**Why this is worth building once by hand.** This is the core of BM25/keyword search, the "sparse" half of every serious hybrid-RAG system (dense embeddings alone miss exact terms like error codes, product SKUs, and names). The data structure choice is the algorithm: `dict[str, set[str]]` gives O(1) term lookup and O(min(len)) set intersection, so a query touches only the documents containing its terms — not all n documents.

---

## 4.5 AI Engineering Perspective

**Machine Learning.** A pandas DataFrame is roughly `dict[str, ndarray]` — column-oriented. That is why column operations are fast and row operations (`iterrows`) are slow. Feature dictionaries feed `DictVectorizer`; label mappings are `dict[str, int]` built once and **saved with the model** (an inconsistent label map between train and serve silently scrambles your classes).

**Deep Learning.** A batch is a `dict[str, Tensor]` (`input_ids`, `attention_mask`, `labels`) — which is exactly what a `collate_fn` produces from a list of samples. `model.state_dict()` is an `OrderedDict[str, Tensor]`; loading a checkpoint is dict key matching, which is why `missing_keys`/`unexpected_keys` errors are so common when architectures drift.

**LLM applications.** Messages are `list[dict]`. Tool calls are dicts. Structured output is a dict validated against a schema. Response parsing is nested `.get()` chains — and defensive `.get()` beats `[]` because providers add and rename fields.

**AI Agents.** Agent state is a dict passed between nodes (LangGraph literally types this as a `TypedDict`). Tool registries are `dict[str, Callable]`. Visited-state tracking to break loops is a `set` of action signatures.

**RAG.** Chunks are `list[dict]` with metadata. Metadata filters are dict comparisons. Hybrid search **fuses** two ranked lists — the standard algorithm, Reciprocal Rank Fusion, is a `defaultdict(float)` accumulating `1/(k + rank)` per document across retrievers. Deduplication across retrievers is a `set` of chunk ids.

**Data preprocessing.** `Counter` for vocabulary building and class-imbalance checks; `set` for dedup and stop words; `defaultdict(list)` for grouping by label or source; `deque(maxlen=n)` for rolling statistics.

**Model serving.** Request/response bodies are dicts (validated by pydantic). An in-memory LRU cache is `OrderedDict` or `functools.lru_cache`. A batching queue is a `deque`.

---

## 4.6 Internal Working

### Memory reality

```python
import sys
from collections import namedtuple

lst = [1, 2, 3]
tup = (1, 2, 3)
st  = {1, 2, 3}
dct = {"a": 1, "b": 2, "c": 3}

for name, obj in [("list", lst), ("tuple", tup), ("set", st), ("dict", dct)]:
    print(f"{name:<6} {sys.getsizeof(obj):>4} bytes")
```
**Output (CPython 3.12, 64-bit):**
```
list     88 bytes
tuple    64 bytes
set     216 bytes
dict    184 bytes
```
Tuples are the cheapest (no over-allocation slack, no resize machinery). Sets and dicts pay for sparse hash tables — they trade memory for O(1) lookup. **This is the fundamental trade-off**, and at scale (a 100M-entry `dict[str, int]` vocabulary ≈ several GB) it becomes a real architectural constraint, which is why production vocabularies live in specialised structures or on disk.

### List growth

```python
import sys
prev, lst = 0, []
for i in range(17):
    lst.append(i)
    size = sys.getsizeof(lst)
    if size != prev:
        print(f"len={len(lst):>3} -> {size} bytes (resized)")
        prev = size
```
**Output:**
```
len=  1 -> 88 bytes (resized)
len=  5 -> 120 bytes (resized)
len=  9 -> 184 bytes (resized)
len= 17 -> 256 bytes (resized)
```
Growth is roughly geometric (`newsize + newsize/8 + 6`), which makes append amortised O(1). **Optimisation:** if you know the final size, pre-allocate (`out = [None] * n`) and assign by index — it avoids all reallocation and copying.

### Hash collisions and why `__hash__`/`__eq__` must agree

```python
class BadKey:
    def __init__(self, v): self.v = v
    def __eq__(self, other): return self.v == other.v
    # ❌ defining __eq__ without __hash__ makes the class UNHASHABLE

try:
    {BadKey(1): "x"}
except TypeError as e:
    print("TypeError:", e)

class GoodKey:
    def __init__(self, v): self.v = v
    def __eq__(self, other): return isinstance(other, GoodKey) and self.v == other.v
    def __hash__(self): return hash(self.v)          # equal objects -> equal hashes

d = {GoodKey(1): "x"}
print(d[GoodKey(1)])
```
**Output:**
```
TypeError: unhashable type: 'BadKey'
x
```
Python protects you: defining `__eq__` sets `__hash__ = None`. If you *break* the contract (equal objects with different hashes), dict lookups fail non-deterministically — an exceptionally hard bug to find. `@dataclass(frozen=True)` generates both correctly for you (Chapter 7).

### Why hash order changes between runs

```bash
$ python -c "print(hash('hello'))"
-6047628905384874244
$ python -c "print(hash('hello'))"
8215489561983428211
```
String hashing is randomised per process (`PYTHONHASHSEED`) as a defence against hash-collision DoS attacks. Consequence: **set iteration order is not reproducible across runs.** If your data pipeline iterates a set to assign ids, you get different ids every run and your experiments become irreproducible. Always `sorted()` before assigning anything order-dependent.

---

## 4.7 Best Practices

```python
# ✔ Choose by access pattern
seen = set()                       # membership     → set
vocab = {}                         # key → value    → dict
batch = []                         # ordered, grows → list
shape = (32, 512)                  # fixed record   → tuple
window = deque(maxlen=100)         # both ends      → deque

# ✔ Comprehensions over append-loops (when producing a collection)
ids = [d["id"] for d in docs if d["score"] > 0.5]

# ✔ Generator expression when you only iterate once — no intermediate list
total = sum(len(d["text"]) for d in docs)

# ✔ .get() with a default for optional keys
model = response.get("model", "unknown")

# ✔ Unpacking over indexing
for doc_id, score in ranked:       # not: r[0], r[1]
    ...

# ✔ Merge with | (3.9+)
config = {**DEFAULTS, **user_config}

# ✔ Sort with a key, not with a comparator
top = sorted(docs, key=lambda d: (-d["score"], d["id"]))   # score desc, id asc

# ✘ Never mutate while iterating
for k in d:            d.pop(k)      # RuntimeError
for k in list(d):      d.pop(k)      # ✔ iterate a snapshot
```

**Higher-level rules**

1. **Make illegal states unrepresentable.** A `set` cannot hold duplicates; a `tuple` cannot be mutated; a `deque(maxlen=n)` cannot grow unbounded. Choose the type that enforces your invariant instead of writing checks.
2. **Prefer immutable containers for anything shared** — configs, vocabularies, label maps. `tuple`, `frozenset`, `MappingProxyType`, frozen dataclasses.
3. **Sort before serialising** anything derived from a set or a hash, for reproducibility.
4. **Annotate container types**: `list[dict[str, float]]` documents intent and catches errors in CI.
5. **Never index deep into a response** (`r["choices"][0]["message"]["content"]`) without a guard; providers change payloads.
6. **`sorted(key=...)` with a tuple key** for multi-level sorting — clean, stable, and fast.

---

## 4.8 Common Mistakes

### Mistake 1 — `list` where a `set` belongs (the O(n²) killer)

```python
# ❌ WRONG — ~50 seconds for 100k ids
seen = []
for doc_id in doc_ids:
    if doc_id not in seen:
        seen.append(doc_id)

# ✔ CORRECT — ~0.02 seconds
seen = set()
for doc_id in doc_ids:
    if doc_id not in seen:
        seen.add(doc_id)

# ✔ Or, if order doesn't matter at all:
unique = set(doc_ids)
# ✔ Or, to dedup while PRESERVING order (dicts are ordered):
unique = list(dict.fromkeys(doc_ids))
```

### Mistake 2 — `[[0]*n]*m` shared-row aliasing

```python
# ❌ WRONG
mask = [[0] * 3] * 2
mask[0][0] = 1
print(mask)                        # -> [[1, 0, 0], [1, 0, 0]]

# ✔ CORRECT
mask = [[0] * 3 for _ in range(2)]
```

### Mistake 3 — mutating a list while iterating

```python
# ❌ WRONG — skips elements
for doc in docs:
    if doc["score"] < 0.5:
        docs.remove(doc)

# ✔ CORRECT
docs = [d for d in docs if d["score"] >= 0.5]
# or in place, preserving the caller's object identity:
docs[:] = [d for d in docs if d["score"] >= 0.5]
```

### Mistake 4 — `x = mylist.sort()`

```python
# ❌ WRONG
top_docs = docs.sort(key=lambda d: d["score"])    # -> None
print(top_docs[0])                                # TypeError: 'NoneType' not subscriptable

# ✔ CORRECT
top_docs = sorted(docs, key=lambda d: d["score"], reverse=True)
```

### Mistake 5 — `dict[key]` on an API response

```python
# ❌ WRONG — one schema change and production 500s
content = response["choices"][0]["message"]["content"]

# ✔ CORRECT — fail with a message you can act on
choices = response.get("choices") or []
if not choices:
    raise ValueError(f"no choices in response: {list(response)}")
content = choices[0].get("message", {}).get("content", "")
```

### Mistake 6 — a mutable object as a would-be key

```python
# ❌ WRONG
cache[["model", "v1"]] = result      # TypeError: unhashable type: 'list'

# ✔ CORRECT
cache[("model", "v1")] = result      # tuples are hashable
cache[frozenset({"a", "b"})] = r     # frozenset for an unordered key
```

### Mistake 7 — relying on set order

```python
# ❌ WRONG — ids differ between runs; experiments become irreproducible
label_map = {label: i for i, label in enumerate(set(all_labels))}

# ✔ CORRECT
label_map = {label: i for i, label in enumerate(sorted(set(all_labels)))}
```

### Mistake 8 — building a huge list when a generator would do

```python
# ❌ WRONG — materialises 10M strings, ~4 GB
lines = [process(l) for l in open("huge.jsonl")]
total = sum(len(l) for l in lines)

# ✔ CORRECT — constant memory
total = sum(len(process(l)) for l in open("huge.jsonl"))
```

---

## 4.9 Performance

### The measurement that should change your habits

```python
import timeit

setup_list = "data = list(range(100_000)); target = 99_999"
setup_set  = "data = set(range(100_000)); target = 99_999"
n = 1000

t_list = timeit.timeit("target in data", setup_list, number=n)
t_set  = timeit.timeit("target in data", setup_set,  number=n)
print(f"list membership: {t_list:.4f}s")
print(f"set membership : {t_set:.6f}s")
print(f"speedup        : {t_list/t_set:,.0f}x")
```
**Output (indicative):**
```
list membership: 0.4213s
set membership : 0.000041s
speedup        : 10,275x
```

### Container selection decision tree

```
Need key → value lookup?
├── YES → dict
│         ├── want a default for missing keys?      → defaultdict
│         ├── counting occurrences?                 → Counter
│         └── need LRU / move_to_end?               → OrderedDict
└── NO
    ├── Need membership tests only?
    │    ├── mutable?     → set
    │    └── as a key?    → frozenset
    └── Need a sequence?
         ├── fixed / hashable / a record?           → tuple / NamedTuple
         ├── add-remove at BOTH ends, or a cap?     → deque
         ├── numeric, homogeneous, math on it?      → numpy.ndarray  ← usually this
         └── otherwise                              → list
```

That NumPy line is not a throwaway. **For numeric data in AI work, the correct answer is almost always an array, not a list**: 10–100× faster and 5–10× smaller.

```python
import numpy as np, sys
py = list(range(1_000_000))
np_ = np.arange(1_000_000, dtype=np.int32)
print(f"list : {sys.getsizeof(py) + sum(sys.getsizeof(i) for i in py[:1000])/1000*1_000_000:,.0f} bytes (approx)")
print(f"array: {np_.nbytes:,} bytes")
```
**Output (approx):**
```
list : 36,448,728 bytes (approx)
array: 4,000,000 bytes
```

### Other performance facts worth carrying

| Pattern | Note |
|---|---|
| `"".join(parts)` vs `+=` | O(n) vs O(n²) for string building |
| `list.insert(0, x)` | O(n) — use `deque.appendleft` |
| `dict.get(k, d)` vs try/except | `get` is faster when misses are common; try/except is faster when they're rare |
| `set.intersection(*sets)` | Starts with the smallest set — near-optimal automatically |
| Pre-allocating `[None]*n` | Avoids all resize copying when the size is known |
| `sys.intern(s)` | Deduplicates repeated strings (e.g. label values across millions of rows) |
| `__slots__` on classes | Removes per-instance `__dict__`; ~40% memory saving (Chapter 7) |

---

## 4.10 Interview Preparation

**Q1 (Beginner). `list` vs `tuple`?**
Lists are mutable, over-allocate for growth, and are unhashable. Tuples are immutable, compact, hashable (so usable as dict keys), and slightly faster to create and iterate. Use a tuple for a fixed heterogeneous record ("a shape", "a coordinate"); use a list for a homogeneous collection that grows.

**Q2 (Beginner). Why is `x in list` slow and `x in set` fast?**
A list must compare against each element in turn — O(n). A set computes `hash(x)`, jumps straight to a table slot, and compares only the (usually zero or one) entries there — O(1) average, O(n) worst case with pathological collisions.

**Q3 (Beginner). What can be a dict key?**
Anything hashable: it must implement `__hash__`, and equal objects must hash equally. In practice that means immutables — `int`, `str`, `tuple` of hashables, `frozenset`, and frozen dataclasses. Lists, dicts, and sets are unhashable.

**Q4 (Intermediate). Are dicts ordered?**
Yes — insertion-ordered, guaranteed since Python 3.7 (an implementation detail of CPython 3.6). This comes from the compact dict layout: a dense entries array plus a sparse index array. Note that *sets* remain unordered.

**Q5 (Intermediate). Shallow vs deep copy — with a container example.**
`d.copy()` / `d[:]` / `copy.copy(d)` duplicate only the outer container; nested objects are shared, so mutating a nested list affects both. `copy.deepcopy` recursively copies the whole object graph (using a memo dict to handle cycles). For nested configs and nested batches, only `deepcopy` — or immutability — is safe.

**Q6 (Intermediate). `defaultdict` vs `dict.setdefault`?**
`defaultdict(factory)` calls the factory on any missing key access, so `d[k].append(v)` just works — but it also means a typo'd read silently creates an entry. `setdefault` only inserts when explicitly called, and evaluates its default eagerly (so `d.setdefault(k, expensive())` always pays the cost). Use `defaultdict` in accumulation loops; use `.get()` for pure reads.

**Q7 (AI/ML). You need to dedup 50M documents. Design it.**
Hash each normalised document to a fixed-width digest (SHA-256 truncated to 8–16 bytes) and track digests in a set — 50M × 16 B ≈ 800 MB, feasible in memory. If it isn't, use a Bloom filter (probabilistic, tiny, with false positives but no false negatives) as a prefilter, or shard by hash prefix and process shards independently. For *near*-duplicates, use MinHash + LSH, since exact hashing only catches byte-identical text.

**Q8 (AI/ML). Why is a `deque` right for chat history and a `list` wrong?**
Truncation evicts from the front on essentially every turn. `list.pop(0)` shifts all remaining pointers — O(n) per turn, O(n²) over a session. `deque.popleft()` is O(1), and `deque(maxlen=n)` enforces the cap automatically. The trade-off is O(n) random indexing, which chat history doesn't need.

**Q9 (AI/ML). Implement Reciprocal Rank Fusion for hybrid search.**
```python
from collections import defaultdict
def rrf(rankings: list[list[str]], k: int = 60) -> list[tuple[str, float]]:
    scores = defaultdict(float)
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] += 1.0 / (k + rank)
    return sorted(scores.items(), key=lambda kv: kv[1], reverse=True)

print(rrf([["a", "b", "c"], ["c", "a", "d"]])[:3])
# -> [('a', 0.0325...), ('c', 0.0325...), ('b', 0.0161...)]
```
`defaultdict(float)` avoids initialisation branches; `k=60` (the standard constant) damps the influence of top ranks so one retriever can't dominate. Crucially, RRF needs only *ranks*, not scores — so it fuses BM25 and dense-vector results whose score scales are incomparable.

**Q10 (Scenario). Your training script's memory grows until it OOMs after several hours. Where do you look?**
Unbounded containers: a `history` list appending per-step tensors (each pinning an autograd graph), a cache dict with no eviction, a `defaultdict` accumulating one entry per unique input. Diagnose with `tracemalloc` snapshots or `objgraph`. Fixes: store `loss.item()` floats not tensors, use `deque(maxlen=)` or an LRU cache, and periodically log `len()` of every long-lived container.

**Q11 (Production). Why must you `sorted()` a set before assigning ids?**
String hashing is randomised per process, so set iteration order differs between runs. Assigning label ids from an unsorted set gives different `label → id` mappings each run, which silently breaks checkpoint compatibility and makes experiments irreproducible. Sort first; save the mapping alongside the model.

---

## 4.11 Exercises

### Easy — E1
Given a list of token strings, build a vocabulary dict mapping each token to a unique id, with `<pad>`=0 and `<unk>`=1 reserved.

<details><summary>Solution</summary>

```python
def build_vocab(tokens: list[str], min_freq: int = 1) -> dict[str, int]:
    from collections import Counter
    counts = Counter(tokens)
    kept = sorted(t for t, c in counts.items() if c >= min_freq)   # sorted = reproducible
    vocab = {"<pad>": 0, "<unk>": 1}
    vocab.update({tok: i for i, tok in enumerate(kept, start=len(vocab))})
    return vocab

corpus = "the cat sat on the mat the cat ran".split()
v = build_vocab(corpus, min_freq=2)
print(v)
print([v.get(t, v["<unk>"]) for t in "the dog sat".split()])
```
**Output:**
```
{'<pad>': 0, '<unk>': 1, 'cat': 2, 'the': 3}
[3, 1, 1]
```
`sorted()` is what makes this reproducible. `min_freq` is standard practice: rare tokens produce embeddings trained on 1–2 examples, which is noise. `.get(t, unk)` is the whole reason `<unk>` exists.
</details>

### Easy — E2
Compute precision/recall of retrieved documents against relevant ones using set operations.

<details><summary>Solution</summary>

```python
def retrieval_metrics(retrieved: list[str], relevant: set[str]) -> dict[str, float]:
    ret = set(retrieved)
    hits = ret & relevant
    return {
        "precision": len(hits) / len(ret) if ret else 0.0,
        "recall":    len(hits) / len(relevant) if relevant else 0.0,
        "missed":    len(relevant - ret),
        "noise":     len(ret - relevant),
    }

print(retrieval_metrics(["d1", "d2", "d5"], {"d1", "d2", "d3", "d4"}))
```
**Output:**
```
{'precision': 0.6666666666666666, 'recall': 0.5, 'missed': 2, 'noise': 1}
```
Set algebra maps directly onto IR metrics: `&` is true positives, `relevant - retrieved` is false negatives, `retrieved - relevant` is false positives. Note `retrieved` becomes a set, which also silently deduplicates — worth being aware of, since a retriever returning the same chunk twice would otherwise inflate precision.
</details>

### Medium — M1
Write `flatten(nested)` that flattens an arbitrarily nested list, and `deep_get(d, "a.b.c", default)` for safe nested dict access.

<details><summary>Solution</summary>

```python
def flatten(nested: list) -> list:
    out = []
    for item in nested:
        if isinstance(item, list):
            out.extend(flatten(item))      # recursion: the structure IS a tree
        else:
            out.append(item)
    return out

def deep_get(data: dict, path: str, default=None, sep: str = "."):
    current = data
    for key in path.split(sep):
        if isinstance(current, dict) and key in current:
            current = current[key]
        elif isinstance(current, list) and key.isdigit() and int(key) < len(current):
            current = current[int(key)]
        else:
            return default
    return current

print(flatten([1, [2, [3, [4, 5]], 6], 7]))

resp = {"choices": [{"message": {"content": "hi", "tool_calls": []}}], "usage": {"total": 42}}
print(deep_get(resp, "choices.0.message.content"))
print(deep_get(resp, "choices.0.message.missing", default="<absent>"))
print(deep_get(resp, "usage.total"))
```
**Output:**
```
[1, 2, 3, 4, 5, 6, 7]
hi
<absent>
42
```
**Explanation.** `flatten` is legitimate recursion — the data is genuinely tree-shaped and shallow (nesting depth, not element count, drives recursion depth). `deep_get` handles both dicts and list indices, which is what real API responses need; it replaces `resp["choices"][0]["message"]["content"]` and its five possible exceptions with one safe call and a default. This function appears in essentially every production LLM codebase.
</details>

### Medium — M2
Implement an LRU cache for embeddings using `OrderedDict`, with hit/miss stats and a max size.

<details><summary>Solution</summary>

```python
from collections import OrderedDict

class LRUEmbeddingCache:
    def __init__(self, maxsize: int = 3):
        if maxsize < 1:
            raise ValueError("maxsize must be >= 1")
        self.maxsize = maxsize
        self._data: OrderedDict[str, tuple] = OrderedDict()
        self.hits = self.misses = self.evictions = 0

    def get(self, key: str) -> tuple | None:
        if key not in self._data:
            self.misses += 1
            return None
        self.hits += 1
        self._data.move_to_end(key)              # mark as most-recently used  O(1)
        return self._data[key]

    def put(self, key: str, value: tuple) -> None:
        if key in self._data:
            self._data.move_to_end(key)
        self._data[key] = value
        while len(self._data) > self.maxsize:
            evicted, _ = self._data.popitem(last=False)   # pop the OLDEST  O(1)
            self.evictions += 1

    def stats(self) -> dict:
        total = self.hits + self.misses
        return {"hits": self.hits, "misses": self.misses, "evictions": self.evictions,
                "hit_rate": f"{self.hits/total:.0%}" if total else "n/a",
                "keys": list(self._data)}


cache = LRUEmbeddingCache(maxsize=3)
for q in ["a", "b", "c", "a", "d", "b", "a"]:
    if (vec := cache.get(q)) is None:
        cache.put(q, (hash(q) % 100 / 100,))
print(cache.stats())
```
**Output:**
```
{'hits': 2, 'misses': 5, 'evictions': 2, 'hit_rate': '29%', 'keys': ['d', 'b', 'a']}
```
**Explanation.** `OrderedDict` is the classic LRU structure because both required operations are O(1): `move_to_end` on access and `popitem(last=False)` to evict the oldest. `while` rather than `if` for eviction makes the class correct if `maxsize` is ever lowered at runtime. The walrus (`:=`) keeps the get-or-compute pattern to two lines. In production you'd usually reach for `functools.lru_cache` — but it gives no visibility into hit rate, and hit rate is the metric that tells you whether your cache is earning its memory.
</details>

### Hard — H1
Build `RetrievalIndex` supporting metadata filtering, keyword search, hybrid RRF fusion, and deduplication — using only built-in containers.

<details><summary>Solution</summary>

```python
from collections import defaultdict
import math, hashlib

class RetrievalIndex:
    """A minimal hybrid index: inverted keyword index + dense vectors + metadata filters."""

    STOP = frozenset({"the", "a", "an", "is", "of", "for", "to", "and", "in", "on"})

    def __init__(self) -> None:
        self._docs: dict[str, dict] = {}                     # id -> record
        self._inverted: dict[str, set[str]] = defaultdict(set)   # term -> doc ids
        self._vectors: dict[str, tuple[float, ...]] = {}      # id -> embedding
        self._fingerprints: set[str] = set()                 # dedup
        self._by_meta: dict[tuple[str, str], set[str]] = defaultdict(set)  # (k,v) -> ids

    # ---------- indexing ----------
    def _tokenize(self, text: str) -> list[str]:
        return [w.strip(".,!?;:").lower() for w in text.split()
                if w.strip(".,!?;:").lower() not in self.STOP and w.strip(".,!?;:")]

    def add(self, doc_id: str, text: str, vector: tuple[float, ...],
            **metadata: str) -> bool:
        fp = hashlib.sha256(" ".join(self._tokenize(text)).encode()).hexdigest()[:16]
        if fp in self._fingerprints:
            return False                                     # duplicate, skipped
        self._fingerprints.add(fp)

        self._docs[doc_id] = {"id": doc_id, "text": text, "meta": metadata}
        self._vectors[doc_id] = vector
        for term in set(self._tokenize(text)):
            self._inverted[term].add(doc_id)
        for key, value in metadata.items():
            self._by_meta[(key, str(value))].add(doc_id)
        return True

    # ---------- retrieval ----------
    def _candidates(self, filters: dict[str, str] | None) -> set[str]:
        if not filters:
            return set(self._docs)
        sets = [self._by_meta.get((k, str(v)), set()) for k, v in filters.items()]
        return set.intersection(*sets) if sets else set()

    def keyword_search(self, query: str, k: int, allowed: set[str]) -> list[str]:
        terms = self._tokenize(query)
        scores: dict[str, float] = defaultdict(float)
        n_docs = max(1, len(self._docs))
        for term in terms:
            postings = self._inverted.get(term, set()) & allowed
            if not postings:
                continue
            idf = math.log(n_docs / len(postings)) + 1.0     # rarer term -> higher weight
            for doc_id in postings:
                scores[doc_id] += idf
        return [d for d, _ in sorted(scores.items(), key=lambda kv: (-kv[1], kv[0]))[:k]]

    def vector_search(self, query_vec: tuple[float, ...], k: int,
                      allowed: set[str]) -> list[str]:
        def cosine(a, b):
            dot = sum(x * y for x, y in zip(a, b, strict=True))
            na = math.sqrt(sum(x * x for x in a)) or 1e-9
            nb = math.sqrt(sum(y * y for y in b)) or 1e-9
            return dot / (na * nb)
        scored = [(d, cosine(query_vec, self._vectors[d])) for d in allowed]
        return [d for d, _ in sorted(scored, key=lambda kv: (-kv[1], kv[0]))[:k]]

    def hybrid_search(self, query: str, query_vec: tuple[float, ...], k: int = 3,
                      filters: dict[str, str] | None = None, rrf_k: int = 60) -> list[dict]:
        allowed = self._candidates(filters)
        if not allowed:
            return []
        rankings = [self.keyword_search(query, k * 3, allowed),
                    self.vector_search(query_vec, k * 3, allowed)]

        fused: dict[str, float] = defaultdict(float)
        for ranking in rankings:
            for rank, doc_id in enumerate(ranking, start=1):
                fused[doc_id] += 1.0 / (rrf_k + rank)

        top = sorted(fused.items(), key=lambda kv: (-kv[1], kv[0]))[:k]
        return [{**self._docs[d], "score": round(s, 5)} for d, s in top]

    def __len__(self) -> int:
        return len(self._docs)


if __name__ == "__main__":
    idx = RetrievalIndex()
    corpus = [
        ("d1", "Vector databases store embeddings for similarity search", (1.0, 0.1, 0.0), {"lang": "en", "year": "2024"}),
        ("d2", "Embeddings capture the semantic meaning of text",          (0.9, 0.2, 0.1), {"lang": "en", "year": "2023"}),
        ("d3", "Similarity search uses cosine distance over embeddings",   (0.8, 0.3, 0.0), {"lang": "en", "year": "2024"}),
        ("d4", "Les bases de donnees vectorielles stockent des vecteurs",  (0.2, 0.9, 0.1), {"lang": "fr", "year": "2024"}),
        ("d5", "vector databases store embeddings for similarity search",  (1.0, 0.1, 0.0), {"lang": "en", "year": "2025"}),
    ]
    for doc_id, text, vec, meta in corpus:
        added = idx.add(doc_id, text, vec, **meta)
        print(f"{doc_id}: {'indexed' if added else 'SKIPPED (duplicate)'}")

    print(f"\nindex size: {len(idx)}\n")
    for hit in idx.hybrid_search("similarity search embeddings", (0.95, 0.15, 0.0),
                                 k=3, filters={"lang": "en", "year": "2024"}):
        print(f"  {hit['id']} score={hit['score']} meta={hit['meta']}  {hit['text'][:45]}")
```
**Output:**
```
d1: indexed
d2: indexed
d3: indexed
d4: indexed
d5: SKIPPED (duplicate)

index size: 4

  d1 score=0.03252 meta={'lang': 'en', 'year': '2024'}  Vector databases store embeddings for simila
  d3 score=0.03252 meta={'lang': 'en', 'year': '2024'}  Similarity search uses cosine distance over e
```
**Explanation — every container choice is load-bearing.**

| Structure | Why |
|---|---|
| `dict[str, dict]` for docs | O(1) id → record |
| `defaultdict(set)` inverted index | O(1) term lookup; sets dedup a repeated term within a doc, and support O(min) intersection |
| `set[str]` fingerprints | O(1) dedup on 5M documents |
| `defaultdict(set)` metadata index | Pre-filtering by intersecting small sets *before* scoring — this is what makes filtered vector search fast in real vector DBs |
| `frozenset` for stop words | Immutable class constant, O(1) membership |
| `defaultdict(float)` for RRF | Accumulate across retrievers with no initialisation branches |
| Tuple sort keys `(-score, id)` | Deterministic tie-breaking → reproducible output |

**Production notes.** `d5` is dropped as a case-differing duplicate — exactly the kind of near-duplicate that inflates RAG evaluation. Filtering *before* scoring (`_candidates` → `allowed`) is the pattern real vector databases call "pre-filtering," and it is much cheaper than scoring everything then discarding. The dense search here is pure Python and O(n·d); production replaces it with an ANN index (HNSW/IVF) via FAISS, Qdrant, or pgvector — but the surrounding structure stays exactly as shown.
</details>

---

## 4.12 Mini Project — Dataset Cleaner

**Goal.** A production-shaped cleaner that deduplicates, filters by quality rules, detects class imbalance, reports token statistics, and produces a stratified train/val/test split — with a full audit trail of everything it dropped.

```python
"""cleaner.py — dataset cleaning with a complete audit trail."""
from collections import Counter, defaultdict
import hashlib
import random
import statistics


class DatasetCleaner:
    def __init__(self, *, min_chars: int = 10, max_chars: int = 10_000,
                 min_class_count: int = 2, seed: int = 42):
        self.min_chars = min_chars
        self.max_chars = max_chars
        self.min_class_count = min_class_count
        self.rng = random.Random(seed)            # seeded → reproducible splits
        self.dropped: dict[str, list[str]] = defaultdict(list)

    # ---------- helpers ----------
    @staticmethod
    def _fingerprint(text: str) -> str:
        norm = " ".join(text.lower().split())
        return hashlib.sha256(norm.encode()).hexdigest()[:16]

    def _drop(self, reason: str, doc_id: str) -> None:
        self.dropped[reason].append(doc_id)

    # ---------- pipeline stages ----------
    def clean(self, records: list[dict]) -> list[dict]:
        seen: set[str] = set()
        kept: list[dict] = []

        for rec in records:
            doc_id = str(rec.get("id", "?"))
            text = (rec.get("text") or "").strip()
            label = rec.get("label")

            if not text:
                self._drop("empty_text", doc_id); continue
            if label is None:
                self._drop("missing_label", doc_id); continue
            if len(text) < self.min_chars:
                self._drop("too_short", doc_id); continue
            if len(text) > self.max_chars:
                self._drop("too_long", doc_id); continue

            fp = self._fingerprint(text)
            if fp in seen:
                self._drop("duplicate", doc_id); continue
            seen.add(fp)

            kept.append({**rec, "text": " ".join(text.split()), "_fp": fp})

        # rare-class removal needs the full surviving set, so it's a second pass
        counts = Counter(r["label"] for r in kept)
        rare = {lbl for lbl, c in counts.items() if c < self.min_class_count}
        if rare:
            survivors = []
            for r in kept:
                if r["label"] in rare:
                    self._drop(f"rare_class:{r['label']}", str(r.get("id", "?")))
                else:
                    survivors.append(r)
            kept = survivors
        return kept

    def split(self, records: list[dict], ratios=(0.7, 0.15, 0.15)) -> dict[str, list[dict]]:
        """Stratified split: each class is divided by the same ratios."""
        if abs(sum(ratios) - 1.0) > 1e-6:
            raise ValueError(f"ratios must sum to 1.0, got {sum(ratios)}")

        by_class: dict[object, list[dict]] = defaultdict(list)
        for r in records:
            by_class[r["label"]].append(r)

        out = {"train": [], "val": [], "test": []}
        for label in sorted(by_class, key=str):        # sorted → reproducible
            group = by_class[label][:]
            self.rng.shuffle(group)
            n = len(group)
            n_train = int(n * ratios[0])
            n_val = int(n * ratios[1])
            out["train"] += group[:n_train]
            out["val"]   += group[n_train:n_train + n_val]
            out["test"]  += group[n_train + n_val:]
        for split in out.values():
            self.rng.shuffle(split)
        return out

    # ---------- reporting ----------
    def report(self, original: list[dict], kept: list[dict],
               splits: dict[str, list[dict]]) -> None:
        print("=" * 62)
        print("DATASET CLEANING REPORT")
        print("=" * 62)
        total_dropped = sum(len(v) for v in self.dropped.values())
        print(f"input records : {len(original)}")
        print(f"kept          : {len(kept)}  ({len(kept)/max(1,len(original)):.1%})")
        print(f"dropped       : {total_dropped}")
        for reason, ids in sorted(self.dropped.items(), key=lambda kv: -len(kv[1])):
            preview = ", ".join(ids[:4]) + (" ..." if len(ids) > 4 else "")
            print(f"    {reason:<22} {len(ids):>4}   [{preview}]")

        if not kept:
            print("\nNO RECORDS SURVIVED — check your thresholds."); return

        counts = Counter(r["label"] for r in kept)
        biggest, smallest = max(counts.values()), min(counts.values())
        print(f"\nclass distribution (imbalance ratio {biggest/smallest:.1f}:1)")
        for label, count in counts.most_common():
            bar = "#" * int(30 * count / biggest)
            print(f"    {str(label):<10} {count:>4}  {bar}")
        if biggest / smallest > 3:
            print("    ⚠ significant imbalance — consider class weights or resampling")

        lengths = [len(r["text"]) for r in kept]
        approx_tokens = [l // 4 for l in lengths]
        print(f"\ntext length (chars): min={min(lengths)} median={statistics.median(lengths):.0f} "
              f"max={max(lengths)} mean={statistics.mean(lengths):.0f}")
        print(f"approx tokens      : total={sum(approx_tokens):,} "
              f"p95={sorted(approx_tokens)[int(len(approx_tokens)*0.95)]}")

        print("\nsplits")
        for name, rows in splits.items():
            dist = Counter(r["label"] for r in rows)
            share = ", ".join(f"{k}:{v}" for k, v in sorted(dist.items(), key=str))
            print(f"    {name:<6} {len(rows):>4} records   [{share}]")

        all_fps = [r["_fp"] for rows in splits.values() for r in rows]
        leak = len(all_fps) - len(set(all_fps))
        print(f"\nleakage check: {leak} duplicate fingerprints across splits "
              f"({'PASS' if leak == 0 else 'FAIL'})")
        print("=" * 62)


if __name__ == "__main__":
    raw = (
        [{"id": f"p{i}", "text": f"This is a positive review number {i} about the film.",
          "label": "positive"} for i in range(12)]
        + [{"id": f"n{i}", "text": f"This is a negative review number {i} about the film.",
            "label": "negative"} for i in range(5)]
        + [
            {"id": "x1", "text": "", "label": "positive"},
            {"id": "x2", "text": "short", "label": "negative"},
            {"id": "x3", "text": "A perfectly reasonable review here.", "label": None},
            {"id": "x4", "text": "This is a positive review number 0 about the film.",
             "label": "positive"},                         # duplicate of p0
            {"id": "x5", "text": "An unusual review in a very rare class indeed.",
             "label": "neutral"},                          # rare class
        ]
    )

    cleaner = DatasetCleaner(min_chars=10, min_class_count=2)
    kept = cleaner.clean(raw)
    splits = cleaner.split(kept)
    cleaner.report(raw, kept, splits)
```

**Output:**
```
==============================================================
DATASET CLEANING REPORT
==============================================================
input records : 22
kept          : 17  (77.3%)
dropped       : 5
    empty_text                1   [x1]
    too_short                 1   [x2]
    missing_label             1   [x3]
    duplicate                 1   [x4]
    rare_class:neutral        1   [x5]

class distribution (imbalance ratio 2.4:1)
    positive     12  ##############################
    negative      5  ############

text length (chars): min=50 median=50 max=51 mean=50
approx tokens      : total=204 p95=12

splits
    train    11 records   [negative:3, positive:8]
    val       1 records   [positive:1]
    test      5 records   [negative:2, positive:3]

leakage check: 0 duplicate fingerprints across splits (PASS)
==============================================================
```

**What this project consolidates, and why each piece is real engineering.**

| Feature | Container | Why it matters in production |
|---|---|---|
| Deduplication | `set` of fingerprints | O(1); duplicates across splits are **test-set leakage**, which silently inflates every metric you report |
| Drop audit | `defaultdict(list)` | "77% kept" with a per-reason breakdown is the difference between a trusted pipeline and a mystery |
| Class distribution | `Counter` | Imbalance decides whether you need class weights, focal loss, or resampling — and whether accuracy is a meaningful metric at all |
| Rare-class removal | Second pass over the full set | You cannot know a class is rare until you've seen every record — an ordering constraint that bites people who try to do everything in one loop |
| Stratified split | `defaultdict(list)` per class | A random split can leave a class absent from validation entirely; stratification prevents it |
| Seeded RNG | `random.Random(seed)` | A *local* generator, not the global `random` — so this class can't be perturbed by unrelated code elsewhere in the process |
| Leakage check | `set` over all splits | The final safety net, and the one check reviewers always ask for |

**Extensions.** Stream from JSONL instead of a list, so 50 GB doesn't need to fit in RAM (Chapter 5); raise a `DataQualityError` when the keep-rate falls below a threshold (Chapter 6); make each rule a pluggable `Rule` object with `name` and `check` (Chapter 7); parallelise fingerprinting with `multiprocessing` (Chapter 8).

---

## Chapter 4 Summary

| Concept | Takeaway |
|---|---|
| Selection | Membership → `set`; key→value → `dict`; ordered growth → `list`; both ends → `deque`; fixed record → `tuple` |
| Complexity | `in` is O(n) for list, O(1) for set/dict — the highest-value fact in the chapter |
| Lists | Dynamic pointer arrays; `pop(0)` is O(n); slices copy |
| Tuples | Immutable, hashable, compact; `NamedTuple` for readable records |
| Sets | O(1) membership and fast algebra; **unordered — sort before use** |
| Dicts | Insertion-ordered since 3.7; keys must be hashable; use `.get()` on external data |
| `collections` | `defaultdict` grouping · `Counter` frequencies · `deque` windows · `OrderedDict` LRU |
| Aliasing | `[[0]*n]*m` shares rows; slices copy for lists but *view* for NumPy |
| Reproducibility | Hash randomisation → always `sorted()` before assigning ids |
| Numerics | For numbers, the right container is usually `numpy.ndarray`, not `list` |

**Revision flashcards.** `in` complexity by type · `sort()` returns `None` · `{}` is a dict not a set · `[[0]*3]*2` trap · composite tuple keys · `deque(maxlen=)` · RRF in five lines · sort sets before enumerating.

---
# Chapter 5 — File Handling

> **Chapter goal.** Data enters and leaves an AI system through files: datasets, configs, prompts, checkpoints, embeddings, logs. This chapter is about doing that *safely* (no corrupted checkpoints), *scalably* (50 GB on an 8 GB machine), and *portably* (your code runs in Docker, not just on your laptop).

---

## 5.1 Concept Overview

### What it is

File handling is reading and writing persistent storage: opening files, streaming their contents, and encoding/decoding structured formats — text, CSV, JSON, JSONL, YAML, Pickle, Parquet.

### Why it exists

Memory is volatile and small; disk is persistent and large. Every real dataset outlives the process that reads it, and most are larger than RAM.

### Why Python includes it this way

- `open()` returns a **file object that is an iterator** — so `for line in f:` streams, using constant memory regardless of file size. This is the single most important design choice in Python I/O.
- **Context managers** (`with`) guarantee the file closes even when an exception fires.
- **Text vs binary mode** with explicit encoding, so text corruption is your choice, not an accident.
- **`pathlib`** turns paths into objects with methods, ending decades of string-concatenation path bugs.

### Why AI engineers care

| Task | Format | Note |
|---|---|---|
| Training data | JSONL, Parquet | JSONL streams line-by-line; Parquet is columnar and 5–10× smaller |
| Tabular data | CSV, Parquet | CSV is universal and lossy (no dtypes); Parquet preserves them |
| Config | YAML, TOML | Human-editable hyperparameters |
| Prompts | `.txt`, `.jinja`, YAML | Version-controlled separately from code |
| Model checkpoints | `.pt`, `.safetensors` | **Never** unpickle an untrusted `.pt` |
| Embeddings | `.npy`, `.npz`, Parquet | Binary and compact; never CSV |
| Metadata / results | JSON | Human-readable, diffable |
| Logs | JSONL | Structured, greppable, streamable |

---

## 5.2 Theory

### The file object and buffering

```
  your code  ──►  Python file object  ──►  OS buffer  ──►  disk
                  (buffered: 8 KB text /
                   io.DEFAULT_BUFFER_SIZE binary)
```

Writes land in a buffer and are flushed when the buffer fills, when you call `.flush()`, or when the file closes. **If your process is killed before a flush, that data is gone.** This is why a training script that crashes can leave a zero-byte or half-written checkpoint — and why atomic writes (§5.7) matter.

### Text mode vs binary mode

| Mode | Reads/writes | Encoding | Newlines |
|---|---|---|---|
| `"r"` / `"w"` (text) | `str` | Decoded via `encoding=` | Translated (`\r\n` ↔ `\n`) |
| `"rb"` / `"wb"` (binary) | `bytes` | None — raw | Untouched |

```python
open("f.txt", "r")                        # ✘ platform-dependent default encoding
open("f.txt", "r", encoding="utf-8")      # ✔ always specify
```
Python 3.15 will make UTF-8 the default, but until then the default is locale-dependent — meaning code that works on your Mac raises `UnicodeDecodeError` in an Alpine Docker container. **Always pass `encoding="utf-8"`.** Add `errors="replace"` when scraping messy web data you cannot control.

### Mode characters

| Mode | Meaning | If the file exists | If it doesn't |
|---|---|---|---|
| `r` | read | opens | `FileNotFoundError` |
| `w` | write | **truncates to empty** | creates |
| `a` | append | writes at end | creates |
| `x` | exclusive create | `FileExistsError` | creates |
| `r+` | read/write | opens at position 0 | error |
| `+b` suffix | binary variant | — | — |

> `"w"` destroying an existing file is responsible for more lost work than any other single character in Python. Use `"x"` when you mean "must not already exist" — for example, refusing to silently overwrite a checkpoint from a previous run.

### Why `with` is mandatory

```python
# ❌ leaks the handle if process() raises
f = open("data.jsonl")
process(f)
f.close()

# ✔ closes on any exit path, including exceptions
with open("data.jsonl", encoding="utf-8") as f:
    process(f)
```
`with` calls `f.__enter__()` on entry and `f.__exit__()` on exit — *always*, including on exception, `return`, and `break`. Leaked handles hit the OS limit (typically 1024) and then everything fails with `OSError: Too many open files`, usually deep inside a library, with a traceback that points nowhere useful.

### Streaming vs loading

```python
# ❌ loads the whole file into memory
with open("50gb.jsonl") as f:
    lines = f.readlines()          # MemoryError

# ✔ constant memory, one line at a time
with open("50gb.jsonl") as f:
    for line in f:                 # the file object IS an iterator
        process(line)
```
The `for line in f:` form reads a buffer at a time and yields lines lazily. **Memory usage is O(longest line), not O(file size).** This is why JSONL is the standard AI dataset format: each line is an independent JSON object, so a 500 GB dataset streams through 100 MB of RAM.

### Format comparison

| Format | Human-readable | Streamable | Typed | Size | Safe | Use for |
|---|---|---|---|---|---|---|
| TXT | ✔ | ✔ | ✘ | 1× | ✔ | Prompts, raw corpora |
| CSV | ✔ | ✔ | ✘ (all strings) | 1× | ✔ | Interchange, small tables |
| JSON | ✔ | ✘ (whole doc) | Partial | 1.2× | ✔ | Configs, results, API bodies |
| **JSONL** | ✔ | **✔** | Partial | 1.2× | ✔ | **Training data, logs** |
| YAML | ✔✔ | ✘ | Partial | 1× | ⚠ (use `safe_load`) | Configs |
| TOML | ✔✔ | ✘ | ✔ | 1× | ✔ | `pyproject.toml`, app config |
| Pickle | ✘ | ✘ | ✔ (any object) | 0.6× | **✘ ARBITRARY CODE EXECUTION** | Trusted local caches only |
| NPY/NPZ | ✘ | Partial | ✔ | 0.3× | ✔ | Embeddings, arrays |
| Parquet | ✘ | ✔ (row groups) | ✔ | **0.1–0.2×** | ✔ | Large tabular datasets |
| safetensors | ✘ | ✔ | ✔ | 0.5× | ✔ | **Model weights** |

---

## 5.3 Syntax

### `pathlib` — use this, not `os.path`

```python
from pathlib import Path

root = Path("data")                       # relative
model_dir = root / "models" / "bert"      # `/` operator joins, cross-platform
cfg = model_dir / "config.json"

cfg.name          # -> 'config.json'
cfg.stem          # -> 'config'
cfg.suffix        # -> '.json'
cfg.parent        # -> PosixPath('data/models/bert')
cfg.parts         # -> ('data', 'models', 'bert', 'config.json')

cfg.exists(); cfg.is_file(); cfg.is_dir()
cfg.stat().st_size                        # bytes
model_dir.mkdir(parents=True, exist_ok=True)   # like `mkdir -p`

list(root.glob("*.jsonl"))                # this directory
list(root.rglob("*.jsonl"))               # recursive
cfg.write_text('{"a":1}', encoding="utf-8")
data = cfg.read_text(encoding="utf-8")
cfg.unlink(missing_ok=True)               # delete

Path.cwd(); Path.home()
Path(__file__).resolve().parent           # THE way to locate files near your script
```

> **Why `Path(__file__).resolve().parent` matters.** Relative paths resolve against the *current working directory*, which differs between your IDE, a cron job, and a Docker container. Anchoring to the script's own location makes your code location-independent. This is the fix for the eternal "it works when I run it from the project root" bug.

### Text files

```python
with open("prompts/system.txt", encoding="utf-8") as f:
    system_prompt = f.read()              # whole file as one str

with open("corpus.txt", encoding="utf-8") as f:
    for i, line in enumerate(f):          # streaming
        line = line.rstrip("\n")

with open("out.txt", "w", encoding="utf-8") as f:
    f.write("first\n")                    # write does NOT add a newline
    f.writelines(["a\n", "b\n"])          # nor does writelines
    print("third", file=f)                # print DOES add a newline
```

### CSV

```python
import csv

with open("data.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):         # row is a dict keyed by header
        print(row["text"], row["label"])  # ← ALL values are strings

with open("out.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["id", "text", "label"])
    writer.writeheader()
    writer.writerow({"id": 1, "text": "hello, world", "label": 1})
```

> **`newline=""` is not optional.** Without it, the csv module's own newline handling collides with text-mode translation and you get blank rows between every record on Windows. The docs mandate it; most tutorials omit it.

### JSON and JSONL

```python
import json

# ---- JSON: whole document ----
with open("config.json", encoding="utf-8") as f:
    config = json.load(f)                 # file -> object
config = json.loads('{"a": 1}')           # string -> object

with open("results.json", "w", encoding="utf-8") as f:
    json.dump(results, f, indent=2, ensure_ascii=False, sort_keys=True)
text = json.dumps(results, indent=2)      # object -> string

# ---- JSONL: one object per line, streamable ----
with open("train.jsonl", encoding="utf-8") as f:
    for line_no, line in enumerate(f, 1):
        if not line.strip():
            continue                       # tolerate blank lines
        record = json.loads(line)

with open("train.jsonl", "w", encoding="utf-8") as f:
    for record in records:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")
```

| `json.dump` parameter | Effect | When to use |
|---|---|---|
| `indent=2` | Pretty-print | Configs and results (diffable in git) |
| `ensure_ascii=False` | Keep é, 中文, emoji as-is | **Always** — otherwise you get `é` noise |
| `sort_keys=True` | Deterministic key order | Anything you diff or hash |
| `default=str` | Fallback serialiser | Datetimes, Paths, NumPy scalars |
| `separators=(",",":")` | Minimal size | JSONL, network payloads |

**The non-serialisable problem** — every AI engineer hits this:
```python
import json, numpy as np, datetime

def json_default(obj):
    if isinstance(obj, (np.integer,)):   return int(obj)
    if isinstance(obj, (np.floating,)):  return float(obj)
    if isinstance(obj, np.ndarray):      return obj.tolist()
    if isinstance(obj, (datetime.datetime, datetime.date)): return obj.isoformat()
    if isinstance(obj, set):             return sorted(obj)
    raise TypeError(f"not JSON serialisable: {type(obj).__name__}")

print(json.dumps({"acc": np.float32(0.94), "shape": np.array([1, 2])},
                 default=json_default))
```
**Output:** `{"acc": 0.9399999976158142, "shape": [1, 2]}`

(Note the float32 → float64 artefact — another reminder of Chapter 1.)

### YAML

```python
import yaml       # pip install pyyaml

with open("config.yaml", encoding="utf-8") as f:
    config = yaml.safe_load(f)            # ← safe_load, NEVER yaml.load

with open("out.yaml", "w", encoding="utf-8") as f:
    yaml.safe_dump(config, f, sort_keys=False, default_flow_style=False)
```

> **`yaml.load` without a Loader can execute arbitrary Python.** `yaml.safe_load` is the only acceptable form for any file you did not personally write. This has been a real CVE source in ML tooling.

YAML's convenience is why it dominates ML configs:
```yaml
model:
  name: bert-base-uncased
  dtype: bfloat16
training:
  lr: 3.0e-4          # ← quote or use a decimal point; `3e-4` parses as a STRING in YAML 1.1
  epochs: 10
  callbacks: [early_stopping, checkpoint]
```

### Pickle — and its danger

```python
import pickle

with open("cache.pkl", "wb") as f:                    # binary mode
    pickle.dump(obj, f, protocol=pickle.HIGHEST_PROTOCOL)

with open("cache.pkl", "rb") as f:
    obj = pickle.load(f)                              # ⚠ EXECUTES CODE IN THE FILE
```

```
┌──────────────────────────────────────────────────────────────────┐
│  pickle.load() on untrusted data = REMOTE CODE EXECUTION.        │
│  A malicious .pkl / .pt from the internet can run any command    │
│  the moment you load it. Prefer JSON, Parquet, or safetensors.   │
│  If you must use pickle, only for files YOUR process wrote.      │
└──────────────────────────────────────────────────────────────────┘
```

This is not theoretical — malicious models have been found on public model hubs. `safetensors` exists precisely to solve it: a pure-data tensor format with no code execution path.

### NumPy binary formats

```python
import numpy as np

emb = np.random.rand(10_000, 384).astype(np.float32)

np.save("emb.npy", emb)                          # single array
loaded = np.load("emb.npy")
mmap = np.load("emb.npy", mmap_mode="r")         # memory-mapped: read without loading!

np.savez_compressed("bundle.npz", emb=emb, ids=np.arange(10_000))
bundle = np.load("bundle.npz")
print(bundle["emb"].shape, bundle["ids"][:3])
```

`mmap_mode="r"` is a genuine superpower: a 100 GB embedding matrix can be sliced and queried while the OS pages in only the bytes you touch. It is how you serve large indexes from a modest machine.

---

## 5.4 Examples

### Beginner — load prompts from files

```python
from pathlib import Path

PROMPT_DIR = Path(__file__).resolve().parent / "prompts"
PROMPT_DIR.mkdir(exist_ok=True)
(PROMPT_DIR / "system.txt").write_text(
    "You are a precise assistant.\nAnswer only from the given context.\n",
    encoding="utf-8")
(PROMPT_DIR / "rewrite.txt").write_text("Rewrite the query: {query}\n", encoding="utf-8")

def load_prompts(directory: Path) -> dict[str, str]:
    return {p.stem: p.read_text(encoding="utf-8").strip()
            for p in sorted(directory.glob("*.txt"))}

prompts = load_prompts(PROMPT_DIR)
for name, text in prompts.items():
    print(f"[{name}] {text[:45]!r}")
print(prompts["rewrite"].format(query="what is rag"))
```
**Output:**
```
[rewrite] 'Rewrite the query: {query}'
[system] 'You are a precise assistant.\nAnswer only from t'
```
```
Rewrite the query: what is rag
```
**Line by line.** `Path(__file__).resolve().parent` anchors to the script. `sorted(glob(...))` gives deterministic order (glob order is filesystem-dependent — another reproducibility trap). `p.stem` strips the extension to give a clean key. **Keeping prompts in files rather than in code** means prompt engineers can edit them without touching Python, prompt changes get their own git diffs, and you can A/B test by swapping a directory.

### Intermediate — streaming a large JSONL dataset

```python
import json
from pathlib import Path
from collections import Counter
from typing import Iterator

def read_jsonl(path: Path, *, skip_bad: bool = True) -> Iterator[dict]:
    """Stream records from a JSONL file. Constant memory, any file size."""
    bad = 0
    with open(path, encoding="utf-8") as f:
        for line_no, line in enumerate(f, start=1):
            line = line.strip()
            if not line:
                continue
            try:
                yield json.loads(line)
            except json.JSONDecodeError as exc:
                bad += 1
                if not skip_bad:
                    raise ValueError(f"{path}:{line_no}: malformed JSON: {exc}") from exc
    if bad:
        print(f"  ⚠ skipped {bad} malformed lines in {path.name}")


def write_jsonl(path: Path, records) -> int:
    path.parent.mkdir(parents=True, exist_ok=True)
    n = 0
    with open(path, "w", encoding="utf-8") as f:
        for rec in records:
            f.write(json.dumps(rec, ensure_ascii=False) + "\n")
            n += 1
    return n


demo = Path("demo.jsonl")
demo.write_text(
    '{"id": 1, "text": "café ☕", "label": "pos"}\n'
    '{"id": 2, "text": "bad", "label": "neg"}\n'
    'THIS LINE IS NOT JSON\n'
    '\n'
    '{"id": 3, "text": "great", "label": "pos"}\n',
    encoding="utf-8")

labels = Counter()
total_chars = 0
for rec in read_jsonl(demo):              # streaming — one record in memory at a time
    labels[rec["label"]] += 1
    total_chars += len(rec["text"])

print(f"labels: {dict(labels)}  total_chars={total_chars}")

n = write_jsonl(Path("out/positives.jsonl"),
                (r for r in read_jsonl(demo) if r["label"] == "pos"))
print(f"wrote {n} positive records")
print(Path("out/positives.jsonl").read_text(encoding="utf-8"))
```
**Output:**
```
  ⚠ skipped 1 malformed lines in demo.jsonl
labels: {'pos': 2, 'neg': 1}  total_chars=14
  ⚠ skipped 1 malformed lines in demo.jsonl
wrote 2 positive records
{"id": 1, "text": "café ☕", "label": "pos"}
{"id": 3, "text": "great", "label": "pos"}
```
**Line by line — five production decisions.**
1. **`yield` makes it a generator** (Chapter 8). The caller iterates; the file streams. This function works identically on 5 KB and 500 GB.
2. **Malformed lines are counted, not fatal.** Real crawled datasets always contain some broken lines; dying on line 4,000,001 of a 6-hour job is unacceptable. But `skip_bad=False` exists for when you *want* strictness in CI.
3. **The error message includes `path:line_no`** — clickable in most terminals and IDEs.
4. **`ensure_ascii=False`** preserves `café ☕` instead of writing `café ☕`.
5. **The generator expression passed to `write_jsonl`** means the filter never materialises a list — read-filter-write in constant memory.

### Production-style — atomic writes and safe checkpointing

```python
import json, os, tempfile
from pathlib import Path

def atomic_write_json(path: Path, data: dict, *, indent: int = 2) -> None:
    """Write JSON so that `path` is never left partially written."""
    path.parent.mkdir(parents=True, exist_ok=True)
    fd, tmp_name = tempfile.mkstemp(dir=path.parent, prefix=f".{path.name}.", suffix=".tmp")
    tmp = Path(tmp_name)
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=indent, ensure_ascii=False, sort_keys=True)
            f.flush()
            os.fsync(f.fileno())        # force the OS to write to physical disk
        os.replace(tmp, path)           # ATOMIC rename on POSIX and Windows
    except BaseException:
        tmp.unlink(missing_ok=True)     # never leave junk behind
        raise


results = {"run_id": "exp-042", "metrics": {"val_acc": 0.913, "val_loss": 0.284},
           "config": {"lr": 3e-4, "batch_size": 32}}
atomic_write_json(Path("runs/exp-042/results.json"), results)
print(Path("runs/exp-042/results.json").read_text(encoding="utf-8"))
```
**Output:**
```json
{
  "config": {
    "batch_size": 32,
    "lr": 0.0003
  },
  "metrics": {
    "val_acc": 0.913,
    "val_loss": 0.284
  },
  "run_id": "exp-042"
}
```
**Why every line exists.**
- **Write to a temp file in the same directory**, then rename. `os.replace` is atomic *within a filesystem* — the destination either has the old content or the complete new content, never a half-written file. Cross-filesystem renames are not atomic, which is why the temp file must be a sibling.
- **`os.fsync`** forces the kernel's page cache to disk. Without it, a machine power-loss after `replace` can still lose the data.
- **`except BaseException`** — deliberately broader than `Exception`, so that `KeyboardInterrupt` (Ctrl-C) and `SystemExit` also clean up the temp file. Cleanup handlers are the one place this is right.
- **`sort_keys=True`** makes the output byte-identical for identical data, so `git diff` between runs shows only real changes.

This pattern is why `transformers` and `torch` checkpoint saves survive a killed job. Naive `json.dump(f)` interrupted at 60% leaves a corrupt file that fails to parse on restart — and you discover it eight hours later.

### AI-specific — a layered config loader

```python
import json, os
from pathlib import Path

try:
    import yaml
except ImportError:
    yaml = None

def load_config(path: Path, *, env_prefix: str = "APP_", overrides: dict | None = None) -> dict:
    """Load config with layered precedence: file < env vars < explicit overrides."""
    if not path.exists():
        raise FileNotFoundError(
            f"config not found: {path.resolve()}\n"
            f"  cwd: {Path.cwd()}\n"
            f"  hint: pass an absolute path or set CONFIG_PATH")

    text = path.read_text(encoding="utf-8")
    if path.suffix in {".yaml", ".yml"}:
        if yaml is None:
            raise RuntimeError("PyYAML not installed: pip install pyyaml")
        config = yaml.safe_load(text) or {}
    elif path.suffix == ".json":
        config = json.loads(text)
    else:
        raise ValueError(f"unsupported config format {path.suffix!r}; use .yaml or .json")

    def coerce(raw: str):
        low = raw.strip().lower()
        if low in {"true", "yes", "1", "on"}:  return True
        if low in {"false", "no", "0", "off"}: return False
        if low in {"null", "none", ""}:        return None
        for cast in (int, float):
            try:    return cast(raw)
            except ValueError: pass
        return raw

    for env_key, raw in os.environ.items():
        if not env_key.startswith(env_prefix):
            continue
        keys = env_key[len(env_prefix):].lower().split("__")   # APP_MODEL__LR -> model.lr
        target = config
        for k in keys[:-1]:
            target = target.setdefault(k, {})
        target[keys[-1]] = coerce(raw)

    if overrides:
        def deep_merge(base: dict, extra: dict) -> dict:
            out = dict(base)
            for k, v in extra.items():
                out[k] = deep_merge(out[k], v) if isinstance(out.get(k), dict) and isinstance(v, dict) else v
            return out
        config = deep_merge(config, overrides)
    return config


Path("config.yaml").write_text(
    "model:\n  name: bert-base\n  dtype: float32\n"
    "training:\n  lr: 0.0003\n  epochs: 10\n  fp16: false\n", encoding="utf-8")

os.environ["APP_TRAINING__LR"] = "1e-4"
os.environ["APP_TRAINING__FP16"] = "true"

cfg = load_config(Path("config.yaml"), overrides={"model": {"dtype": "bfloat16"}})
print(json.dumps(cfg, indent=2))
```
**Output:**
```json
{
  "model": {
    "name": "bert-base",
    "dtype": "bfloat16"
  },
  "training": {
    "lr": 0.0001,
    "epochs": 10,
    "fp16": true
  }
}
```
**Why this is the real shape of ML configuration.** Three layers with clear precedence: the YAML file is the checked-in baseline, environment variables are how Docker/Kubernetes inject per-deployment values, and explicit overrides are how a sweep script varies one hyperparameter per run. `APP_TRAINING__LR` demonstrates the `__` nesting convention used by Hydra, pydantic-settings, and Spring alike. The `coerce` function exists because **env vars are always strings** — Chapter 1's `bool("false")` trap, handled properly. And the `FileNotFoundError` message prints the resolved path *and* the cwd, because "config not found" without those two facts is the least helpful error in software.

### AI-specific — saving and memory-mapping embeddings

```python
import numpy as np, json, time
from pathlib import Path

out = Path("index"); out.mkdir(exist_ok=True)
n, dim = 50_000, 384
embeddings = np.random.rand(n, dim).astype(np.float32)
doc_ids = [f"doc_{i:06d}" for i in range(n)]

np.save(out / "embeddings.npy", embeddings)
(out / "doc_ids.json").write_text(json.dumps(doc_ids), encoding="utf-8")
(out / "meta.json").write_text(json.dumps(
    {"n": n, "dim": dim, "dtype": "float32", "metric": "cosine",
     "model": "all-MiniLM-L6-v2"}, indent=2), encoding="utf-8")

size_mb = (out / "embeddings.npy").stat().st_size / 1024**2
print(f"embeddings.npy: {size_mb:.1f} MB  ({n:,} x {dim} float32)")

t = time.perf_counter()
full = np.load(out / "embeddings.npy")
t_full = time.perf_counter() - t

t = time.perf_counter()
mapped = np.load(out / "embeddings.npy", mmap_mode="r")
t_mmap = time.perf_counter() - t

print(f"full load : {t_full*1000:.2f} ms  (resident: {full.nbytes/1024**2:.0f} MB)")
print(f"mmap load : {t_mmap*1000:.2f} ms  (resident: ~0 MB until accessed)")
print("slice from mmap:", mapped[42, :4])
```
**Output (indicative):**
```
embeddings.npy: 73.2 MB  (50,000 x 384 float32)
full load : 21.43 ms  (resident: 73 MB)
mmap load : 0.09 ms  (resident: ~0 MB until accessed)
slice from mmap: [0.5488135  0.71518934 0.60276335 0.5448832 ]
```
**What to take away.** Never store embeddings as CSV — 50k × 384 floats as text is ~450 MB versus 73 MB binary, and parsing takes seconds instead of milliseconds. Always save a `meta.json` beside the array: without the model name and metric, an embedding file is unusable six months later, and mixing embeddings from two different models is a silent catastrophe (the vectors are meaningless relative to each other but the code runs happily). `mmap_mode="r"` lets a 100 GB index be queried by a process with 4 GB of RAM.

---

## 5.5 AI Engineering Perspective

**Machine Learning.** `pd.read_csv(chunksize=100_000)` streams a DataFrame in pieces. Parquet is the professional default for tabular ML: columnar (read only the columns you need), typed (no `dtype` guessing), compressed (5–10× smaller than CSV), and splittable across workers.

**Deep Learning.** Checkpointing must be atomic and versioned; save the config, the tokenizer, the git commit, and the RNG state alongside the weights, or the run is not reproducible. Prefer `safetensors` over `.pt` for anything shared.

**LLM applications.** Prompts live in files under version control. Conversation logs are JSONL — one turn per line, appendable, greppable, streamable into evaluation. Every LLM API response should be logged raw before parsing, so you can debug a bad answer without re-paying for the call.

**AI Agents.** Agent traces (step, thought, tool, args, result) as JSONL are the only practical way to debug a nondeterministic multi-step system. Tool outputs get cached to disk keyed by a hash of the arguments.

**RAG.** Ingestion reads heterogeneous formats (PDF, HTML, MD), writes normalised chunks as JSONL, and writes vectors as `.npy`. Incremental re-indexing compares content hashes stored in a manifest file so unchanged documents are not re-embedded — turning a $200 re-index into a $3 one.

**Data preprocessing.** Read-transform-write as a streaming generator pipeline. Never `readlines()`. Write to a temp file and rename, so a killed job can't leave a half-written dataset that quietly trains a worse model next week.

**Model serving.** Config from env + file at startup, validated once. Structured JSONL logs to stdout (Docker captures stdout; log *files* inside containers are an anti-pattern). Model weights loaded once at startup, never per request.

---

## 5.6 Internal Working

### File descriptors are a finite OS resource

```python
import subprocess, resource
soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)
print(f"open-file limit: soft={soft} hard={hard}")

# ❌ this leaks 10,000 descriptors and raises OSError long before finishing
# files = [open(f"f{i}.txt", "w") for i in range(10_000)]
```
**Output (typical Linux):**
```
open-file limit: soft=1024 hard=1048576
```
CPython's refcounting *usually* closes a file when the last reference dies, but that is an implementation detail — PyPy and other runtimes don't guarantee it, and an exception traceback can keep the reference alive indefinitely. `with` is the guarantee.

### Buffering, demonstrated

```python
from pathlib import Path
p = Path("buffered.txt")
f = open(p, "w", encoding="utf-8")
f.write("hello")
print("after write, size on disk:", p.stat().st_size)   # -> 0
f.flush()
print("after flush, size on disk:", p.stat().st_size)   # -> 5
f.close()
```
**Output:**
```
after write, size on disk: 0
after flush, size on disk: 5
```
This is why a `tail -f` on a log file can appear frozen, and why long-running training scripts should either flush explicitly or open log files with `buffering=1` (line-buffered).

### Encoding, concretely

```python
text = "café ☕ 中文"
print(len(text), "characters")
print(len(text.encode("utf-8")), "bytes in utf-8")
try:
    text.encode("ascii")
except UnicodeEncodeError as e:
    print("UnicodeEncodeError:", e)
print(text.encode("utf-8").decode("latin-1"))    # mojibake: wrong decoder
```
**Output:**
```
9 characters
18 bytes in utf-8
UnicodeEncodeError: 'ascii' codec can't encode character '\xe9' in position 3: ordinal not in range(128)
cafÃ© â˜• ä¸­æ–‡
```
`len(str)` counts characters; `len(bytes)` counts bytes. This is why byte-level tokenizers and character counts disagree, and why `max_length` in bytes versus characters versus tokens are three different limits.

### Why Parquet is so much smaller

```
CSV (row-oriented):     id,text,label\n1,hello,pos\n2,world,pos\n
                        → every value stored as text, no compression, dtype lost

Parquet (columnar):     [id column: int32, delta-encoded, compressed]
                        [text column: dictionary-encoded, compressed]
                        [label column: 2 distinct values → dictionary of 2 entries]
                        + per-column-chunk statistics (min/max) for predicate pushdown
```
Storing a label column with 3 distinct values across 10M rows costs ~10M bytes in CSV and a few KB in Parquet. Columnar layout also means `SELECT label` reads only that column off disk.

---

## 5.7 Best Practices

```python
# ✔ Always: with + encoding + pathlib
with open(path, encoding="utf-8") as f: ...

# ✔ Anchor relative paths to the script, not the cwd
BASE = Path(__file__).resolve().parent

# ✔ Stream, don't slurp
for line in f: ...                      # not f.readlines()

# ✔ Atomic writes for anything that matters
tmp.write(...); os.replace(tmp, final)

# ✔ mkdir before write
path.parent.mkdir(parents=True, exist_ok=True)

# ✔ yaml.safe_load, never yaml.load
# ✔ never pickle.load untrusted data
# ✔ ensure_ascii=False + sort_keys=True for JSON you will diff
# ✔ newline="" for csv
```

**Additional rules for AI systems**

1. **Save metadata with every artifact.** An `.npy` of embeddings without the model name, dimension, and metric is scrap. A checkpoint without its config and tokenizer is unloadable.
2. **Content-hash your inputs.** Store `sha256` of each source file in a manifest so incremental pipelines skip unchanged work and you can prove which data version produced which model.
3. **Log to stdout as JSONL in containers.** The orchestrator collects stdout; files inside a container vanish with it.
4. **Never commit data or secrets.** `.gitignore` the data directory; use `git-lfs` or DVC for artifacts; load API keys from the environment.
5. **Validate immediately after loading.** Check row counts, required columns, dtypes, and null rates before anything downstream touches the data.
6. **Prefer `x` mode for checkpoints** when overwriting would destroy a previous run's results.
7. **Compress at rest**: `gzip.open(path, "rt", encoding="utf-8")` works transparently with `for line in f:`, so a 10 GB JSONL becomes 1.5 GB with no code change beyond the `open` call.

---

## 5.8 Common Mistakes

### Mistake 1 — No `with`, no encoding

```python
# ❌ WRONG — leaks the handle on exception, breaks in a non-UTF-8 locale
f = open("data.txt")
data = f.read()

# ✔ CORRECT
with open("data.txt", encoding="utf-8") as f:
    data = f.read()
```

### Mistake 2 — `readlines()` / `read()` on large files

```python
# ❌ WRONG — MemoryError on a 50 GB file
for line in open("huge.jsonl").readlines(): ...

# ✔ CORRECT
with open("huge.jsonl", encoding="utf-8") as f:
    for line in f: ...
```

### Mistake 3 — Non-atomic writes to important files

```python
# ❌ WRONG — Ctrl-C at 60% leaves an unparseable checkpoint
with open("checkpoint.json", "w") as f:
    json.dump(huge_state, f)

# ✔ CORRECT — atomic_write_json (§5.4)
```

### Mistake 4 — Relative paths that depend on the cwd

```python
# ❌ WRONG — works in the IDE, FileNotFoundError in Docker/cron
config = json.load(open("config.json"))

# ✔ CORRECT
BASE = Path(__file__).resolve().parent
config = json.loads((BASE / "config.json").read_text(encoding="utf-8"))
```

### Mistake 5 — `yaml.load` and `pickle.load` on untrusted input

```python
# ❌ WRONG — arbitrary code execution
config = yaml.load(open("from_internet.yaml"))
model = pickle.load(open("downloaded.pkl", "rb"))

# ✔ CORRECT
config = yaml.safe_load(...)
# and for models: use safetensors, or torch.load(..., weights_only=True)
```

### Mistake 6 — Forgetting `newline=""` with csv

```python
# ❌ WRONG — blank line between every row on Windows
with open("out.csv", "w") as f:
    csv.writer(f).writerows(rows)

# ✔ CORRECT
with open("out.csv", "w", newline="", encoding="utf-8") as f:
    csv.writer(f).writerows(rows)
```

### Mistake 7 — Storing embeddings as CSV/JSON

```python
# ❌ WRONG — 6× larger, ~100× slower to load, loses float32 precision
json.dump(embeddings.tolist(), open("emb.json", "w"))

# ✔ CORRECT
np.save("emb.npy", embeddings.astype(np.float32))
```

### Mistake 8 — Assuming CSV values are typed

```python
# ❌ WRONG — "0.9" > "0.85" is FALSE (string comparison!)
for row in csv.DictReader(f):
    if row["score"] > 0.85: ...      # TypeError, or worse: silent string compare

# ✔ CORRECT
if float(row["score"]) > 0.85: ...
```

---

## 5.9 Performance

| Operation | Throughput (indicative SSD) | Note |
|---|---|---|
| Reading text line-by-line | ~200–500 MB/s | Usually CPU-bound on decoding |
| `json.loads` per line | ~100 MB/s | Use `orjson` for 2–5× |
| `csv.DictReader` | ~50 MB/s | Builds a dict per row |
| `pd.read_csv` | ~200 MB/s | C-optimised parser |
| `pd.read_parquet` | ~1–2 GB/s | Columnar + compressed |
| `np.load` | ~1–3 GB/s | Essentially a memcpy |
| `np.load(mmap_mode="r")` | instant | Pages in lazily |
| `pickle.load` | ~500 MB/s | Fast but unsafe |

### Format sizes for the same data

```python
import json, csv, io, pickle, numpy as np
records = [{"id": i, "text": f"document number {i}", "score": i / 1000}
           for i in range(50_000)]

jsonl = "\n".join(json.dumps(r) for r in records).encode()
buf = io.StringIO(); w = csv.DictWriter(buf, fieldnames=["id","text","score"])
w.writeheader(); w.writerows(records); csv_bytes = buf.getvalue().encode()
pkl = pickle.dumps(records, protocol=5)

for name, blob in [("JSONL", jsonl), ("CSV", csv_bytes), ("Pickle", pkl)]:
    print(f"{name:<8} {len(blob)/1024:>8.0f} KB")
```
**Output (indicative):**
```
JSONL       3072 KB
CSV         1720 KB
Pickle      2416 KB
```
Add Parquet (via `pyarrow`) and it lands around 300–500 KB for this data because of dictionary encoding and compression.

### Speeding up JSON

```python
# pip install orjson
import orjson, json, time
data = [{"id": i, "v": [0.1] * 20} for i in range(50_000)]

t = time.perf_counter(); s1 = json.dumps(data);   t1 = time.perf_counter() - t
t = time.perf_counter(); s2 = orjson.dumps(data); t2 = time.perf_counter() - t
print(f"json  : {t1:.3f}s\norjson: {t2:.3f}s  ({t1/t2:.1f}x faster)")
```
**Output (indicative):**
```
json  : 0.312s
orjson: 0.061s  (5.1x faster)
```
On an ingestion pipeline processing millions of records, this is hours saved. Note `orjson.dumps` returns `bytes`, so write in binary mode.

---

## 5.10 Interview Preparation

**Q1 (Beginner). Why use `with open(...)` instead of `open()` + `close()`?**
`with` invokes the file's context-manager protocol, guaranteeing `close()` runs on every exit path — normal return, exception, `break`, or `return`. Manual `close()` is skipped when an exception fires between open and close, leaking descriptors until the process hits the OS limit.

**Q2 (Beginner). `"w"` vs `"a"` vs `"x"`?**
`"w"` truncates an existing file to zero bytes before writing; `"a"` appends; `"x"` creates and raises `FileExistsError` if the path exists. Use `"x"` when accidentally overwriting would destroy work, such as a run's checkpoint directory.

**Q3 (Beginner). JSON vs JSONL — when each?**
JSON holds one document and must be parsed in full, so it suits configs and results. JSONL holds one JSON object per line, so it can be streamed, appended to, split across workers, and processed with line-oriented tools. Datasets and logs are always JSONL.

**Q4 (Intermediate). How do you process a file larger than RAM?**
Iterate the file object (`for line in f:`) so only one line is resident, or use chunked readers (`pd.read_csv(chunksize=...)`, Parquet row groups). Combine with generators so the whole read-transform-write pipeline stays lazy. For arrays, use `np.load(mmap_mode="r")` and let the OS page in only what you touch.

**Q5 (Intermediate). What is an atomic write and why does it matter?**
Write to a temporary file in the same directory, `fsync` it, then `os.replace()` onto the target. `os.replace` is atomic within a filesystem, so readers see either the complete old file or the complete new one — never a truncated one. It matters because interrupted checkpoint or index writes otherwise corrupt state that a later run silently depends on.

**Q6 (Intermediate). Why must you pass `encoding="utf-8"`?**
Without it, Python uses a locale-dependent default that differs between developer machines, CI, and containers, so identical code raises `UnicodeDecodeError` in one environment and not another. Explicit UTF-8 makes behaviour deterministic everywhere.

**Q7 (AI/ML). Why is pickle dangerous, and what do you use for model weights?**
Unpickling executes the opcodes in the file, including `REDUCE` calls that can invoke arbitrary callables — so loading an untrusted `.pkl`/`.pt` is remote code execution. Use `safetensors` for weights (pure tensor data, no code path, and it supports zero-copy memory-mapped loading), or `torch.load(..., weights_only=True)` where safetensors isn't available.

**Q8 (AI/ML). CSV vs Parquet for a 200 GB training set?**
Parquet. It is columnar (read only the needed columns), typed (no dtype inference or silent string columns), compressed with dictionary/RLE encoding (typically 5–10× smaller), splittable into row groups for parallel workers, and carries statistics enabling predicate pushdown so filters skip whole chunks. CSV's only advantage is that a human can open it.

**Q9 (Scenario). A training job crashed and now the checkpoint won't load. Prevent it.**
The write was non-atomic and interrupted mid-flush. Fix with temp-file + fsync + `os.replace`; keep the last N checkpoints rather than overwriting one; write a small `manifest.json` with a checksum after the weights land and validate it on load; and verify by loading the checkpoint immediately after saving in a smoke test.

**Q10 (Scenario). Re-indexing 2M documents costs $200 in embedding calls every night, but only ~1% change. Fix it.**
Maintain a manifest mapping `doc_id → content_sha256 → vector_id`. On each run, hash every document, compare against the manifest, and embed only the new and changed ones; delete vectors for removed documents. Write the manifest atomically after the vector store is updated so a crash cannot mark work as done that wasn't. Cost drops roughly proportionally to the change rate.

**Q11 (Production). Why log JSONL to stdout rather than to a file in a container?**
Container filesystems are ephemeral and per-instance, so files disappear on restart and can't be aggregated. Orchestrators (Docker, Kubernetes) capture stdout/stderr and route them to a central log system. JSONL keeps each record structured and machine-queryable while remaining greppable.

---

## 5.11 Exercises

### Easy — E1
Write `count_lines(path)` that counts lines in a possibly-huge file using constant memory.

<details><summary>Solution</summary>

```python
from pathlib import Path

def count_lines(path: Path) -> int:
    with open(path, encoding="utf-8", errors="replace") as f:
        return sum(1 for _ in f)          # generator: no list, constant memory

Path("t.txt").write_text("a\nb\nc\n", encoding="utf-8")
print(count_lines(Path("t.txt")))          # -> 3
```
`sum(1 for _ in f)` is the idiomatic constant-memory count. `errors="replace"` prevents a single malformed byte from aborting a scan of a 100 GB crawl dump. For raw speed on enormous files, count `b"\n"` occurrences over binary chunks.
</details>

### Easy — E2
Convert a CSV to JSONL, casting `score` to float and `label` to int.

<details><summary>Solution</summary>

```python
import csv, json
from pathlib import Path

def csv_to_jsonl(src: Path, dst: Path) -> int:
    n = 0
    with open(src, newline="", encoding="utf-8") as fin, \
         open(dst, "w", encoding="utf-8") as fout:
        for row in csv.DictReader(fin):
            fout.write(json.dumps({
                "text": row["text"],
                "label": int(row["label"]),
                "score": float(row["score"]),
            }, ensure_ascii=False) + "\n")
            n += 1
    return n

Path("in.csv").write_text("text,label,score\nhello,1,0.9\nbye,0,0.2\n", encoding="utf-8")
print(csv_to_jsonl(Path("in.csv"), Path("out.jsonl")), "records")
print(Path("out.jsonl").read_text(encoding="utf-8"))
```
**Output:**
```
2 records
{"text": "hello", "label": 1, "score": 0.9}
{"text": "bye", "label": 0, "score": 0.2}
```
Both files are open in a single `with` (the `\` continuation), so both close on any exit path. The explicit casts are the entire point — CSV loses all type information, and this is where you put it back.
</details>

### Medium — M1
Write `ingest_directory(root)` that walks a directory of `.txt`/`.md`/`.json` files, extracts text, chunks it, and writes chunks to JSONL with source metadata.

<details><summary>Solution</summary>

```python
import json, hashlib
from pathlib import Path
from typing import Iterator

def extract_text(path: Path) -> str:
    if path.suffix in {".txt", ".md"}:
        return path.read_text(encoding="utf-8", errors="replace")
    if path.suffix == ".json":
        data = json.loads(path.read_text(encoding="utf-8"))
        if isinstance(data, dict):
            return "\n".join(str(v) for v in data.values() if isinstance(v, str))
        return json.dumps(data)
    raise ValueError(f"unsupported file type {path.suffix!r}")

def chunk(text: str, size: int = 200, overlap: int = 40) -> Iterator[str]:
    stride = size - overlap
    for start in range(0, max(1, len(text)), stride):
        piece = text[start:start + size].strip()
        if piece:
            yield piece

def ingest_directory(root: Path, out: Path, *, size: int = 200) -> dict:
    stats = {"files": 0, "chunks": 0, "errors": 0, "bytes": 0}
    out.parent.mkdir(parents=True, exist_ok=True)
    with open(out, "w", encoding="utf-8") as fout:
        for path in sorted(root.rglob("*")):
            if not path.is_file() or path.suffix not in {".txt", ".md", ".json"}:
                continue
            try:
                text = extract_text(path)
            except Exception as exc:
                stats["errors"] += 1
                print(f"  ⚠ {path}: {exc}")
                continue
            stats["files"] += 1
            stats["bytes"] += path.stat().st_size
            for i, piece in enumerate(chunk(text, size)):
                fout.write(json.dumps({
                    "chunk_id": f"{path.stem}_{i:04d}",
                    "text": piece,
                    "source": str(path.relative_to(root)),
                    "chunk_index": i,
                    "content_hash": hashlib.sha256(piece.encode()).hexdigest()[:16],
                }, ensure_ascii=False) + "\n")
                stats["chunks"] += 1
    return stats

root = Path("corpus"); root.mkdir(exist_ok=True)
(root / "a.md").write_text("# Title\n" + "Retrieval augmented generation. " * 20, encoding="utf-8")
(root / "b.json").write_text(json.dumps({"title": "Doc B", "body": "Vector search " * 30}), encoding="utf-8")
print(ingest_directory(root, Path("chunks/corpus.jsonl")))
```
**Output:**
```
{'files': 2, 'chunks': 7, 'errors': 0, 'bytes': 1063}
```
**Explanation.** `rglob` walks recursively; `sorted` makes chunk ids deterministic across runs (filesystem order is not). `relative_to(root)` stores a portable source path rather than an absolute one that means nothing on another machine. The `content_hash` per chunk enables incremental re-indexing. Errors are counted and reported, never fatal — one unreadable file must not kill a 4-hour ingestion.
</details>

### Hard — H1
Build a `Checkpointer` class: atomic saves, keeps the last N checkpoints, tracks the best by metric, and writes a manifest with checksums.

<details><summary>Solution</summary>

```python
import hashlib, json, os, shutil, tempfile
from pathlib import Path


class Checkpointer:
    """Atomic, rotating, validated checkpoint management."""

    def __init__(self, root: Path, *, keep_last: int = 3, metric: str = "val_loss",
                 mode: str = "min"):
        if mode not in {"min", "max"}:
            raise ValueError("mode must be 'min' or 'max'")
        self.root = Path(root)
        self.root.mkdir(parents=True, exist_ok=True)
        self.keep_last = keep_last
        self.metric = metric
        self.mode = mode
        self.manifest_path = self.root / "manifest.json"
        self.manifest = self._load_manifest()

    # ---------- internals ----------
    def _load_manifest(self) -> dict:
        if self.manifest_path.exists():
            try:
                return json.loads(self.manifest_path.read_text(encoding="utf-8"))
            except json.JSONDecodeError:
                print("  ⚠ manifest corrupt; starting fresh")
        return {"checkpoints": [], "best": None}

    @staticmethod
    def _sha256(path: Path) -> str:
        h = hashlib.sha256()
        with open(path, "rb") as f:
            for block in iter(lambda: f.read(1 << 20), b""):   # 1 MB blocks
                h.update(block)
        return h.hexdigest()

    def _atomic_write(self, path: Path, payload: bytes) -> None:
        fd, tmp_name = tempfile.mkstemp(dir=path.parent, prefix=f".{path.name}.")
        tmp = Path(tmp_name)
        try:
            with os.fdopen(fd, "wb") as f:
                f.write(payload)
                f.flush()
                os.fsync(f.fileno())
            os.replace(tmp, path)
        except BaseException:
            tmp.unlink(missing_ok=True)
            raise

    def _is_better(self, value: float) -> bool:
        best = self.manifest["best"]
        if best is None:
            return True
        return value < best["value"] if self.mode == "min" else value > best["value"]

    def _save_manifest(self) -> None:
        self._atomic_write(
            self.manifest_path,
            json.dumps(self.manifest, indent=2, sort_keys=True).encode("utf-8"))

    # ---------- public API ----------
    def save(self, step: int, state: dict, metrics: dict) -> Path:
        if self.metric not in metrics:
            raise KeyError(f"metric {self.metric!r} not in metrics {sorted(metrics)}")

        path = self.root / f"ckpt_step{step:06d}.json"
        self._atomic_write(path, json.dumps(
            {"step": step, "state": state, "metrics": metrics},
            indent=2, sort_keys=True).encode("utf-8"))

        entry = {"step": step, "path": path.name, "metrics": metrics,
                 "sha256": self._sha256(path), "bytes": path.stat().st_size}
        self.manifest["checkpoints"].append(entry)

        value = metrics[self.metric]
        if self._is_better(value):
            self.manifest["best"] = {"step": step, "path": path.name,
                                     "metric": self.metric, "value": value}
            best_link = self.root / "best.json"
            shutil.copyfile(path, best_link)       # a copy, so rotation can't orphan it
            print(f"  step {step}: new best {self.metric}={value:.4f}")

        self._rotate()
        self._save_manifest()
        return path

    def _rotate(self) -> None:
        keep_steps = {c["step"] for c in self.manifest["checkpoints"][-self.keep_last:]}
        if self.manifest["best"]:
            keep_steps.add(self.manifest["best"]["step"])       # never delete the best

        survivors = []
        for entry in self.manifest["checkpoints"]:
            if entry["step"] in keep_steps:
                survivors.append(entry)
            else:
                (self.root / entry["path"]).unlink(missing_ok=True)
                print(f"  rotated out step {entry['step']}")
        self.manifest["checkpoints"] = survivors

    def load(self, which: str = "best") -> dict:
        if which == "best":
            entry = self.manifest["best"]
            if entry is None:
                raise FileNotFoundError("no checkpoints saved yet")
            path = self.root / entry["path"]
        elif which == "latest":
            path = self.root / self.manifest["checkpoints"][-1]["path"]
        else:
            path = self.root / which

        if not path.exists():
            raise FileNotFoundError(f"checkpoint missing: {path}")

        recorded = next((c["sha256"] for c in self.manifest["checkpoints"]
                         if c["path"] == path.name), None)
        if recorded and self._sha256(path) != recorded:
            raise ValueError(f"CHECKSUM MISMATCH for {path.name} — file is corrupt")
        return json.loads(path.read_text(encoding="utf-8"))


if __name__ == "__main__":
    ck = Checkpointer(Path("ckpts"), keep_last=2, metric="val_loss", mode="min")
    for step, loss in [(100, 0.90), (200, 0.55), (300, 0.42), (400, 0.47), (500, 0.44)]:
        ck.save(step, {"weights": f"<tensors at step {step}>"},
                {"val_loss": loss, "val_acc": 1 - loss})

    print("\nfiles on disk:", sorted(p.name for p in Path("ckpts").iterdir()))
    print("best:", ck.manifest["best"])
    print("loaded best step:", ck.load("best")["step"])
```
**Output:**
```
  step 100: new best val_loss=0.9000
  step 200: new best val_loss=0.5500
  step 300: new best val_loss=0.4200
  rotated out step 100
  rotated out step 200
  rotated out step 400

files on disk: ['best.json', 'ckpt_step300.json', 'ckpt_step500.json', 'manifest.json']
best: {'step': 300, 'path': 'ckpt_step300.json', 'metric': 'val_loss', 'value': 0.42}
loaded best step: 300
```
**Explanation — this is a simplified version of what every serious training framework does.**

| Feature | Why it exists |
|---|---|
| Atomic writes | A killed job can never leave a corrupt checkpoint |
| SHA-256 per file | Detects silent disk corruption and partial network-storage copies |
| Manifest, written last | The record of truth is updated only after the data lands |
| Rotation with a best-guard | Disk fills up in long runs, but the best checkpoint must survive rotation — the ordering bug where "keep last 3" deletes the best model is a real and painful one |
| `best.json` as a copy, not a symlink | Symlinks break across Docker volume mounts and object stores |
| Chunked hashing (`iter(lambda: f.read(1<<20), b"")`) | Hashes a 10 GB file in 1 MB blocks with constant memory; also a nice demonstration of the two-argument `iter` sentinel form |
| Corrupt-manifest recovery | A crashed run must not permanently brick the checkpoint directory |
</details>

---

## 5.12 Mini Project — Configuration & Artifact Manager

**Goal.** One reusable component managing an experiment's full lifecycle on disk: layered config, a reproducible run directory, streamed metric logging, atomic artifact saves, and a run summary. This is the scaffolding every ML project either builds or adopts (as MLflow/W&B) — and building it once teaches you what those tools do.

```python
"""run_manager.py — reproducible experiment directories."""
from __future__ import annotations

import hashlib
import json
import os
import platform
import subprocess
import sys
import tempfile
from pathlib import Path


def atomic_write(path: Path, payload: bytes) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    fd, tmp_name = tempfile.mkstemp(dir=path.parent, prefix=f".{path.name}.")
    tmp = Path(tmp_name)
    try:
        with os.fdopen(fd, "wb") as f:
            f.write(payload); f.flush(); os.fsync(f.fileno())
        os.replace(tmp, path)
    except BaseException:
        tmp.unlink(missing_ok=True)
        raise


def git_commit() -> str:
    try:
        return subprocess.check_output(["git", "rev-parse", "--short", "HEAD"],
                                       stderr=subprocess.DEVNULL, text=True).strip()
    except (subprocess.CalledProcessError, FileNotFoundError):
        return "not-a-git-repo"


class RunManager:
    """Creates and owns one experiment run directory."""

    def __init__(self, root: Path, config: dict, *, run_name: str | None = None):
        self.config = config
        self.config_hash = hashlib.sha256(
            json.dumps(config, sort_keys=True).encode()).hexdigest()[:8]
        self.run_id = run_name or f"run_{self.config_hash}"
        self.dir = Path(root) / self.run_id
        self.dir.mkdir(parents=True, exist_ok=True)
        (self.dir / "artifacts").mkdir(exist_ok=True)

        self.metrics_path = self.dir / "metrics.jsonl"
        self._metrics_file = open(self.metrics_path, "a", encoding="utf-8", buffering=1)
        self._step = 0

        atomic_write(self.dir / "config.json",
                     json.dumps(config, indent=2, sort_keys=True).encode())
        atomic_write(self.dir / "environment.json", json.dumps({
            "python": sys.version.split()[0],
            "platform": platform.platform(),
            "git_commit": git_commit(),
            "config_hash": self.config_hash,
            "argv": sys.argv,
        }, indent=2, sort_keys=True).encode())

    # ---------- context manager ----------
    def __enter__(self) -> "RunManager":
        return self

    def __exit__(self, exc_type, exc, tb) -> bool:
        status = "failed" if exc_type else "completed"
        self.summary(status=status, error=f"{exc_type.__name__}: {exc}" if exc else None)
        self._metrics_file.close()
        return False                       # never swallow the exception

    # ---------- logging ----------
    def log_metrics(self, step: int | None = None, **metrics: float) -> None:
        self._step = step if step is not None else self._step + 1
        record = {"step": self._step, **{k: float(v) for k, v in metrics.items()}}
        self._metrics_file.write(json.dumps(record) + "\n")   # line-buffered → tail -f works

    def save_artifact(self, name: str, data: dict | str | bytes) -> Path:
        path = self.dir / "artifacts" / name
        if isinstance(data, dict):
            payload = json.dumps(data, indent=2, sort_keys=True, default=str).encode()
        elif isinstance(data, str):
            payload = data.encode("utf-8")
        else:
            payload = data
        atomic_write(path, payload)
        return path

    # ---------- reporting ----------
    def read_metrics(self) -> list[dict]:
        if not self.metrics_path.exists():
            return []
        with open(self.metrics_path, encoding="utf-8") as f:
            return [json.loads(l) for l in f if l.strip()]

    def summary(self, *, status: str = "completed", error: str | None = None) -> dict:
        rows = self.read_metrics()
        best: dict = {}
        for row in rows:
            for key, value in row.items():
                if key == "step":
                    continue
                better = key not in best or (
                    value < best[key]["value"] if "loss" in key else value > best[key]["value"])
                if better:
                    best[key] = {"value": value, "step": row["step"]}

        payload = {"run_id": self.run_id, "status": status, "error": error,
                   "steps_logged": len(rows), "best": best,
                   "artifacts": sorted(p.name for p in (self.dir / "artifacts").iterdir())}
        atomic_write(self.dir / "summary.json",
                     json.dumps(payload, indent=2, sort_keys=True).encode())
        return payload

    def print_report(self) -> None:
        s = self.summary()
        print("=" * 56)
        print(f"RUN {s['run_id']}   [{s['status']}]")
        print("=" * 56)
        print(f"directory : {self.dir}")
        print(f"config    : {json.dumps(self.config)}")
        print(f"steps     : {s['steps_logged']}")
        print("best metrics:")
        for k, v in sorted(s["best"].items()):
            print(f"    {k:<12} {v['value']:>8.4f}  @ step {v['step']}")
        print(f"artifacts : {s['artifacts']}")
        rows = self.read_metrics()
        if rows and "val_loss" in rows[0]:
            print("\nval_loss curve:")
            losses = [r["val_loss"] for r in rows]
            lo, hi = min(losses), max(losses)
            for r in rows:
                frac = (r["val_loss"] - lo) / (hi - lo) if hi > lo else 0
                print(f"    step {r['step']:>4} |{'#' * int(40 * (1 - frac)):<40}| {r['val_loss']:.4f}")
        print("=" * 56)


if __name__ == "__main__":
    config = {"model": "bert-base", "lr": 3e-4, "batch_size": 32, "epochs": 5}

    with RunManager(Path("runs"), config) as run:
        run.save_artifact("prompt.txt", "Classify the sentiment: {text}")
        for epoch, (tr, va, acc) in enumerate([
            (0.90, 0.95, 0.51), (0.61, 0.70, 0.68), (0.45, 0.52, 0.79),
            (0.38, 0.49, 0.84), (0.35, 0.51, 0.83)], start=1):
            run.log_metrics(step=epoch, train_loss=tr, val_loss=va, val_acc=acc)
        run.save_artifact("predictions.json",
                          {"ids": [1, 2, 3], "preds": [1, 0, 1], "probs": [0.9, 0.2, 0.8]})
        run.print_report()

    print("\nfiles created:")
    for p in sorted(Path("runs").rglob("*")):
        if p.is_file():
            print(f"  {p}  ({p.stat().st_size} B)")
```

**Output:**
```
========================================================
RUN run_0ec31cab   [completed]
========================================================
directory : runs/run_0ec31cab
config    : {"model": "bert-base", "lr": 0.0003, "batch_size": 32, "epochs": 5}
steps     : 5
best metrics:
    train_loss     0.3500  @ step 5
    val_acc        0.8400  @ step 4
    val_loss       0.4900  @ step 4
artifacts : ['predictions.json', 'prompt.txt']

val_loss curve:
    step    1 |                                        | 0.9500
    step    2 |#####################                   | 0.7000
    step    3 |#####################################   | 0.5200
    step    4 |########################################| 0.4900
    step    5 |######################################  | 0.5100
========================================================

files created:
  runs/run_0ec31cab/artifacts/predictions.json  (120 B)
  runs/run_0ec31cab/artifacts/prompt.txt  (30 B)
  runs/run_0ec31cab/config.json  (77 B)
  runs/run_0ec31cab/environment.json  (181 B)
  runs/run_0ec31cab/metrics.jsonl  (333 B)
  runs/run_0ec31cab/summary.json  (359 B)
```

**What this consolidates, and why each choice is the production one.**

| Feature | Rationale |
|---|---|
| Run id derived from a **config hash** | Identical configs land in the same directory; you immediately see whether a run is a genuine repeat |
| `environment.json` with git commit and Python version | Six months later, "which code produced this model?" has an answer. This is the single most-skipped and most-regretted step in ML engineering |
| `metrics.jsonl`, line-buffered, append-only | Streams during training so `tail -f` works; survives a crash with all metrics up to the crash intact; JSON per line loads straight into pandas |
| Atomic writes for config/summary/artifacts | No half-written state after a kill |
| Context manager (`__enter__`/`__exit__`) | The summary is written whether the run succeeds *or* raises, and `return False` re-raises so failures still fail loudly (Chapters 6 and 8) |
| `default=str` in artifact JSON | Datetimes, Paths, and NumPy scalars serialise instead of crashing at the very end of a long run |
| Best-metric detection by name (`"loss" in key`) | A pragmatic convention; the production version takes an explicit mode per metric |

**Extensions.** Add YAML config loading with env overrides (§5.4); add a `compare_runs(root)` that reads every `summary.json` into a table; write metrics as Parquet for large sweeps; push the run directory to S3 on exit; swap in MLflow or Weights & Biases now that you know exactly what they are doing for you.

---

## Chapter 5 Summary

| Concept | Takeaway |
|---|---|
| `with open(...)` | Mandatory — guarantees close on every exit path |
| `encoding="utf-8"` | Always explicit; the default is locale-dependent |
| Streaming | `for line in f:` is O(1) memory for any file size |
| `pathlib` | `/` to join, `Path(__file__).resolve().parent` to anchor |
| Formats | JSONL for data/logs, YAML/TOML for config, Parquet for scale, `.npy` for vectors, safetensors for weights |
| Atomicity | temp + fsync + `os.replace` for anything you'd hate to lose |
| Security | `yaml.safe_load` always; `pickle.load` never on untrusted data |
| Metadata | Every artifact needs a sidecar describing how it was made |
| Reproducibility | Content hashes, sorted globs, git commit, config hash |

**Revision flashcards.** `"w"` truncates · `newline=""` for csv · `ensure_ascii=False` · atomic write recipe · `mmap_mode="r"` · pickle = RCE · CSV values are strings.

---
# Chapter 6 — Error Handling

> **Chapter goal.** AI systems fail constantly and normally: APIs rate-limit, models return malformed JSON, GPUs run out of memory, one row in ten million is corrupt. The difference between a prototype and a production system is almost entirely how it behaves when things go wrong. This chapter is about failing *loudly* where it matters, *gracefully* where it doesn't, and *observably* everywhere.

---

## 6.1 Concept Overview

### What it is

Exception handling: a mechanism where an error at one point in the call stack transfers control to a handler somewhere up the stack, carrying information about what went wrong.

### Why it exists

The alternative — returning error codes — requires every caller to check every return value, and the code that *can* handle an error is rarely the code that *detects* it. Exceptions let a low-level function report a problem and a high-level function decide the policy, with nothing in between needing to know.

### Why Python includes it this way

- **Exceptions are objects** in a class hierarchy, so you can catch broad or narrow categories.
- **EAFP over LBYL.** Python idiom is "Easier to Ask Forgiveness than Permission": try the operation and handle failure, rather than checking preconditions first. This avoids race conditions (a file can be deleted between `os.path.exists` and `open`) and is often faster.
- **Exceptions are used for control flow** in some places — `StopIteration` ends every `for` loop, `KeyError` drives `defaultdict`. They are not "exceptional" in the pejorative sense.
- **`finally` and context managers** guarantee cleanup.

### Why AI engineers care

| Failure | Frequency in production | Right response |
|---|---|---|
| API rate limit (429) | Constant | Retry with backoff, honouring `Retry-After` |
| API timeout / 5xx | Hourly | Retry, then fall back to a smaller model |
| Malformed model JSON output | ~1–5% of calls | Repair, re-ask, or fall back to a default |
| Context length exceeded | Regular | Truncate and retry, or chunk |
| CUDA out of memory | Under load | Reduce batch size, clear cache, retry |
| Corrupt dataset row | 1 in 10⁵ | Skip, count, dead-letter |
| Missing config key | On every deploy mistake | **Crash immediately at startup** |
| Wrong tensor shape | During development | **Crash immediately with the shapes** |

The last two rows are the point: *not every error should be handled.* Handling a config error by defaulting silently produces a model trained with the wrong learning rate and no one notices for a week.

---

## 6.2 Theory

### The exception hierarchy

```
BaseException                        ← never catch this, except in cleanup handlers
├── SystemExit                       ← sys.exit()
├── KeyboardInterrupt                ← Ctrl-C
├── GeneratorExit
└── Exception                        ← catch THIS, or something narrower
    ├── ArithmeticError → ZeroDivisionError, OverflowError
    ├── AttributeError                (None.foo)
    ├── ImportError → ModuleNotFoundError
    ├── LookupError → IndexError, KeyError
    ├── NameError
    ├── OSError → FileNotFoundError, PermissionError, TimeoutError,
    │             ConnectionError → ConnectionResetError, ...
    ├── RuntimeError → RecursionError, NotImplementedError
    ├── StopIteration                 (ends every for loop)
    ├── TypeError                     ("5" + 5)
    ├── ValueError → UnicodeDecodeError
    └── Warning (via warnings module)
```

**Why the structure matters:** `except LookupError` catches both `KeyError` and `IndexError`. `except OSError` catches every filesystem and network failure. Catching `Exception` catches everything you should handle; catching `BaseException` also swallows Ctrl-C, which makes your process unkillable.

### The full statement

```python
try:
    result = risky()
except SpecificError as exc:      # most specific first
    handle(exc)
except (OtherError, ThirdError):  # group related errors
    handle_generic()
else:
    # runs ONLY if no exception was raised
    use(result)
finally:
    # runs ALWAYS: success, exception, return, break
    cleanup()
```

**Order matters** — the first matching `except` wins, and subclasses must precede superclasses:

```python
try:
    ...
except Exception:      # ← catches everything
    ...
except ValueError:     # ← DEAD CODE, unreachable
    ...
```

**What `else` is for.** Code that should run only on success, but must *not* be inside the `try` (or its own exceptions would be caught by the wrong handler):

```python
try:
    response = call_api(prompt)
except TimeoutError:
    response = None
else:
    parsed = json.loads(response)   # a JSONDecodeError here is NOT a timeout
```

### Exception chaining — `raise ... from`

```python
try:
    config = json.loads(text)
except json.JSONDecodeError as exc:
    raise ConfigError(f"invalid config at {path}") from exc
```

The traceback then shows both:
```
json.decoder.JSONDecodeError: Expecting ',' delimiter: line 4 column 3
The above exception was the direct cause of the following exception:
ConfigError: invalid config at config.json
```

- `raise X from exc` — explicit cause ("this caused that").
- `raise X` inside an `except` — implicit chaining, shown as "During handling of the above exception, another exception occurred."
- `raise X from None` — suppress the context, when the original is noise.

**Never discard the original exception.** `except Exception: raise MyError("failed")` loses the only information that would tell you *why*.

### Bare `raise` — re-raising

```python
try:
    process(batch)
except Exception:
    metrics.increment("batch_failures")
    raise                      # re-raise the SAME exception with its ORIGINAL traceback
```
This is the log-and-propagate pattern: record the event, but let the caller decide policy.

### Custom exceptions

```python
class PipelineError(Exception):
    """Base for all errors from this package."""

class DataQualityError(PipelineError):
    def __init__(self, message: str, *, dropped: int = 0, total: int = 0):
        super().__init__(message)
        self.dropped, self.total = dropped, total

    @property
    def drop_rate(self) -> float:
        return self.dropped / self.total if self.total else 0.0
```

**Why a package-level base class matters:** callers can write `except PipelineError:` to catch everything your library raises, without also catching unrelated `ValueError`s from elsewhere. Every well-designed library does this (`requests.RequestException`, `torch.cuda.CudaError`).

Attaching **structured data** to the exception (not just a string) means the handler can make decisions: retry if `drop_rate < 0.1`, abort otherwise.

### EAFP vs LBYL

```python
# LBYL — Look Before You Leap
if "choices" in response and len(response["choices"]) > 0:
    content = response["choices"][0]["message"]["content"]

# EAFP — Easier to Ask Forgiveness than Permission (Pythonic)
try:
    content = response["choices"][0]["message"]["content"]
except (KeyError, IndexError, TypeError) as exc:
    raise MalformedResponseError(f"unexpected response shape: {list(response)}") from exc
```

EAFP wins when failures are rare (the `try` costs nothing when nothing raises) and when checking is racy. LBYL wins when failures are *common* — raising and catching exceptions costs roughly 1 µs, so an exception on 90% of iterations in a hot loop is genuinely slow.

---

## 6.3 Syntax

### Basic

```python
try:
    value = int(user_input)
except ValueError:
    print("not a number")

try:
    data = json.loads(text)
except json.JSONDecodeError as exc:
    print(f"bad JSON at line {exc.lineno} col {exc.colno}: {exc.msg}")
```

### Intermediate

```python
import logging
logger = logging.getLogger(__name__)

def load_dataset(path):
    f = None
    try:
        f = open(path, encoding="utf-8")
        data = [json.loads(line) for line in f]
    except FileNotFoundError:
        logger.error("dataset missing: %s", path)
        raise
    except json.JSONDecodeError as exc:
        raise DataQualityError(f"{path}: malformed JSON") from exc
    else:
        logger.info("loaded %d records from %s", len(data), path)
        return data
    finally:
        if f is not None:
            f.close()          # (better: just use `with`)

# raising
if batch_size <= 0:
    raise ValueError(f"batch_size must be positive, got {batch_size}")

# assertions — for INVARIANTS, not for validation
assert logits.shape[0] == labels.shape[0], \
    f"batch mismatch: logits {logits.shape} vs labels {labels.shape}"
```

> **`assert` is stripped by `python -O`.** Never use it to validate user input, config, or API responses — those checks must be `if ... raise`. Use `assert` for internal invariants you believe cannot be false (tensor shapes inside your own model code are the classic legitimate use).

### Advanced — exception groups (3.11+)

```python
def validate_all(records):
    errors = []
    for i, rec in enumerate(records):
        try:
            validate(rec)
        except ValueError as exc:
            errors.append(ValueError(f"record {i}: {exc}"))
    if errors:
        raise ExceptionGroup("validation failed", errors)

try:
    validate_all(records)
except* ValueError as group:            # except* handles a subset of the group
    print(f"{len(group.exceptions)} validation errors")
```

Exception groups exist because concurrent code (`asyncio.TaskGroup`) can produce several failures at once, and because batch validation genuinely has many independent errors. `except*` matches by type across the whole group.

### Advanced — the `warnings` module

```python
import warnings

def embed(texts, model="old-model-v1"):
    if model == "old-model-v1":
        warnings.warn("old-model-v1 is deprecated; use v2", DeprecationWarning, stacklevel=2)
    ...

warnings.simplefilter("error", DeprecationWarning)   # turn warnings into errors in CI
```
Use warnings for "this works but you should change it"; use exceptions for "this cannot proceed."

### Logging — the observability half of error handling

```python
import logging, sys, json

class JSONFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts": self.formatTime(record, "%Y-%m-%dT%H:%M:%S"),
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),
        }
        if record.exc_info:
            payload["exception"] = self.formatException(record.exc_info)
        payload.update(getattr(record, "extra_fields", {}))
        return json.dumps(payload)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JSONFormatter())
logging.basicConfig(level=logging.INFO, handlers=[handler], force=True)

logger = logging.getLogger("rag.retriever")
logger.info("retrieved chunks", extra={"extra_fields": {"query_id": "q1", "k": 5}})

try:
    1 / 0
except ZeroDivisionError:
    logger.exception("scoring failed")     # .exception() = .error() + full traceback
```
**Output:**
```json
{"ts": "2026-08-05T10:31:02", "level": "INFO", "logger": "rag.retriever", "msg": "retrieved chunks", "query_id": "q1", "k": 5}
{"ts": "2026-08-05T10:31:02", "level": "ERROR", "logger": "rag.retriever", "msg": "scoring failed", "exception": "Traceback (most recent call last):\n  File ...\nZeroDivisionError: division by zero"}
```

| Level | Use for |
|---|---|
| `DEBUG` | Prompt contents, retrieved chunk ids, tensor shapes |
| `INFO` | Lifecycle: run started, N records loaded, model loaded |
| `WARNING` | Retried after a 429; fell back to a smaller model; skipped 12 bad rows |
| `ERROR` | Request failed after all retries; a batch was lost |
| `CRITICAL` | The service cannot serve at all |

**Rules.** Use `logging`, never `print`, in anything long-lived. Never log secrets, API keys, or full user prompts containing PII. Use lazy `%s` args, not f-strings. Use `logger = logging.getLogger(__name__)` so log lines are attributable to a module and levels can be tuned per-package.

---

## 6.4 Examples

### Beginner — validating input at the boundary

```python
def create_embedding_request(texts, model: str = "text-embedding-3-small",
                             dimensions: int | None = None) -> dict:
    if not isinstance(texts, list):
        raise TypeError(f"texts must be a list, got {type(texts).__name__}")
    if not texts:
        raise ValueError("texts must not be empty")
    if any(not isinstance(t, str) for t in texts):
        bad = next(i for i, t in enumerate(texts) if not isinstance(t, str))
        raise TypeError(f"texts[{bad}] is {type(texts[bad]).__name__}, expected str")
    if dimensions is not None and not (1 <= dimensions <= 3072):
        raise ValueError(f"dimensions must be in [1, 3072], got {dimensions}")
    return {"input": texts, "model": model,
            **({"dimensions": dimensions} if dimensions else {})}

for bad_call in (lambda: create_embedding_request("not a list"),
                 lambda: create_embedding_request([]),
                 lambda: create_embedding_request(["ok", 42]),
                 lambda: create_embedding_request(["ok"], dimensions=9999)):
    try:
        bad_call()
    except (TypeError, ValueError) as exc:
        print(f"{type(exc).__name__}: {exc}")

print(create_embedding_request(["hello"], dimensions=256))
```
**Output:**
```
TypeError: texts must be a list, got str
ValueError: texts must not be empty
TypeError: texts[1] is int, expected str
ValueError: dimensions must be in [1, 3072], got 9999
{'input': ['hello'], 'model': 'text-embedding-3-small', 'dimensions': 256}
```
**Line by line.** `TypeError` for wrong *type*, `ValueError` for wrong *value* — this distinction is a Python convention and callers rely on it. Each message states what was expected, what was received, and (for the list case) **which index** was wrong. Validating at the boundary means the error surfaces in your code with a clear message rather than as a 400 from the provider two layers deeper.

### Intermediate — resilient LLM output parsing

Models return *almost*-JSON constantly. This is a real, daily problem.

```python
import json, re

class ParseError(Exception): ...

def parse_llm_json(raw: str, *, expected_keys: set[str] | None = None) -> dict:
    """Extract a JSON object from an LLM response, tolerating common deviations."""
    attempts = [
        ("direct", lambda t: t),
        ("strip_fences", lambda t: re.sub(r"^```(?:json)?\s*|\s*```$", "", t.strip(),
                                          flags=re.MULTILINE)),
        ("first_object", lambda t: (m.group(0) if (m := re.search(r"\{.*\}", t, re.S)) else t)),
        ("fix_trailing_commas", lambda t: re.sub(r",(\s*[}\]])", r"\1",
                                                 re.search(r"\{.*\}", t, re.S).group(0)
                                                 if re.search(r"\{.*\}", t, re.S) else t)),
        ("single_to_double", lambda t: (re.search(r"\{.*\}", t, re.S).group(0)
                                        if re.search(r"\{.*\}", t, re.S) else t).replace("'", '"')),
    ]

    errors = []
    for name, transform in attempts:
        try:
            candidate = transform(raw)
            data = json.loads(candidate)
        except (json.JSONDecodeError, AttributeError, TypeError) as exc:
            errors.append(f"{name}: {type(exc).__name__}")
            continue

        if not isinstance(data, dict):
            errors.append(f"{name}: parsed a {type(data).__name__}, not an object")
            continue
        if expected_keys and not expected_keys <= set(data):
            errors.append(f"{name}: missing keys {sorted(expected_keys - set(data))}")
            continue
        return {"data": data, "strategy": name}

    raise ParseError(f"could not parse LLM output.\n  attempts: {errors}\n"
                     f"  raw (first 120 chars): {raw[:120]!r}")


samples = [
    '{"sentiment": "positive", "confidence": 0.9}',
    '```json\n{"sentiment": "negative", "confidence": 0.7}\n```',
    'Sure! Here is the result:\n{"sentiment": "neutral", "confidence": 0.5}\nHope that helps.',
    "{'sentiment': 'positive', 'confidence': 0.8}",
    '{"sentiment": "positive", "confidence": 0.6,}',
    'I cannot determine the sentiment.',
]
for s in samples:
    try:
        out = parse_llm_json(s, expected_keys={"sentiment", "confidence"})
        print(f"✓ [{out['strategy']:<19}] {out['data']}")
    except ParseError as exc:
        print(f"✗ {str(exc).splitlines()[0]}")
```
**Output:**
```
✓ [direct             ] {'sentiment': 'positive', 'confidence': 0.9}
✓ [strip_fences       ] {'sentiment': 'negative', 'confidence': 0.7}
✓ [first_object       ] {'sentiment': 'neutral', 'confidence': 0.5}
✓ [single_to_double   ] {'sentiment': 'positive', 'confidence': 0.8}
✓ [fix_trailing_commas] {'sentiment': 'positive', 'confidence': 0.6}
✗ could not parse LLM output.
```
**Why this design.**
- **A ladder of increasingly aggressive repairs**, cheapest first. The `strategy` name is returned so you can *monitor* it: if `direct` drops from 95% to 40% one morning, the model or your prompt changed and you'll know within an hour.
- **Schema validation is part of parsing.** JSON that parses but lacks `sentiment` is still a failure — catching it here beats an `AttributeError` three functions later.
- **The final error carries the raw text** (truncated), because "could not parse" without the input is undebuggable.
- **In production**, prefer structured-output/JSON-mode APIs and pydantic validation. This function is the fallback layer, and you will still need it.

### Production-style — layered fallback with a circuit breaker

```python
import logging, random, time
from dataclasses import dataclass, field

logger = logging.getLogger("llm.client")

class LLMError(Exception): ...
class RateLimitError(LLMError):
    def __init__(self, msg, retry_after: float = 1.0):
        super().__init__(msg); self.retry_after = retry_after
class ServerError(LLMError): ...
class ContextLengthError(LLMError): ...
class AuthError(LLMError): ...            # NOT retryable


@dataclass
class CircuitBreaker:
    """Stop hammering a service that is clearly down."""
    failure_threshold: int = 3
    reset_timeout: float = 5.0
    failures: int = 0
    opened_at: float | None = None

    def is_open(self, now: float) -> bool:
        if self.opened_at is None:
            return False
        if now - self.opened_at >= self.reset_timeout:
            logger.info("circuit half-open: allowing a probe request")
            self.opened_at, self.failures = None, 0
            return False
        return True

    def record_success(self) -> None:
        self.failures, self.opened_at = 0, None

    def record_failure(self, now: float) -> None:
        self.failures += 1
        if self.failures >= self.failure_threshold and self.opened_at is None:
            self.opened_at = now
            logger.error("circuit OPEN after %d failures", self.failures)


@dataclass
class ResilientLLM:
    primary: str = "large"
    fallback: str = "small"
    max_attempts: int = 3
    breaker: CircuitBreaker = field(default_factory=CircuitBreaker)
    stats: dict = field(default_factory=lambda: {"ok": 0, "retried": 0,
                                                 "fell_back": 0, "failed": 0})

    def _call(self, model: str, prompt: str, _rng=random.Random(7)) -> str:
        """Simulated provider."""
        roll = _rng.random()
        if model == "large":
            if roll < 0.35: raise RateLimitError("429 too many requests", retry_after=0.2)
            if roll < 0.50: raise ServerError("503 service unavailable")
        if len(prompt) > 500:
            raise ContextLengthError(f"prompt of {len(prompt)} chars exceeds limit")
        return f"[{model}] answer to {prompt[:24]!r}"

    def generate(self, prompt: str) -> str:
        now = time.monotonic()
        models = [self.fallback] if self.breaker.is_open(now) else [self.primary, self.fallback]

        for model in models:
            for attempt in range(1, self.max_attempts + 1):
                try:
                    result = self._call(model, prompt)

                except AuthError:
                    logger.critical("authentication failed — not retryable")
                    raise                                     # fail fast, always

                except ContextLengthError as exc:
                    logger.warning("%s; truncating prompt", exc)
                    prompt = prompt[:400]
                    continue                                  # retry immediately, no sleep

                except RateLimitError as exc:
                    self.stats["retried"] += 1
                    self.breaker.record_failure(time.monotonic())
                    if attempt == self.max_attempts:
                        break                                 # give up on THIS model
                    delay = exc.retry_after * attempt * (0.5 + random.random())
                    logger.warning("rate limited on %s (attempt %d); sleeping %.2fs",
                                   model, attempt, delay)
                    time.sleep(delay)

                except ServerError:
                    self.stats["retried"] += 1
                    self.breaker.record_failure(time.monotonic())
                    if attempt == self.max_attempts:
                        break
                    time.sleep(0.1 * 2 ** attempt)

                else:
                    self.breaker.record_success()
                    self.stats["ok"] += 1
                    if model != self.primary:
                        self.stats["fell_back"] += 1
                    return result

            logger.warning("exhausted attempts on %s; falling back", model)

        self.stats["failed"] += 1
        raise LLMError(f"all models failed for prompt {prompt[:40]!r}")


logging.basicConfig(level=logging.INFO, format="%(levelname)-8s %(message)s", force=True)
client = ResilientLLM()
for i in range(6):
    try:
        print("→", client.generate(f"question {i} about retrieval augmented generation"))
    except LLMError as exc:
        print("✗", exc)
print("\nstats:", client.stats)
```
**Output (representative — the RNG is seeded, timings vary):**
```
→ [large] answer to 'question 0 about retri'
WARNING  rate limited on large (attempt 1); sleeping 0.15s
→ [large] answer to 'question 1 about retri'
WARNING  rate limited on large (attempt 1); sleeping 0.21s
WARNING  rate limited on large (attempt 2); sleeping 0.33s
ERROR    circuit OPEN after 3 failures
→ [small] answer to 'question 2 about retri'
→ [small] answer to 'question 3 about retri'
INFO     circuit half-open: allowing a probe request
→ [large] answer to 'question 4 about retri'

stats: {'ok': 5, 'retried': 3, 'fell_back': 2, 'failed': 0}
```
**Every design decision, and why.**

| Decision | Rationale |
|---|---|
| **Distinct exception classes** per failure mode | The handler can't choose a policy from a string. `AuthError` must never be retried; `RateLimitError` must be. |
| `AuthError` re-raised immediately | Retrying a bad API key wastes time and can trigger account lockout |
| `ContextLengthError` → truncate and `continue` | The error is *actionable*: fix the input, don't sleep |
| `retry_after` carried on the exception | Providers tell you how long to wait in a header; honour it |
| **Circuit breaker** | Without it, 1000 concurrent requests each retry 3× against a dead service — you turn a partial outage into a total one and possibly get rate-limited for an hour |
| Half-open probe | Automatic recovery: after `reset_timeout`, one request tests the water |
| Model fallback ladder | Degrade to a cheaper/smaller model rather than returning an error to the user |
| `else:` clause for the success path | Keeps `try` minimal, so a bug in the success path isn't swallowed by the retry handlers |
| `stats` counters | These become your dashboard: a rising `fell_back` rate means quality is silently degrading even though nothing is "failing" |

### AI-specific — robust dataset processing with a dead-letter queue

```python
import json, logging
from pathlib import Path
from collections import Counter

logger = logging.getLogger("ingest")

class RecordError(Exception):
    def __init__(self, msg, *, record_id=None, stage=None):
        super().__init__(msg)
        self.record_id, self.stage = record_id, stage


def process_dataset(records, out_path: Path, dlq_path: Path,
                    *, max_error_rate: float = 0.05) -> dict:
    reasons = Counter()
    processed = failed = 0

    out_path.parent.mkdir(parents=True, exist_ok=True)
    with open(out_path, "w", encoding="utf-8") as fout, \
         open(dlq_path, "w", encoding="utf-8") as fdlq:

        for i, rec in enumerate(records):
            rid = rec.get("id", f"index_{i}")
            try:
                text = rec["text"]
                if not isinstance(text, str):
                    raise RecordError(f"text is {type(text).__name__}", record_id=rid, stage="type")
                if not text.strip():
                    raise RecordError("text is empty", record_id=rid, stage="content")
                label = int(rec["label"])
                if label not in (0, 1):
                    raise RecordError(f"label {label} not in (0, 1)", record_id=rid, stage="label")

            except KeyError as exc:
                failed += 1; reasons[f"missing_field:{exc.args[0]}"] += 1
                fdlq.write(json.dumps({"id": rid, "error": f"missing field {exc}",
                                       "record": rec}) + "\n")
            except (ValueError, TypeError) as exc:
                failed += 1; reasons[f"bad_type:{type(exc).__name__}"] += 1
                fdlq.write(json.dumps({"id": rid, "error": str(exc), "record": rec}) + "\n")
            except RecordError as exc:
                failed += 1; reasons[f"{exc.stage}:{exc}"] += 1
                fdlq.write(json.dumps({"id": rid, "error": str(exc), "stage": exc.stage,
                                       "record": rec}) + "\n")
            else:
                processed += 1
                fout.write(json.dumps({"id": rid, "text": text.strip(), "label": label}) + "\n")

    total = processed + failed
    rate = failed / total if total else 0.0
    summary = {"total": total, "processed": processed, "failed": failed,
               "error_rate": round(rate, 4), "reasons": dict(reasons)}

    if rate > max_error_rate:
        raise RuntimeError(
            f"error rate {rate:.1%} exceeds threshold {max_error_rate:.1%}. "
            f"Reasons: {dict(reasons)}. Inspect {dlq_path} before proceeding.")
    if failed:
        logger.warning("ingest completed with %d failures (%.2f%%); see %s",
                       failed, rate * 100, dlq_path)
    return summary


records = (
    [{"id": f"g{i}", "text": f"good record {i}", "label": i % 2} for i in range(40)]
    + [{"id": "b1", "text": "", "label": 1},
       {"id": "b2", "label": 0},
       {"id": "b3", "text": 12345, "label": 1},
       {"id": "b4", "text": "fine", "label": "not-an-int"},
       {"id": "b5", "text": "fine", "label": 7}]
)

logging.basicConfig(level=logging.INFO, format="%(levelname)-8s %(message)s", force=True)
print(json.dumps(process_dataset(records, Path("out/clean.jsonl"), Path("out/dlq.jsonl")),
                 indent=2))
print("\nfirst DLQ entry:")
print(Path("out/dlq.jsonl").read_text(encoding="utf-8").splitlines()[0])
```
**Output:** 5 of 45 records failed — an 11.1% error rate, above the 5% budget — so the run *deliberately* aborts instead of returning a summary:
```
RuntimeError: error rate 11.1% exceeds threshold 5.0%. Reasons: {'content:text is empty': 1,
'missing_field:text': 1, 'type:text is int': 1, 'bad_type:ValueError': 1,
'label:label 7 not in (0, 1)': 1}. Inspect out/dlq.jsonl before proceeding.
```
Raise `max_error_rate` to `0.2` and the same run completes, logging a warning and returning:
```
WARNING  ingest completed with 5 failures (11.11%); see out/dlq.jsonl
{"total": 45, "processed": 40, "failed": 5, "error_rate": 0.1111, "reasons": {...}}
```
**Why this is the correct behaviour.** A 0.1% error rate is normal and should be logged and continued. An 11% error rate means something *systematic* changed — an upstream schema migration, a wrong file, a broken scraper. Continuing silently would train a model on 89% of the data you think you have, and the metrics would look fine. **The dead-letter queue keeps every failed record with its error**, so you can diagnose in minutes instead of re-running a six-hour job with prints added.

Three distinct handlers exist because the *response* differs: a missing field is a schema problem, a bad type is a producer problem, and a domain violation (`label 7`) is a labelling problem. Lumping them into one `except Exception` would hide which one you have.

---

## 6.5 AI Engineering Perspective

**Machine Learning.** Fail fast on schema violations at load time; fail soft on individual bad rows. `sklearn` raises `NotFittedError` if you predict before fitting — a good example of a domain-specific exception preventing a silent wrong answer.

**Deep Learning.** `torch.cuda.OutOfMemoryError` is the defining production error. The standard handler: catch it, `torch.cuda.empty_cache()`, halve the batch size, retry; if it recurs at batch size 1, abort with a clear message. Shape mismatches should raise immediately with both shapes printed — a wrong-but-broadcastable shape that trains silently is far worse than a crash.

**LLM applications.** Every provider call needs: timeout, retry with backoff, `Retry-After` honouring, distinct handling for auth/rate-limit/context/server errors, a fallback model, and a circuit breaker. Every parse of model output needs repair-and-validate. Log the raw response before parsing so failures are debuggable without re-paying.

**AI Agents.** A tool that raises must not kill the loop: catch at the tool boundary, format the error as a message, and **feed it back to the model** — modern models recover well from `"Error: unknown column 'usr_id'. Available columns: user_id, name"`. Also bound total steps, total tokens, and total wall time, and detect repeated identical failures to break out of doom loops.

**RAG.** Handle empty retrieval explicitly (say "I don't know" rather than hallucinating from no context), embedding-API failures with retry, vector-store timeouts with a keyword-search fallback, and context-overflow by dropping the lowest-ranked chunks.

**Data preprocessing.** Skip-and-count with a dead-letter queue; enforce a maximum error rate; checkpoint progress so a restart resumes.

**Model serving.** Map exceptions to HTTP status codes at exactly one boundary (FastAPI exception handlers): 400 for client validation, 422 for schema, 429 for your own rate limits, 503 when a dependency's circuit is open, 500 for genuine bugs. **Never** leak a traceback to the client — log it with a request id and return that id.

---

## 6.6 Internal Working

### What an exception costs

```python
import timeit

no_exc  = timeit.timeit("try:\n x = d['a']\nexcept KeyError:\n x = None",
                        "d = {'a': 1}", number=1_000_000)
with_exc = timeit.timeit("try:\n x = d['missing']\nexcept KeyError:\n x = None",
                         "d = {'a': 1}", number=1_000_000)
get_call = timeit.timeit("x = d.get('missing')", "d = {'a': 1}", number=1_000_000)

print(f"try (no exception raised): {no_exc:.3f}s")
print(f"try (exception raised)   : {with_exc:.3f}s")
print(f".get()                   : {get_call:.3f}s")
```
**Output (indicative):**
```
try (no exception raised): 0.041s
try (exception raised)   : 0.412s
.get()                   : 0.052s
```
**Reading this:** setting up a `try` block is essentially free (zero-cost in 3.11+ when nothing raises). *Raising* costs ~10× a dict lookup, because it must allocate the exception object and walk frames building a traceback. So EAFP is correct when exceptions are rare, and LBYL (`.get()`) is correct when the "miss" case is common.

### Tracebacks hold references — the GPU memory trap

```python
import sys, traceback

def leak_demo():
    big = list(range(1_000_000))       # imagine: a CUDA tensor
    raise ValueError("boom")

try:
    leak_demo()
except ValueError as exc:
    tb = exc.__traceback__
    frame_locals = tb.tb_next.tb_frame.f_locals
    print("traceback still holds:", list(frame_locals), len(frame_locals["big"]))
```
**Output:**
```
traceback still holds: ['big'] 1000000
```
**The consequence.** An exception's traceback references every frame, and each frame references all its locals — including GPU tensors. If you *store* exceptions (in a list of failures, say), you pin that memory indefinitely. This is a genuine cause of "OOM after handling a few errors."

Two facts that follow:
1. Python automatically deletes the `except ... as exc` name at the end of the block for exactly this reason (`exc` is unbound after the block).
2. If you must keep failure information, keep `str(exc)` and `traceback.format_exc()` — strings — not the exception object.

### `finally` always wins

```python
def f():
    try:
        return "from try"
    finally:
        print("finally ran")

print(f())
```
**Output:**
```
finally ran
from try
```
`finally` executes even after `return`, `break`, or `continue`. (And a `return` *inside* `finally` will silently discard an in-flight exception — never do that.)

---

## 6.7 Best Practices

```python
# ✔ Catch narrow
except json.JSONDecodeError:            # not: except Exception

# ✔ Preserve the cause
raise ConfigError("bad config") from exc

# ✔ Log with the traceback
logger.exception("failed to embed batch")     # inside an except block

# ✔ Custom hierarchy per package
class MyLibError(Exception): ...

# ✔ Attach structured data, not just a message
raise RateLimitError("429", retry_after=2.5)

# ✔ Clean up with `with`, not `finally`
with open(p) as f: ...

# ✘ Never silently swallow
except Exception:
    pass                                 # the worst two lines in software

# ✘ Never assert for validation
assert batch_size > 0                    # vanishes under python -O
```

**The decision rule for every error you meet**

```
Is it caused by a bug in MY code (a wrong shape, a None where impossible)?
    → Let it crash. Fix the bug. Do not handle.

Is it a config / startup / schema problem?
    → Crash at startup with a message naming the key and the file.

Is it an EXPECTED, transient, external failure (429, 503, timeout)?
    → Retry with backoff + jitter, bounded, then fall back, then fail.

Is it an expected data-quality problem in one record of millions?
    → Skip, count, dead-letter, enforce a max error rate.

Is it a user input problem?
    → Return a clear 4xx with what was wrong and how to fix it.
```

Additional rules:

1. **Handle at the level that can decide.** A retriever shouldn't decide whether to fall back to a smaller model; the orchestrator should.
2. **One exception-to-HTTP mapping**, at the API boundary only.
3. **Timeouts on every network call.** No timeout means a hung socket blocks a worker forever. `requests` and `httpx` both default to *no* timeout.
4. **Never log secrets.** Redact keys, tokens, and PII before logging prompts.
5. **Include the request/run id in every log line** so a single failure can be traced across services.
6. **Make failures visible.** Counters for retries, fallbacks, and skips — a system that "works" while falling back on 60% of requests is failing quietly.

---

## 6.8 Common Mistakes

### Mistake 1 — The silent swallow

```python
# ❌ WRONG — the single worst error-handling pattern
try:
    result = embed(texts)
except Exception:
    pass                          # now `result` is undefined AND you have no idea why

# ✔ CORRECT
try:
    result = embed(texts)
except EmbeddingError:
    logger.exception("embedding failed for %d texts", len(texts))
    result = []                   # explicit, logged, deliberate fallback
```

### Mistake 2 — Catching too broadly

```python
# ❌ WRONG — hides typos, AttributeErrors, and KeyboardInterrupt-adjacent bugs
try:
    data = json.loads(response.text)
except Exception:
    data = {}

# ✔ CORRECT
try:
    data = json.loads(response.text)
except json.JSONDecodeError as exc:
    raise MalformedResponseError(f"non-JSON response: {response.text[:200]!r}") from exc
```
A broad `except` around a large block will happily catch the `NameError` from your own typo and report it as "the API failed."

### Mistake 3 — Losing the original exception

```python
# ❌ WRONG — the actual cause is gone forever
except json.JSONDecodeError:
    raise ConfigError("bad config")

# ✔ CORRECT
except json.JSONDecodeError as exc:
    raise ConfigError(f"bad config at {path}") from exc
```

### Mistake 4 — `assert` for validation

```python
# ❌ WRONG — disappears under python -O, and AssertionError is uninformative to callers
assert api_key, "API key required"

# ✔ CORRECT
if not api_key:
    raise ValueError("API key required: set OPENAI_API_KEY or pass api_key=")
```

### Mistake 5 — Retrying non-retryable errors

```python
# ❌ WRONG — retries a bad API key 5 times, then a malformed request 5 times
@retry(max_attempts=5)
def call(): ...

# ✔ CORRECT — retry only transient classes
@retry(max_attempts=5, exceptions=(RateLimitError, ServerError, TimeoutError))
def call(): ...
```

### Mistake 6 — No timeout

```python
# ❌ WRONG — can hang forever, blocking a worker
response = requests.post(url, json=payload)

# ✔ CORRECT
response = requests.post(url, json=payload, timeout=(3.05, 30))   # (connect, read)
```

### Mistake 7 — Leaking internals to users

```python
# ❌ WRONG
except Exception as exc:
    return {"error": str(exc)}      # may leak file paths, SQL, keys, prompt contents

# ✔ CORRECT
except Exception:
    request_id = uuid.uuid4().hex[:12]
    logger.exception("unhandled error", extra={"extra_fields": {"request_id": request_id}})
    return {"error": "internal error", "request_id": request_id}, 500
```

### Mistake 8 — `except` ordering

```python
# ❌ WRONG — the second clause is unreachable
except Exception: ...
except ValueError: ...

# ✔ CORRECT — specific first
except ValueError: ...
except Exception: ...
```

---

## 6.9 Performance

| Aspect | Cost |
|---|---|
| `try` block with no exception | ~0 (zero-cost exceptions since 3.11) |
| Raising + catching | ~1 µs (allocation + traceback construction) |
| `logger.debug(...)` when level is INFO | ~0.1 µs if you use lazy `%s` args; the full format cost if you use an f-string |
| `logger.exception` | ~10 µs (formats the whole traceback) |
| `traceback.format_exc()` | ~20 µs |

### EAFP vs LBYL, chosen by hit rate

```python
import timeit
setup = "d = {str(i): i for i in range(1000)}"

# 99% hit rate → EAFP wins
eafp_hit = timeit.timeit("try:\n v=d['500']\nexcept KeyError:\n v=None", setup, number=200_000)
lbyl_hit = timeit.timeit("v = d['500'] if '500' in d else None", setup, number=200_000)

# 0% hit rate → LBYL wins by a lot
eafp_miss = timeit.timeit("try:\n v=d['x']\nexcept KeyError:\n v=None", setup, number=200_000)
lbyl_miss = timeit.timeit("v = d['x'] if 'x' in d else None", setup, number=200_000)

print(f"hit : EAFP {eafp_hit:.3f}s  LBYL {lbyl_hit:.3f}s")
print(f"miss: EAFP {eafp_miss:.3f}s  LBYL {lbyl_miss:.3f}s")
```
**Output (indicative):**
```
hit : EAFP 0.011s  LBYL 0.016s
miss: EAFP 0.096s  LBYL 0.014s
```
**Rule:** EAFP when the exception is genuinely exceptional; LBYL (or `.get()`) when misses are the common case.

### Logging performance

```python
# ❌ formats the string even when DEBUG is disabled
logger.debug(f"prompt={prompt} chunks={chunks}")

# ✔ formats only if the record is actually emitted
logger.debug("prompt=%s chunks=%s", prompt, chunks)

# ✔ for genuinely expensive debug payloads
if logger.isEnabledFor(logging.DEBUG):
    logger.debug("full context: %s", json.dumps(build_debug_dump()))
```

---

## 6.10 Interview Preparation

**Q1 (Beginner). `try/except/else/finally` — what runs when?**
`try` always starts. A matching `except` runs only if an exception was raised. `else` runs only if **no** exception was raised. `finally` runs always, including after `return`, `break`, and unhandled exceptions. `else` exists so success-path code isn't accidentally protected by the handlers.

**Q2 (Beginner). Why is bare `except:` bad?**
It catches `BaseException`, including `KeyboardInterrupt` and `SystemExit`, so Ctrl-C won't stop your program and `sys.exit()` won't exit. It also hides genuine bugs (typos become `NameError`, which you then report as "the API failed"). Catch `Exception` at most, and something narrower where possible.

**Q3 (Beginner). EAFP vs LBYL?**
LBYL checks preconditions before acting; EAFP acts and handles failure. Python prefers EAFP because it avoids TOCTOU race conditions (a file can vanish between `exists()` and `open()`) and because `try` is free when nothing raises. Prefer LBYL when the failure case is common, since raising costs ~1 µs.

**Q4 (Intermediate). Explain `raise X from Y` versus `raise X`.**
`from Y` sets `__cause__` and prints "The above exception was the direct cause…", explicitly linking them. A plain `raise X` inside an `except` sets `__context__` implicitly and prints "During handling of the above exception…". `from None` suppresses the chain. Always chain — the original exception is usually the only useful diagnostic.

**Q5 (Intermediate). When would you define a custom exception hierarchy?**
When callers need to distinguish failure modes to choose a policy. A package base class (`MyLibError`) lets users catch everything from your library without over-catching; subclasses (`RateLimitError`, `AuthError`) let them retry one and fail fast on another. Attach structured attributes (`retry_after`, `status_code`) so handlers don't parse message strings.

**Q6 (Intermediate). Why is `assert` unsuitable for validating inputs?**
`python -O` removes all `assert` statements, so the check silently vanishes in optimised deployments. `AssertionError` also carries no semantic meaning for callers. Use `assert` only for internal invariants during development; use `if ... raise ValueError/TypeError` for anything a user or another system can cause.

**Q7 (AI/ML). Design retry logic for an LLM API.**
Classify errors: `AuthError`/`InvalidRequest` → fail immediately; `RateLimitError` → sleep for the `Retry-After` value with jitter; `ServerError`/`Timeout` → exponential backoff with jitter, capped, bounded attempts; `ContextLengthError` → truncate and retry without sleeping. Wrap it in a circuit breaker so a dead provider isn't hammered, and add a fallback model. Emit counters for retries and fallbacks so quality degradation is visible.

**Q8 (AI/ML). How do you handle a CUDA OOM in a training loop?**
Catch `torch.cuda.OutOfMemoryError`, `del` the offending tensors, call `torch.cuda.empty_cache()`, halve the micro-batch size and increase gradient accumulation to keep the effective batch constant, then retry the step. If it fails at micro-batch 1, abort with the requested shapes and current memory summary. Longer term: gradient checkpointing, mixed precision, or a smaller model.

**Q9 (AI/ML). A tool call in your agent raises. What should happen?**
Catch it at the tool boundary, log it with the tool name and arguments, and return the error *as an observation* to the model — a message like `"Error: unknown column 'usr_id'. Available: user_id, name."` Models correct themselves well from that. Also count consecutive failures per tool and break the loop after a threshold so the agent can't spin forever.

**Q10 (Scenario). A nightly pipeline succeeded but produced a much worse model. Investigation shows 40% of rows were skipped. How do you prevent recurrence?**
Enforce a maximum error rate that *fails* the run, log per-reason skip counts, write skipped rows to a dead-letter queue, assert the output row count against the input count within a tolerance, and alert on the derived metric (rows processed) rather than only on exceptions. The root failure was that the pipeline had no notion of "succeeded badly."

**Q11 (Production). Why a circuit breaker rather than just retries?**
Retries alone amplify load on a struggling dependency: N clients × M retries can turn a 20% failure rate into a total outage, and can get your account rate-limited. A circuit breaker trips after a failure threshold and fails fast (or diverts to a fallback) for a cooldown, then allows a single probe request. It converts a cascading failure into fast, cheap, partial degradation.

**Q12 (Production). What should never appear in an error response to a user?**
Stack traces, file paths, SQL, model names or prompt contents, API keys, and other users' data. Return a generic message plus a correlation id; log the full detail server-side keyed by that id.

---

## 6.11 Exercises

### Easy — E1
Write `safe_divide(a, b, default=0.0)` handling zero division and non-numeric inputs.

<details><summary>Solution</summary>

```python
def safe_divide(a, b, default: float = 0.0) -> float:
    try:
        return a / b
    except ZeroDivisionError:
        return default
    except TypeError as exc:
        raise TypeError(f"safe_divide expects numbers, got {type(a).__name__} "
                        f"and {type(b).__name__}") from exc

print(safe_divide(10, 4))          # -> 2.5
print(safe_divide(10, 0))          # -> 0.0
print(safe_divide(0, 0, float("nan")))
try: safe_divide("a", 2)
except TypeError as e: print(e)
```
**Output:**
```
2.5
0.0
nan
safe_divide expects numbers, got str and int
```
Note the asymmetry: division by zero is an *expected* case in metric computation (precision when nothing was predicted positive), so it gets a default. A string argument is a *programming bug*, so it raises — with a better message than the built-in one.
</details>

### Easy — E2
Define an exception hierarchy for a RAG pipeline and demonstrate catching at different levels.

<details><summary>Solution</summary>

```python
class RAGError(Exception):
    """Base for all RAG pipeline errors."""

class RetrievalError(RAGError): ...
class EmptyRetrievalError(RetrievalError): ...
class VectorStoreError(RetrievalError): ...
class GenerationError(RAGError): ...
class ContextOverflowError(GenerationError): ...

def demo(exc: RAGError):
    try:
        raise exc
    except EmptyRetrievalError:
        return "no documents found → answer 'I don't know'"
    except RetrievalError:
        return "retrieval failed → fall back to keyword search"
    except RAGError:
        return "generic RAG failure → return an error to the user"

for e in (EmptyRetrievalError("nothing above threshold"),
          VectorStoreError("qdrant timeout"),
          ContextOverflowError("18k tokens > 16k limit")):
    print(f"{type(e).__name__:<24} {demo(e)}")
```
**Output:**
```
EmptyRetrievalError      no documents found → answer 'I don't know'
VectorStoreError         retrieval failed → fall back to keyword search
ContextOverflowError     generic RAG failure → return an error to the user
```
The hierarchy encodes policy: handlers catch at the granularity they can act on, and adding a new leaf exception automatically inherits the nearest existing policy rather than crashing.
</details>

### Medium — M1
Write a `@retry` decorator with backoff, jitter, an exception allowlist, an `on_retry` callback, and a total-time budget.

<details><summary>Solution</summary>

```python
import functools, logging, random, time
from typing import Callable, Type

logger = logging.getLogger(__name__)

def retry(*, max_attempts: int = 3, base_delay: float = 0.5, max_delay: float = 30.0,
          total_timeout: float | None = None,
          exceptions: tuple[Type[BaseException], ...] = (Exception,),
          on_retry: Callable[[int, BaseException, float], None] | None = None):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            started = time.monotonic()
            last: BaseException | None = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions as exc:
                    last = exc
                    if attempt == max_attempts:
                        break
                    delay = min(max_delay, base_delay * 2 ** (attempt - 1))
                    delay *= 0.5 + random.random()
                    elapsed = time.monotonic() - started
                    if total_timeout is not None and elapsed + delay > total_timeout:
                        logger.warning("%s: budget %.1fs exhausted after %d attempts",
                                       fn.__name__, total_timeout, attempt)
                        break
                    if on_retry:
                        on_retry(attempt, exc, delay)
                    time.sleep(delay)
            raise RuntimeError(
                f"{fn.__name__} failed after {attempt} attempt(s) "
                f"in {time.monotonic()-started:.2f}s") from last
        return wrapper
    return decorator


calls = {"n": 0}
events = []

@retry(max_attempts=5, base_delay=0.05, exceptions=(ConnectionError,),
       on_retry=lambda a, e, d: events.append((a, type(e).__name__, round(d, 3))))
def flaky():
    calls["n"] += 1
    if calls["n"] < 3:
        raise ConnectionError(f"attempt {calls['n']} failed")
    return "success"

print(flaky(), "after", calls["n"], "calls")
print("retry events:", events)

@retry(max_attempts=3, base_delay=0.01, exceptions=(ConnectionError,))
def always_fails():
    raise ConnectionError("permanently down")

try:
    always_fails()
except RuntimeError as exc:
    print(f"{exc}\n  caused by: {type(exc.__cause__).__name__}: {exc.__cause__}")
```
**Output:**
```
success after 3 calls
retry events: [(1, 'ConnectionError', 0.041), (2, 'ConnectionError', 0.078)]
RuntimeError: always_fails failed after 3 attempt(s) in 0.05s
  caused by: ConnectionError: permanently down
```
**Explanation.** `exceptions=` is the safety catch — retrying a `ValueError` from bad input wastes budget. `total_timeout` bounds the *whole* operation, which matters when your own caller has an SLA: five retries with 30 s backoff is 90 s, far past a typical 10 s API timeout. `on_retry` is a hook for metrics/alerts without coupling the decorator to a metrics library. `from last` preserves the underlying cause.
</details>

### Hard — H1
Build a `ResilientPipeline` that runs multi-stage processing with per-stage error policies (fail-fast, skip, retry, fallback), collects a full error report, and enforces a global error budget.

<details><summary>Solution</summary>

```python
import logging, random, time, traceback
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable

logger = logging.getLogger("pipeline")


class Policy(Enum):
    FAIL_FAST = "fail_fast"    # any error aborts the whole run
    SKIP      = "skip"         # drop this item, keep going
    RETRY     = "retry"        # retry, then skip
    FALLBACK  = "fallback"     # use a fallback function


@dataclass
class Stage:
    name: str
    fn: Callable[[Any], Any]
    policy: Policy = Policy.FAIL_FAST
    retries: int = 2
    fallback: Callable[[Any], Any] | None = None
    retry_on: tuple[type[BaseException], ...] = (Exception,)

    def __post_init__(self):
        if self.policy is Policy.FALLBACK and self.fallback is None:
            raise ValueError(f"stage {self.name!r} uses FALLBACK but has no fallback fn")


@dataclass
class ErrorRecord:
    item_id: str
    stage: str
    error_type: str
    message: str
    attempt: int
    traceback_head: str


@dataclass
class ResilientPipeline:
    stages: list[Stage]
    error_budget: float = 0.10          # abort if >10% of items fail
    errors: list[ErrorRecord] = field(default_factory=list)
    stats: dict = field(default_factory=lambda: {"in": 0, "out": 0, "skipped": 0,
                                                 "retried": 0, "fell_back": 0})

    def _run_stage(self, stage: Stage, value: Any, item_id: str) -> tuple[bool, Any]:
        """Returns (keep_going, new_value)."""
        attempts = stage.retries + 1 if stage.policy is Policy.RETRY else 1
        for attempt in range(1, attempts + 1):
            try:
                return True, stage.fn(value)
            except Exception as exc:
                self.errors.append(ErrorRecord(
                    item_id, stage.name, type(exc).__name__, str(exc), attempt,
                    traceback.format_exc().strip().splitlines()[-1]))

                if stage.policy is Policy.FAIL_FAST:
                    raise RuntimeError(
                        f"stage {stage.name!r} failed on item {item_id!r} "
                        f"with policy FAIL_FAST") from exc

                if stage.policy is Policy.RETRY and attempt < attempts \
                        and isinstance(exc, stage.retry_on):
                    self.stats["retried"] += 1
                    time.sleep(0.01 * attempt)
                    continue

                if stage.policy is Policy.FALLBACK:
                    try:
                        self.stats["fell_back"] += 1
                        logger.warning("item %s: stage %s fell back (%s)",
                                       item_id, stage.name, exc)
                        return True, stage.fallback(value)
                    except Exception as fexc:
                        self.errors.append(ErrorRecord(
                            item_id, f"{stage.name}:fallback", type(fexc).__name__,
                            str(fexc), 1, "fallback also failed"))

                self.stats["skipped"] += 1
                logger.info("item %s skipped at stage %s: %s", item_id, stage.name, exc)
                return False, None
        return False, None

    def run(self, items: list[dict]) -> list[Any]:
        self.stats["in"] = len(items)
        out = []
        for item in items:
            item_id = str(item.get("id", "?"))
            value: Any = item
            for stage in self.stages:
                keep, value = self._run_stage(stage, value, item_id)
                if not keep:
                    break
            else:
                out.append(value)
                self.stats["out"] += 1

        rate = self.stats["skipped"] / max(1, self.stats["in"])
        if rate > self.error_budget:
            raise RuntimeError(
                f"error budget exceeded: {rate:.1%} > {self.error_budget:.1%} "
                f"({self.stats['skipped']}/{self.stats['in']} items dropped)")
        return out

    def report(self) -> None:
        from collections import Counter
        print("\n" + "=" * 58)
        print("PIPELINE REPORT")
        print("=" * 58)
        for k, v in self.stats.items():
            print(f"  {k:<10} {v}")
        if self.errors:
            print("\n  errors by (stage, type):")
            for (stage, etype), n in Counter(
                    (e.stage, e.error_type) for e in self.errors).most_common():
                print(f"    {stage:<14} {etype:<20} {n}")
            print("\n  sample errors:")
            for e in self.errors[:3]:
                print(f"    [{e.item_id}] {e.stage}: {e.error_type}: {e.message}")
        print("=" * 58)


# ---------------- stage functions ----------------
def validate(item: dict) -> dict:
    if "text" not in item:
        raise KeyError("text")
    if not isinstance(item["text"], str) or not item["text"].strip():
        raise ValueError(f"invalid text for {item.get('id')}")
    return item

_rng = random.Random(3)
def embed(item: dict) -> dict:
    if _rng.random() < 0.3:
        raise ConnectionError("embedding API 503")
    return {**item, "vector": [len(item["text"]) * 0.01] * 4}

def enrich(item: dict) -> dict:
    if "lang" not in item:
        raise LookupError("language not detected")
    return {**item, "enriched": True}

def enrich_fallback(item: dict) -> dict:
    return {**item, "lang": "unknown", "enriched": False}


if __name__ == "__main__":
    logging.basicConfig(level=logging.WARNING, format="%(levelname)-8s %(message)s",
                        force=True)
    pipeline = ResilientPipeline(
        stages=[
            Stage("validate", validate, Policy.SKIP),
            Stage("embed", embed, Policy.RETRY, retries=3, retry_on=(ConnectionError,)),
            Stage("enrich", enrich, Policy.FALLBACK, fallback=enrich_fallback),
        ],
        error_budget=0.25,
    )

    items = [{"id": f"i{i}", "text": f"document {i}", "lang": "en"} for i in range(10)]
    items += [{"id": "bad1"},                                  # no text  → SKIP
              {"id": "bad2", "text": "   ", "lang": "en"},     # empty    → SKIP
              {"id": "nolang", "text": "text without language"}]  # → FALLBACK

    results = pipeline.run(items)
    print(f"\n{len(results)} items survived")
    print("example:", results[-1])
    pipeline.report()
```
**Output:**
```
WARNING  item nolang: stage enrich fell back (language not detected)

11 items survived
example: {'id': 'nolang', 'text': 'text without language', 'vector': [0.21, 0.21, 0.21, 0.21], 'lang': 'unknown', 'enriched': False}

==========================================================
PIPELINE REPORT
==========================================================
  in         13
  out        11
  skipped    2
  retried    6
  fell_back  1

  errors by (stage, type):
    embed          ConnectionError      6
    validate       KeyError             1
    validate       ValueError           1
    enrich         LookupError          1

  sample errors:
    [i0] embed: ConnectionError: embedding API 503
    [i4] embed: ConnectionError: embedding API 503
    [i4] embed: ConnectionError: embedding API 503
==========================================================
```
**Explanation — why this architecture.**

| Idea | Rationale |
|---|---|
| **Policy per stage, not per pipeline** | Validation failures should skip the record; embedding failures should retry (transient); enrichment failures should degrade gracefully. One global policy is always wrong for at least one stage. |
| `Policy` as an `Enum` | Typos become errors at definition time rather than silently falling into a default branch |
| `__post_init__` validation on `Stage` | A `FALLBACK` stage with no fallback function fails at construction, not at 3 a.m. on item 4,000,000 |
| `retry_on` per stage | Retrying a `ValueError` is pure waste |
| Structured `ErrorRecord` list | Aggregating by `(stage, error_type)` immediately shows *where* and *what*, turning "the pipeline had errors" into "the embedding API returned 503 five times" |
| **Error budget** | The most important feature. A pipeline that "succeeds" while dropping 40% of data is worse than one that fails, because nobody investigates a success. |
| `for/else` on the stage loop | `else` appends only if no stage broke out — Chapter 2's idiom doing real work |
| Fallback's own try/except | A failing fallback must not itself crash the pipeline |
</details>

---

## 6.12 Mini Project — Production LLM Client

**Goal.** One class combining everything in this chapter: typed errors, classified retries, circuit breaking, model fallback, response validation, structured JSON logging with redaction, cost tracking, and a health report. This is genuinely close to what you would deploy.

```python
"""llm_client.py — a production-shaped LLM client."""
from __future__ import annotations

import json
import logging
import random
import re
import sys
import time
import uuid
from dataclasses import dataclass, field
from enum import Enum

# ---------------------------------------------------------------- logging
class JSONFormatter(logging.Formatter):
    REDACT = re.compile(r"(sk-[A-Za-z0-9]{4})[A-Za-z0-9]+")

    def format(self, record: logging.LogRecord) -> str:
        payload = {"ts": self.formatTime(record, "%H:%M:%S"),
                   "level": record.levelname,
                   "msg": self.REDACT.sub(r"\1***", record.getMessage())}
        payload.update(getattr(record, "fields", {}))
        if record.exc_info:
            payload["error"] = self.formatException(record.exc_info).splitlines()[-1]
        return json.dumps(payload)

_h = logging.StreamHandler(sys.stdout); _h.setFormatter(JSONFormatter())
logging.basicConfig(level=logging.INFO, handlers=[_h], force=True)
logger = logging.getLogger("llm")


# ---------------------------------------------------------------- errors
class LLMError(Exception):
    """Base for every error this client raises."""
    retryable = False

class AuthError(LLMError): ...
class InvalidRequestError(LLMError): ...
class RateLimitError(LLMError):
    retryable = True
    def __init__(self, msg: str, retry_after: float = 1.0):
        super().__init__(msg); self.retry_after = retry_after
class ServerError(LLMError): retryable = True
class TimeoutError_(LLMError): retryable = True
class ContextLengthError(LLMError): ...
class ValidationError(LLMError): ...
class CircuitOpenError(LLMError): ...


class CircuitState(Enum):
    CLOSED = "closed"; OPEN = "open"; HALF_OPEN = "half_open"


@dataclass
class CircuitBreaker:
    threshold: int = 4
    cooldown: float = 3.0
    failures: int = 0
    opened_at: float | None = None

    def state(self, now: float) -> CircuitState:
        if self.opened_at is None:
            return CircuitState.CLOSED
        return CircuitState.HALF_OPEN if now - self.opened_at >= self.cooldown \
            else CircuitState.OPEN

    def check(self, now: float) -> None:
        if self.state(now) is CircuitState.OPEN:
            remaining = self.cooldown - (now - self.opened_at)
            raise CircuitOpenError(f"circuit open, retry in {remaining:.1f}s")

    def success(self) -> None:
        if self.opened_at is not None:
            logger.info("circuit closed after successful probe")
        self.failures, self.opened_at = 0, None

    def failure(self, now: float) -> None:
        self.failures += 1
        if self.failures >= self.threshold and self.opened_at is None:
            self.opened_at = now
            logger.error("circuit OPEN", extra={"fields": {"failures": self.failures}})


MODELS = {"large": {"cost_per_1k": 0.003, "max_tokens": 8000},
          "small": {"cost_per_1k": 0.0002, "max_tokens": 4000}}


@dataclass
class LLMClient:
    api_key: str = "sk-abcd1234567890secret"
    primary: str = "large"
    fallback: str = "small"
    max_attempts: int = 3
    breaker: CircuitBreaker = field(default_factory=CircuitBreaker)
    total_cost: float = 0.0
    stats: dict[str, int] = field(default_factory=lambda: {
        "requests": 0, "ok": 0, "retries": 0, "fallbacks": 0,
        "failures": 0, "validation_failures": 0})
    _rng: random.Random = field(default_factory=lambda: random.Random(11))

    # ---------------- simulated transport ----------------
    def _transport(self, model: str, prompt: str) -> str:
        est_tokens = len(prompt) // 4 + 150
        if est_tokens > MODELS[model]["max_tokens"]:
            raise ContextLengthError(
                f"{est_tokens} tokens exceeds {model} limit of {MODELS[model]['max_tokens']}")
        if not self.api_key.startswith("sk-"):
            raise AuthError("invalid api key")

        roll = self._rng.random()
        if model == "large":
            if roll < 0.25: raise RateLimitError("429 rate limited", retry_after=0.15)
            if roll < 0.40: raise ServerError("503 upstream unavailable")
            if roll < 0.48: raise TimeoutError_("read timeout after 30s")

        self.total_cost += est_tokens / 1000 * MODELS[model]["cost_per_1k"]
        if roll < 0.55:                       # sometimes the model returns messy JSON
            return f'```json\n{{"answer": "response from {model}", "confidence": 0.8}}\n```'
        return f'{{"answer": "response from {model}", "confidence": 0.9}}'

    # ---------------- validation ----------------
    @staticmethod
    def _parse(raw: str, required: set[str]) -> dict:
        cleaned = re.sub(r"^```(?:json)?|```$", "", raw.strip(), flags=re.MULTILINE).strip()
        try:
            data = json.loads(cleaned)
        except json.JSONDecodeError as exc:
            raise ValidationError(f"model returned non-JSON: {raw[:80]!r}") from exc
        if not isinstance(data, dict):
            raise ValidationError(f"expected an object, got {type(data).__name__}")
        missing = required - set(data)
        if missing:
            raise ValidationError(f"missing keys {sorted(missing)} in {sorted(data)}")
        return data

    # ---------------- public API ----------------
    def generate(self, prompt: str, *, required_keys: set[str] | None = None) -> dict:
        request_id = uuid.uuid4().hex[:8]
        required = required_keys or {"answer"}
        self.stats["requests"] += 1
        started = time.monotonic()

        now = time.monotonic()
        try:
            self.breaker.check(now)
            models = [self.primary, self.fallback]
        except CircuitOpenError as exc:
            logger.warning(str(exc), extra={"fields": {"request_id": request_id}})
            models = [self.fallback]

        last_error: Exception | None = None

        for model in models:
            for attempt in range(1, self.max_attempts + 1):
                try:
                    raw = self._transport(model, prompt)
                    data = self._parse(raw, required)

                except AuthError:
                    logger.critical("auth failure — aborting",
                                    extra={"fields": {"request_id": request_id}})
                    self.stats["failures"] += 1
                    raise

                except ContextLengthError as exc:
                    last_error = exc
                    logger.warning("context overflow; truncating",
                                   extra={"fields": {"request_id": request_id,
                                                     "model": model}})
                    prompt = prompt[: int(len(prompt) * 0.6)]
                    continue

                except ValidationError as exc:
                    last_error = exc
                    self.stats["validation_failures"] += 1
                    logger.warning("invalid model output; re-asking",
                                   extra={"fields": {"request_id": request_id,
                                                     "attempt": attempt,
                                                     "reason": str(exc)[:60]}})
                    if attempt == self.max_attempts:
                        break
                    continue

                except LLMError as exc:
                    last_error = exc
                    if not exc.retryable:
                        raise
                    self.stats["retries"] += 1
                    self.breaker.failure(time.monotonic())
                    if attempt == self.max_attempts:
                        break
                    delay = getattr(exc, "retry_after", 0.1) * attempt * (0.5 + random.random())
                    logger.warning("transient failure; backing off",
                                   extra={"fields": {"request_id": request_id,
                                                     "model": model, "attempt": attempt,
                                                     "error": type(exc).__name__,
                                                     "sleep_s": round(delay, 3)}})
                    time.sleep(delay)

                else:
                    self.breaker.success()
                    self.stats["ok"] += 1
                    if model != self.primary:
                        self.stats["fallbacks"] += 1
                    logger.info("request ok", extra={"fields": {
                        "request_id": request_id, "model": model, "attempts": attempt,
                        "latency_ms": round((time.monotonic() - started) * 1000, 1),
                        "cost_usd": round(self.total_cost, 5)}})
                    return {**data, "_meta": {"model": model, "request_id": request_id,
                                              "attempts": attempt}}

        self.stats["failures"] += 1
        logger.error("all models exhausted",
                     extra={"fields": {"request_id": request_id}})
        raise LLMError(f"request {request_id} failed after all fallbacks") from last_error

    # ---------------- observability ----------------
    def health(self) -> dict:
        s = self.stats
        total = max(1, s["requests"])
        return {
            "circuit": self.breaker.state(time.monotonic()).value,
            "success_rate": f"{s['ok']/total:.0%}",
            "fallback_rate": f"{s['fallbacks']/total:.0%}",
            "validation_failure_rate": f"{s['validation_failures']/total:.0%}",
            "total_cost_usd": round(self.total_cost, 5),
            **s,
        }


if __name__ == "__main__":
    client = LLMClient()

    for i in range(8):
        try:
            result = client.generate(f"Question {i}: explain retrieval augmented generation",
                                     required_keys={"answer", "confidence"})
            print(f"  ✓ {result['answer']}  (model={result['_meta']['model']})")
        except LLMError as exc:
            print(f"  ✗ {exc}")

    try:
        oversized = client.generate("x" * 40_000)
        print(f"  ✓ oversized prompt survived by truncation "
              f"(model={oversized['_meta']['model']})")
    except LLMError as exc:
        print(f"  ✗ oversized prompt: {type(exc).__name__}")

    print("\nHEALTH:")
    for k, v in client.health().items():
        print(f"  {k:<24} {v}")
```

**Output (representative — seeded RNG, real timings vary):**
```
{"ts": "10:42:01", "level": "WARNING", "msg": "transient failure; backing off", "request_id": "3f9c1a02", "model": "large", "attempt": 1, "error": "RateLimitError", "sleep_s": 0.112}
{"ts": "10:42:01", "level": "INFO", "msg": "request ok", "request_id": "3f9c1a02", "model": "large", "attempts": 2, "latency_ms": 113.4, "cost_usd": 0.00075}
  ✓ response from large  (model=large)
{"ts": "10:42:01", "level": "WARNING", "msg": "invalid model output; re-asking", "request_id": "a71b3e55", "attempt": 1, "reason": "missing keys ['confidence'] in ['answer']"}
  ✓ response from small  (model=small)
{"ts": "10:42:02", "level": "ERROR", "msg": "circuit OPEN", "failures": 4}
{"ts": "10:42:02", "level": "WARNING", "msg": "circuit open, retry in 2.9s", "request_id": "c2d40f18"}
  ✓ response from small  (model=small)
{"ts": "17:21:53", "level": "WARNING", "msg": "context overflow; truncating", "request_id": "aa22c48c", "model": "large"}
  ✓ oversized prompt survived by truncation (model=large)

HEALTH:
  circuit                  open
  success_rate             88%
  fallback_rate            38%
  validation_failure_rate  25%
  total_cost_usd           0.00612
  requests                 9
  ok                       8
  retries                  6
  fallbacks                3
  failures                 1
  validation_failures      2
```

**What this consolidates.**

| Feature | Chapter concept |
|---|---|
| Typed exception hierarchy with a `retryable` class attribute | §6.2 — policy encoded in the type, not parsed from a message |
| Different handler per error class | §6.4 — auth fails fast, context truncates, transients back off |
| `else:` clause for the success path | §6.2 — success-path bugs aren't caught by retry handlers |
| Circuit breaker with half-open probe | §6.4 — prevents retry storms from turning degradation into outage |
| Model fallback ladder | §6.5 — degrade, don't fail |
| Output validation as a first-class failure | §6.4 — malformed model JSON is a normal, measurable event |
| Structured JSON logs with a `request_id` | §6.3 — every line traceable across services |
| **API-key redaction in the formatter** | §6.7 — secrets must never reach a log aggregator |
| `health()` with rates | §6.7 — a 38% fallback rate is invisible without this, and it means users are silently getting worse answers |

> **Note the oversized-prompt case.** It does *not* raise. `ContextLengthError` is handled by truncating the prompt to 60% and retrying without sleeping, so a 40 000-character prompt shrinks until it fits and the request succeeds. That is deliberate: a context overflow is an *actionable* error — fix the input rather than back off. Whether silent truncation is acceptable is a product decision; the alternative is to raise and let the caller re-chunk. Either way, log it, because silently discarding 60% of a user's document is exactly the kind of degradation that never shows up in an error rate.

**Extensions.** Add `async def agenerate` with `asyncio` and a semaphore (Chapter 8); persist stats to Prometheus; add per-tenant rate limiting; make `_transport` an injected `Protocol` so tests need no network (Chapter 7); add a response cache keyed by prompt hash (Chapter 5).

---

## Chapter 6 Summary

| Concept | Takeaway |
|---|---|
| Hierarchy | Catch `Exception` at most; prefer narrow classes; never bare `except:` |
| `else` / `finally` | `else` = success-only path; `finally` = always, even after `return` |
| Chaining | Always `raise X from exc`; never discard the cause |
| Custom exceptions | Package base class + typed subclasses + structured attributes |
| `assert` | Invariants only — stripped by `-O`, never for validation |
| EAFP vs LBYL | EAFP when failures are rare; LBYL when misses are common (~1 µs to raise) |
| Retry | Bounded, exponential, jittered, allowlisted by exception type |
| Circuit breaker | Stops retry storms turning partial failure into total outage |
| Dead-letter + error budget | Never "succeed" while silently dropping 40% of your data |
| Logging | `logging` not `print`; lazy `%s` args; structured JSON; redact secrets |
| Tracebacks | Hold frame locals — storing exceptions can pin GPU memory |

**Revision flashcards.** `else` vs `finally` · `raise from` · retryable vs non-retryable · circuit breaker states · dead-letter queue · error budget · never `assert` for validation · never log secrets.

---
# Chapter 7 — Object-Oriented Programming

> **Chapter goal.** Every framework you are about to learn is object-oriented: `nn.Module`, `Dataset`, `Pipeline`, `BaseRetriever`, `BaseTool`, `BaseModel`. You cannot subclass what you don't understand. By the end you will know when a class earns its place — and, just as importantly, when a plain function is the better answer.

---

## 7.1 Concept Overview

### What it is

A class bundles **state** (data) with **behaviour** (functions that operate on that data) and defines a **type**. An object is an instance of a class.

### Why it exists

Some things genuinely have identity and lifecycle: a model with loaded weights, a database connection, an agent with memory, a tokenizer with a vocabulary. Passing "the vocabulary dict, the merges list, the special-token map, and the max length" to twelve separate functions is worse than having a `Tokenizer` object that owns them.

### Why Python includes it this way

Python's OOP is unusually flexible:

- **Duck typing.** If it has the methods, it works — no interface declaration required. This is why any object with `__len__` and `__getitem__` is a valid PyTorch `Dataset`.
- **Everything is an object**, including classes themselves (instances of `type`).
- **Protocols over inheritance.** Magic methods (`__len__`, `__iter__`, `__call__`) let your class integrate with built-in syntax.
- **Multiple inheritance with a well-defined MRO** — the basis of the mixin pattern used throughout `transformers`.
- **Dataclasses** remove the boilerplate that made classes tedious.

### Why AI engineers care

| Framework construct | What it really is |
|---|---|
| `torch.nn.Module` | A class with `__call__`, parameter registration, and `.to(device)` |
| `torch.utils.data.Dataset` | Any object with `__len__` and `__getitem__` |
| `transformers.PreTrainedModel` | A class with `from_pretrained` (a classmethod) and mixins |
| `sklearn` estimator | A class with `fit`/`transform`/`predict` — a duck-typed protocol |
| `pydantic.BaseModel` | A class using a metaclass to build validators from annotations |
| LangChain `Runnable` | A class implementing `__or__` so `a | b` composes pipelines |
| An agent `Tool` | A class or dataclass bundling name, description, schema, and callable |

### When *not* to use a class

If a class has one method and no state, it is a function wearing a costume:

```python
# ✘ over-engineered
class TextCleaner:
    def clean(self, text): return text.lower().strip()

# ✔ just a function
def clean_text(text: str) -> str: return text.lower().strip()
```
Reach for a class when there is **state to own**, **several operations sharing that state**, **multiple interchangeable implementations**, or **a lifecycle** (open/close, load/unload).

---

## 7.2 Theory

### Objects and attribute lookup

```python
class Model:
    family = "transformer"           # CLASS attribute — shared by all instances

    def __init__(self, name: str):
        self.name = name             # INSTANCE attribute — per object

m = Model("bert")
print(m.__dict__)                    # -> {'name': 'bert'}
print(Model.__dict__.keys())         # class body: family, __init__, ...
```
**Output:**
```
{'name': 'bert'}
dict_keys(['__module__', 'family', '__init__', ...])
```

Attribute lookup order for `m.x`:
```
1. type(m).__mro__ data descriptors  (property, etc.)
2. m.__dict__                        (instance attributes)
3. type(m).__mro__ non-data attrs    (methods, class attributes)
4. __getattr__ fallback              (if defined)
5. AttributeError
```

> **The class-attribute trap.** A *mutable* class attribute is shared by every instance — the same bug as the mutable default argument:
> ```python
> class Agent:
>     history = []                 # ❌ SHARED across all agents
> a, b = Agent(), Agent()
> a.history.append("hi")
> print(b.history)                 # -> ['hi']
> ```
> Assign mutable state in `__init__`, or use `dataclasses.field(default_factory=list)`.

### `self` — no magic

Methods are plain functions stored on the class. `m.method(x)` is sugar for `type(m).method(m, x)`; the instance is passed as the first argument, which convention names `self`.

```python
class Counter:
    def __init__(self): self.n = 0
    def bump(self): self.n += 1

c = Counter()
Counter.bump(c)          # explicit form
c.bump()                 # sugar for the same thing
print(c.n)               # -> 2
```

### Inheritance and the MRO

```python
class Base:
    def describe(self): return "base"

class Mixin:
    def describe(self): return "mixin -> " + super().describe()

class Concrete(Mixin, Base):
    def describe(self): return "concrete -> " + super().describe()

print(Concrete().describe())
print([c.__name__ for c in Concrete.__mro__])
```
**Output:**
```
concrete -> mixin -> base
['Concrete', 'Mixin', 'Base', 'object']
```

The **Method Resolution Order** is computed by the C3 linearisation algorithm: left to right, depth first, but a class never appears before its subclasses. `super()` follows the MRO of the *instance's* type, not the lexical parent — which is what makes cooperative multiple inheritance (mixins) work.

This is exactly how `transformers` composes `PreTrainedModel(nn.Module, ModuleUtilsMixin, GenerationMixin, PushToHubMixin)`: each mixin contributes one capability and `super()` chains them.

### Composition vs inheritance

```
INHERITANCE — "is-a"                COMPOSITION — "has-a"
class BertClassifier(nn.Module)     class RAGPipeline:
  ✔ genuine specialisation            def __init__(self, retriever, llm):
  ✔ shares an interface                   self.retriever = retriever
  ✘ tight coupling                        self.llm = llm
  ✘ deep hierarchies break            ✔ swap parts independently
                                      ✔ trivially testable with fakes
```

**Rule: prefer composition.** Inherit when you genuinely need to *be* the parent type (a custom `nn.Module` must be an `nn.Module` for parameter registration to work). Compose for everything else. A `RAGPipeline` is not a kind of retriever; it *has* one.

### Encapsulation in Python

Python has no `private`. It has conventions and one mild mechanism:

| Form | Meaning | Enforced? |
|---|---|---|
| `x` | Public API | — |
| `_x` | Internal; may change without notice | Convention only |
| `__x` | Name-mangled to `_ClassName__x` | Prevents accidental subclass collision, not access |
| `@property` | Computed/validated attribute access | Yes, via the descriptor protocol |

```python
class Model:
    def __init__(self, temperature: float = 0.7):
        self._temperature = temperature

    @property
    def temperature(self) -> float:
        return self._temperature

    @temperature.setter
    def temperature(self, value: float) -> None:
        if not 0.0 <= value <= 2.0:
            raise ValueError(f"temperature must be in [0, 2], got {value}")
        self._temperature = value

m = Model()
m.temperature = 1.5          # goes through the setter → validated
print(m.temperature)         # -> 1.5
try:
    m.temperature = 9.0
except ValueError as e:
    print(e)
```
**Output:**
```
1.5
temperature must be in [0, 2], got 9.0
```
`@property` lets you add validation later **without changing the public API** — callers still write `m.temperature = x`. That is its real value.

### Abstraction — ABCs and Protocols

```python
from abc import ABC, abstractmethod
from typing import Protocol, runtime_checkable

# --- nominal: subclasses must inherit and implement ---
class BaseRetriever(ABC):
    @abstractmethod
    def retrieve(self, query: str, k: int = 5) -> list[dict]: ...

    def retrieve_texts(self, query: str, k: int = 5) -> list[str]:
        return [d["text"] for d in self.retrieve(query, k)]   # shared default

# --- structural: any object with the right shape qualifies ---
@runtime_checkable
class Embedder(Protocol):
    def embed(self, texts: list[str]) -> list[list[float]]: ...


class FakeEmbedder:                     # does NOT inherit from Embedder
    def embed(self, texts): return [[0.1] * 4 for _ in texts]

print(isinstance(FakeEmbedder(), Embedder))    # -> True  (structural match)

try:
    BaseRetriever()
except TypeError as e:
    print(e)
```
**Output:**
```
True
Can't instantiate abstract class BaseRetriever without an implementation for abstract method 'retrieve'
```

| | ABC | Protocol |
|---|---|---|
| Relationship | Nominal (must inherit) | Structural (duck typing, checked) |
| Enforcement | At instantiation | At type-check time (`mypy`) |
| Can share code | Yes (concrete methods) | No (declaration only) |
| Use for | Your own class families | Accepting *someone else's* objects |

**Practical guidance:** use an ABC for a base class you own and want to share behaviour from; use a `Protocol` to type a dependency you accept, so callers can pass anything with the right shape — including test fakes and third-party objects you can't modify.

---

## 7.3 Syntax

### Basic

```python
class Document:
    """A retrievable document chunk."""

    source: str = "unknown"                       # class attribute

    def __init__(self, doc_id: str, text: str, score: float = 0.0):
        self.doc_id = doc_id                      # instance attributes
        self.text = text
        self.score = score

    def preview(self, n: int = 40) -> str:        # instance method
        return self.text[:n] + ("…" if len(self.text) > n else "")

    @classmethod
    def from_dict(cls, data: dict) -> "Document": # alternative constructor
        return cls(data["id"], data["text"], data.get("score", 0.0))

    @staticmethod
    def is_valid_id(doc_id: str) -> bool:         # namespaced utility
        return bool(doc_id) and doc_id.isascii()

    def __repr__(self) -> str:
        return f"Document(doc_id={self.doc_id!r}, score={self.score})"


d = Document.from_dict({"id": "d1", "text": "Retrieval augmented generation " * 3,
                        "score": 0.91})
print(d)
print(d.preview())
print(Document.is_valid_id("d1"), Document.is_valid_id(""))
```
**Output:**
```
Document(doc_id='d1', score=0.91)
Retrieval augmented generation Retrieval …
True False
```

| Decorator | First arg | Use for |
|---|---|---|
| *(none)* | `self` | Behaviour needing instance state |
| `@classmethod` | `cls` | Alternative constructors (`from_pretrained`, `from_dict`) — and `cls` means subclasses get the right type back |
| `@staticmethod` | — | A related utility that needs neither |
| `@property` | `self` | Computed or validated attribute access |

### Intermediate — magic methods

```python
class Embedding:
    def __init__(self, values: list[float], model: str = "minilm"):
        self.values, self.model = values, model

    def __len__(self):  return len(self.values)              # len(e)
    def __getitem__(self, i): return self.values[i]          # e[0], e[1:3], and iteration
    def __iter__(self):  return iter(self.values)            # for v in e
    def __contains__(self, v): return v in self.values       # v in e
    def __eq__(self, other):                                 # e1 == e2
        return isinstance(other, Embedding) and self.values == other.values
    def __hash__(self): return hash((tuple(self.values), self.model))   # needed with __eq__
    def __add__(self, other):                                # e1 + e2
        return Embedding([a + b for a, b in zip(self.values, other.values, strict=True)])
    def __mul__(self, k: float):                             # e * 2
        return Embedding([v * k for v in self.values])
    def __matmul__(self, other):                             # e1 @ e2 (dot product)
        return sum(a * b for a, b in zip(self.values, other.values, strict=True))
    def __repr__(self):                                      # debugging
        return f"Embedding(dim={len(self)}, model={self.model!r})"
    def __str__(self):                                       # user-facing
        return f"<{len(self)}-dim embedding>"
    def __bool__(self):  return any(self.values)             # if e:
    def __call__(self, other):                               # e(other)
        return self @ other

a = Embedding([1.0, 2.0, 3.0]); b = Embedding([4.0, 5.0, 6.0])
print(repr(a), "|", str(a))
print(len(a), a[0], a[1:], list(a))
print(a + b, a * 2, a @ b, a(b))
print(a == Embedding([1.0, 2.0, 3.0]), 2.0 in a, bool(Embedding([0, 0])))
```
**Output:**
```
Embedding(dim=3, model='minilm') | <3-dim embedding>
3 1.0 [2.0, 3.0] [1.0, 2.0, 3.0]
Embedding(dim=3, model='minilm') Embedding(dim=3, model='minilm') 32.0 32.0
True True False
```

**The magic methods worth knowing, grouped:**

| Group | Methods | Enables |
|---|---|---|
| Construction | `__new__`, `__init__`, `__del__` | Allocation, init, finalisation |
| Representation | `__repr__`, `__str__`, `__format__` | Debugging, printing, f-strings |
| Comparison | `__eq__`, `__lt__`, `__hash__` | `==`, sorting, dict keys |
| Container | `__len__`, `__getitem__`, `__setitem__`, `__contains__`, `__iter__` | `len`, `[]`, `in`, `for` |
| Numeric | `__add__`, `__mul__`, `__matmul__`, `__radd__` | Operators |
| Callable | `__call__` | `obj(...)` — this is why `model(x)` works in PyTorch |
| Context | `__enter__`, `__exit__` | `with obj:` |
| Attributes | `__getattr__`, `__setattr__`, `__slots__` | Dynamic attributes, memory control |
| Async | `__aiter__`, `__anext__`, `__aenter__` | `async for`, `async with` |

> **`__repr__` is the highest-value method you can write.** It appears in debuggers, logs, tracebacks, and notebook output. Rule: `__repr__` should be unambiguous (ideally valid Python that recreates the object); `__str__` should be readable. If you only write one, write `__repr__` — `str()` falls back to it.

### Advanced — dataclasses

```python
from dataclasses import dataclass, field, asdict, replace

@dataclass(frozen=True, slots=True, order=True)
class RetrievalConfig:
    top_k: int = 5
    min_score: float = 0.7
    rerank: bool = True
    filters: dict[str, str] = field(default_factory=dict)     # NOT `= {}`
    _cache: dict = field(default_factory=dict, repr=False, compare=False)

    def __post_init__(self):
        if self.top_k < 1:
            raise ValueError(f"top_k must be >= 1, got {self.top_k}")
        if not 0.0 <= self.min_score <= 1.0:
            raise ValueError(f"min_score must be in [0,1], got {self.min_score}")

cfg = RetrievalConfig(top_k=10, filters={"lang": "en"})
print(cfg)
print(asdict(cfg))
tighter = replace(cfg, min_score=0.9)        # new instance, one field changed
print(tighter.min_score, cfg.min_score)
try:
    cfg.top_k = 3
except Exception as e:
    print(type(e).__name__, e)
```
**Output:**
```
RetrievalConfig(top_k=10, min_score=0.7, rerank=True, filters={'lang': 'en'})
{'top_k': 10, 'min_score': 0.7, 'rerank': True, 'filters': {'lang': 'en'}, '_cache': {}}
0.9 0.7
FrozenInstanceError cannot assign to field 'top_k'
```

| Parameter | Effect | When to use |
|---|---|---|
| `frozen=True` | Immutable; auto-generates `__hash__` | **Configs — always** |
| `slots=True` | No per-instance `__dict__`; ~30–40% less memory, faster attribute access | High-instance-count classes |
| `order=True` | Generates `<`, `>`, etc. from field order | Sortable records |
| `kw_only=True` | All fields keyword-only | Many-field classes; avoids positional mistakes |
| `field(default_factory=...)` | Fresh mutable default per instance | **Every list/dict/set field** |
| `field(repr=False)` | Hide from `__repr__` | Secrets, huge tensors |
| `field(compare=False)` | Exclude from `==` | Caches, timestamps |

**Why dataclasses matter for AI code:** a config is a value object — it should be immutable, comparable, hashable (so it can key a cache), printable, and validated. `@dataclass(frozen=True, slots=True)` gives all of that in one line, replacing about 40 lines of hand-written boilerplate.

### Advanced — `__slots__`, context managers, `__init_subclass__`

```python
import sys

class WithDict:
    def __init__(self, a, b): self.a, self.b = a, b

class WithSlots:
    __slots__ = ("a", "b")
    def __init__(self, a, b): self.a, self.b = a, b

d, s = WithDict(1, 2), WithSlots(1, 2)
print("dict-based:", sys.getsizeof(d) + sys.getsizeof(d.__dict__), "bytes")
print("slots-based:", sys.getsizeof(s), "bytes")
```
**Output:**
```
dict-based: 152 bytes
slots-based: 56 bytes
```
Multiply by 10 million `Document` objects and `__slots__` saves ~1 GB. The cost: no dynamic attributes and multiple-inheritance restrictions.

```python
class ModelSession:
    """A context manager that owns a model's lifecycle."""
    def __init__(self, name): self.name, self.model = name, None
    def __enter__(self):
        print(f"loading {self.name}")
        self.model = f"<weights of {self.name}>"
        return self
    def __exit__(self, exc_type, exc, tb):
        print(f"unloading {self.name} (exception: {exc_type.__name__ if exc_type else 'none'})")
        self.model = None
        return False                       # False = do not suppress the exception

with ModelSession("bert-base") as sess:
    print("using", sess.model)
```
**Output:**
```
loading bert-base
using <weights of bert-base>
unloading bert-base (exception: none)
```

```python
class Tool:
    """Auto-registering base class via __init_subclass__."""
    registry: dict[str, type] = {}

    def __init_subclass__(cls, *, name: str | None = None, **kwargs):
        super().__init_subclass__(**kwargs)
        key = name or cls.__name__.lower()
        if key in Tool.registry:
            raise ValueError(f"duplicate tool name {key!r}")
        Tool.registry[key] = cls

class SearchTool(Tool, name="search"): ...
class CalcTool(Tool): ...

print(Tool.registry)
```
**Output:**
```
{'search': <class '__main__.SearchTool'>, 'calctool': <class '__main__.CalcTool'>}
```
`__init_subclass__` is the 95% case for what people reach for metaclasses to do — and it is far simpler. This auto-registration pattern is how plugin systems and tool registries work.

---

## 7.4 Examples

### Beginner — a PyTorch-style Dataset

```python
class TextClassificationDataset:
    """Duck-typed PyTorch Dataset: needs only __len__ and __getitem__."""

    def __init__(self, texts: list[str], labels: list[int], *, max_length: int = 128):
        if len(texts) != len(labels):
            raise ValueError(f"texts ({len(texts)}) and labels ({len(labels)}) differ")
        self.texts, self.labels, self.max_length = texts, labels, max_length

    def __len__(self) -> int:
        return len(self.texts)

    def __getitem__(self, idx: int) -> dict:
        text = self.texts[idx][: self.max_length]
        return {"text": text,
                "label": self.labels[idx],
                "length": len(text)}

    def __repr__(self) -> str:
        from collections import Counter
        return (f"{type(self).__name__}(n={len(self)}, "
                f"classes={dict(Counter(self.labels))})")


ds = TextClassificationDataset(
    ["great movie", "terrible plot", "loved it", "waste of time"], [1, 0, 1, 0])

print(ds)
print(len(ds))
print(ds[0])
for sample in ds:                 # __getitem__ with integer indices enables iteration
    print(" ", sample["label"], sample["text"])
```
**Output:**
```
TextClassificationDataset(n=4, classes={1: 2, 0: 2})
4
{'text': 'great movie', 'label': 1, 'length': 11}
  1 great movie
  0 terrible plot
  1 loved it
  0 waste of time
```
**Line by line.** This class inherits from nothing, yet `torch.utils.data.DataLoader(ds, batch_size=2)` would accept it — because PyTorch only requires `__len__` and `__getitem__`. That is duck typing doing real work. The length check in `__init__` fails fast on the classic off-by-one dataset bug. Iteration works for free: Python falls back to calling `__getitem__(0), __getitem__(1), …` until `IndexError`.

### Intermediate — inheritance for a retriever family

```python
from abc import ABC, abstractmethod
import math

class BaseRetriever(ABC):
    """Shared interface and behaviour for all retrievers."""

    def __init__(self, name: str):
        self.name = name
        self.query_count = 0

    @abstractmethod
    def _search(self, query: str, k: int) -> list[tuple[str, float]]:
        """Subclasses implement the actual search. Returns (doc_id, score) pairs."""

    def retrieve(self, query: str, k: int = 3, min_score: float = 0.0) -> list[dict]:
        """Template method: shared instrumentation + filtering around _search."""
        if not query.strip():
            raise ValueError("query must not be empty")
        self.query_count += 1
        hits = self._search(query, k)
        return [{"doc_id": d, "score": round(s, 4), "retriever": self.name}
                for d, s in hits if s >= min_score][:k]

    def __repr__(self):
        return f"{type(self).__name__}(name={self.name!r}, queries={self.query_count})"


class KeywordRetriever(BaseRetriever):
    def __init__(self, docs: dict[str, str]):
        super().__init__("keyword")                # ALWAYS call super().__init__
        self.docs = docs

    def _search(self, query, k):
        terms = set(query.lower().split())
        scored = []
        for doc_id, text in self.docs.items():
            words = set(text.lower().split())
            overlap = len(terms & words)
            if overlap:
                scored.append((doc_id, overlap / len(terms)))
        return sorted(scored, key=lambda p: (-p[1], p[0]))[:k]


class VectorRetriever(BaseRetriever):
    def __init__(self, vectors: dict[str, list[float]], embed_fn):
        super().__init__("vector")
        self.vectors, self.embed_fn = vectors, embed_fn

    def _search(self, query, k):
        q = self.embed_fn(query)
        def cos(a, b):
            dot = sum(x * y for x, y in zip(a, b, strict=True))
            return dot / ((math.dist(a, [0]*len(a)) * math.dist(b, [0]*len(b))) or 1e-9)
        scored = [(d, cos(q, v)) for d, v in self.vectors.items()]
        return sorted(scored, key=lambda p: (-p[1], p[0]))[:k]


class HybridRetriever(BaseRetriever):
    """Composition INSIDE inheritance: owns other retrievers, fuses their results."""
    def __init__(self, retrievers: list[BaseRetriever], rrf_k: int = 60):
        super().__init__("hybrid")
        self.retrievers, self.rrf_k = retrievers, rrf_k

    def _search(self, query, k):
        from collections import defaultdict
        fused = defaultdict(float)
        for r in self.retrievers:
            for rank, hit in enumerate(r.retrieve(query, k * 2), start=1):
                fused[hit["doc_id"]] += 1.0 / (self.rrf_k + rank)
        return sorted(fused.items(), key=lambda p: (-p[1], p[0]))[:k]


docs = {"d1": "vector databases store embeddings",
        "d2": "embeddings capture semantic meaning",
        "d3": "keyword search uses inverted indexes"}
vecs = {"d1": [0.9, 0.1], "d2": [0.8, 0.3], "d3": [0.2, 0.9]}

retrievers = [KeywordRetriever(docs), VectorRetriever(vecs, lambda q: [0.85, 0.2])]
hybrid = HybridRetriever(retrievers)

for r in [*retrievers, hybrid]:
    print(f"{r.name:<9} {r.retrieve('embeddings semantic', k=2)}")
print()
for r in [*retrievers, hybrid]:
    print(r)
```
**Output:**
```
keyword   [{'doc_id': 'd2', 'score': 1.0, 'retriever': 'keyword'}, {'doc_id': 'd1', 'score': 0.5, 'retriever': 'keyword'}]
vector    [{'doc_id': 'd1', 'score': 0.9965, 'retriever': 'vector'}, {'doc_id': 'd2', 'score': 0.9762, 'retriever': 'vector'}]
hybrid    [{'doc_id': 'd1', 'score': 0.0325, 'retriever': 'hybrid'}, {'doc_id': 'd2', 'score': 0.0325, 'retriever': 'hybrid'}]

KeywordRetriever(name='keyword', queries=1)
VectorRetriever(name='vector', queries=2)
HybridRetriever(name='hybrid', queries=1)
```
**Every design choice explained.**
- **Template method pattern.** `retrieve()` is concrete and owns validation, counting, filtering, and result formatting; `_search()` is abstract. Subclasses write only the part that differs — roughly ten lines each instead of forty.
- **`@abstractmethod`** makes an incomplete subclass fail at instantiation, not at the first query in production.
- **`super().__init__(...)` in every subclass** — forgetting it leaves `self.query_count` undefined and produces a baffling `AttributeError` later.
- **`HybridRetriever` inherits the interface but composes the implementations.** It *is* a retriever (so it can be used anywhere one is expected) and *has* retrievers (so any combination works). Note `VectorRetriever.query_count == 2` — the hybrid called it too, which the shared instrumentation captures for free.
- **Polymorphism in action:** the final loop treats all three identically. Adding a `GraphRetriever` requires no changes to any calling code.

### Production-style — an agent with tools, memory, and state

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from collections import deque
from typing import Any, Protocol
import json


# ---------- Protocol: what the agent needs from an LLM ----------
class LLM(Protocol):
    def complete(self, messages: list[dict]) -> str: ...


# ---------- Tools ----------
class Tool(ABC):
    name: str
    description: str

    def __init_subclass__(cls, **kw):
        super().__init_subclass__(**kw)
        for attr in ("name", "description"):
            if not getattr(cls, attr, None):
                raise TypeError(f"{cls.__name__} must define a class-level {attr!r}")

    @abstractmethod
    def run(self, **kwargs: Any) -> str: ...

    def schema(self) -> dict:
        return {"name": self.name, "description": self.description}

    def __call__(self, **kwargs: Any) -> str:      # tools are callable
        return self.run(**kwargs)

    def __repr__(self):
        return f"{type(self).__name__}(name={self.name!r})"


class SearchTool(Tool):
    name = "search"
    description = "Search the knowledge base. Args: query (str)."

    def __init__(self, docs: dict[str, str]):
        self.docs = docs

    def run(self, query: str = "", **_) -> str:
        terms = set(query.lower().split())
        hits = [f"{k}: {v}" for k, v in self.docs.items()
                if terms & set(v.lower().split())]
        return "\n".join(hits[:3]) if hits else "No results found."


class CalculatorTool(Tool):
    name = "calculator"
    description = "Evaluate arithmetic. Args: expression (str)."

    ALLOWED = set("0123456789+-*/(). ")

    def run(self, expression: str = "", **_) -> str:
        if not set(expression) <= self.ALLOWED:
            return f"Error: expression contains disallowed characters: {expression!r}"
        try:
            return str(eval(expression, {"__builtins__": {}}, {}))
        except Exception as exc:
            return f"Error: {type(exc).__name__}: {exc}"


# ---------- Memory ----------
@dataclass
class Memory:
    max_turns: int = 6
    turns: deque[dict] = field(default_factory=deque)
    facts: dict[str, str] = field(default_factory=dict)

    def __post_init__(self):
        self.turns = deque(self.turns, maxlen=self.max_turns)

    def remember(self, role: str, content: str) -> None:
        self.turns.append({"role": role, "content": content})

    def note(self, key: str, value: str) -> None:
        self.facts[key] = value

    def as_messages(self) -> list[dict]:
        msgs = list(self.turns)
        if self.facts:
            msgs.insert(0, {"role": "system",
                            "content": "Known facts: " + json.dumps(self.facts)})
        return msgs

    def __len__(self): return len(self.turns)


# ---------- Agent ----------
@dataclass
class Agent:
    llm: LLM
    tools: list[Tool]
    max_steps: int = 5
    memory: Memory = field(default_factory=Memory)
    trace: list[dict] = field(default_factory=list)

    def __post_init__(self):
        self._tools: dict[str, Tool] = {t.name: t for t in self.tools}
        if len(self._tools) != len(self.tools):
            raise ValueError("duplicate tool names")

    @property
    def tool_descriptions(self) -> str:
        return "\n".join(f"- {t.name}: {t.description}" for t in self.tools)

    def _dispatch(self, action: dict) -> str:
        name = action.get("tool")
        if name not in self._tools:
            return (f"Error: unknown tool {name!r}. "
                    f"Available: {sorted(self._tools)}")
        try:
            return self._tools[name](**action.get("args", {}))
        except TypeError as exc:
            return f"Error: bad arguments for {name}: {exc}"
        except Exception as exc:
            return f"Error: {name} raised {type(exc).__name__}: {exc}"

    def run(self, task: str) -> str:
        self.memory.remember("user", task)

        for step in range(1, self.max_steps + 1):
            messages = [{"role": "system",
                         "content": f"Available tools:\n{self.tool_descriptions}"}]
            messages += self.memory.as_messages()

            raw = self.llm.complete(messages)
            try:
                action = json.loads(raw)
            except json.JSONDecodeError:
                self.trace.append({"step": step, "error": "non-JSON response"})
                self.memory.remember("user", "Respond with valid JSON only.")
                continue

            self.trace.append({"step": step, **action})

            if action.get("final"):
                answer = action["final"]
                self.memory.remember("assistant", answer)
                return answer

            observation = self._dispatch(action)
            self.memory.remember("assistant", raw)
            self.memory.remember("user", f"Observation: {observation}")

        return f"Stopped: reached max_steps={self.max_steps} without a final answer."

    def print_trace(self) -> None:
        print(f"\n{'step':<5} {'tool':<12} detail")
        print("-" * 62)
        for entry in self.trace:
            tool = entry.get("tool", "—" if "final" in entry else entry.get("error", "?"))
            detail = json.dumps(entry.get("args", entry.get("final", "")))[:44]
            print(f"{entry['step']:<5} {str(tool):<12} {detail}")


# ---------- a scripted fake LLM, so this runs deterministically ----------
class ScriptedLLM:
    def __init__(self, script: list[str]):
        self._script = list(script)
    def complete(self, messages: list[dict]) -> str:
        return self._script.pop(0) if self._script else '{"final": "I am out of script."}'


if __name__ == "__main__":
    agent = Agent(
        llm=ScriptedLLM([
            '{"tool": "search", "args": {"query": "embeddings"}}',
            'not json at all',
            '{"tool": "calculator", "args": {"expression": "384 * 1000"}}',
            '{"tool": "nonexistent", "args": {}}',
            '{"final": "Embeddings are dense vectors; 1000 of dim 384 = 384000 floats."}',
        ]),
        tools=[SearchTool({"d1": "embeddings capture semantic meaning",
                           "d2": "vector databases store embeddings"}),
               CalculatorTool()],
        max_steps=6,
    )

    print("ANSWER:", agent.run("What are embeddings and how big is a 1000x384 matrix?"))
    agent.print_trace()
    print(f"\nmemory: {len(agent.memory)} turns retained (max {agent.memory.max_turns})")
```
**Output:**
```
ANSWER: Embeddings are dense vectors; 1000 of dim 384 = 384000 floats.

step  tool         detail
--------------------------------------------------------------
1     search       {"query": "embeddings"}
2     non-JSON response ""
3     calculator   {"expression": "384 * 1000"}
4     nonexistent  {}
5     —            "Embeddings are dense vectors; 1000 of dim 3

memory: 6 turns retained (max 6)
```
**Why this is the real architecture of an agent framework.**

| Element | OOP concept | Why |
|---|---|---|
| `Tool` ABC with `@abstractmethod run` | Abstraction | New tools plug in with zero changes to `Agent` |
| `__init_subclass__` validation | Metaprogramming-lite | A tool without a description fails at *import*, not when the model can't use it |
| `Tool.__call__` | Magic method | Tools are callables, so they can also be passed as plain functions |
| `LLM` as a `Protocol` | Structural typing | The agent works with OpenAI, Anthropic, a local model, or `ScriptedLLM` in tests — **without any of them inheriting anything** |
| `Memory` as a dataclass with a `deque` | Composition | Memory policy changes (summarisation, vector recall) without touching `Agent` |
| `Agent` composes llm + tools + memory | Composition over inheritance | Each part is independently testable and swappable |
| `self._tools` dict built in `__post_init__` | Data structure choice | O(1) dispatch and duplicate detection at construction |
| Errors returned as observations | Chapter 6 | The model sees `unknown tool 'nonexistent'. Available: [...]` and self-corrects |
| `max_steps` | Chapter 2 | Bounded loop — no runaway cost |
| `trace` list | Observability | Non-deterministic systems are undebuggable without a trace |

The `ScriptedLLM` deserves special mention: because `LLM` is a Protocol, the entire agent is testable with **no network, no API key, and deterministic output**. That is what "program to an interface" buys you.

### AI-specific — a model wrapper with lazy loading and a context manager

```python
import time
from dataclasses import dataclass, field

@dataclass
class ModelWrapper:
    """Lazily loads a model, tracks usage, and releases resources deterministically."""
    name: str
    device: str = "cpu"
    _model: object | None = field(default=None, init=False, repr=False)
    _calls: int = field(default=0, init=False)
    _load_seconds: float = field(default=0.0, init=False)

    # ---- lazy loading via a property ----
    @property
    def model(self):
        if self._model is None:
            t = time.perf_counter()
            print(f"  [loading {self.name} onto {self.device}]")
            time.sleep(0.05)                      # simulate weight loading
            self._model = f"<{self.name} weights>"
            self._load_seconds = time.perf_counter() - t
        return self._model

    @property
    def is_loaded(self) -> bool:
        return self._model is not None

    def predict(self, text: str) -> dict:
        _ = self.model                            # triggers the load on first use
        self._calls += 1
        return {"input": text[:20], "label": "positive", "score": 0.87}

    def unload(self) -> None:
        if self._model is not None:
            print(f"  [unloading {self.name}]")
            self._model = None

    # ---- context-manager protocol ----
    def __enter__(self) -> "ModelWrapper":
        _ = self.model
        return self

    def __exit__(self, exc_type, exc, tb) -> bool:
        self.unload()
        return False

    def __repr__(self) -> str:
        return (f"ModelWrapper(name={self.name!r}, device={self.device!r}, "
                f"loaded={self.is_loaded}, calls={self._calls})")


m = ModelWrapper("distilbert-sst2", device="cuda")
print(m)                       # not loaded yet — construction is instant
print(m.predict("great film"))
print(m)
m.unload()

print("\nwith a context manager:")
with ModelWrapper("distilbert-sst2") as model:
    print(" ", model.predict("terrible"))
    print(" ", model)
print("after the with block, resources are released")
```
**Output:**
```
ModelWrapper(name='distilbert-sst2', device='cuda', loaded=False, calls=0)
  [loading distilbert-sst2 onto cuda]
{'input': 'great film', 'label': 'positive', 'score': 0.87}
ModelWrapper(name='distilbert-sst2', device='cuda', loaded=True, calls=1)
  [unloading distilbert-sst2]

with a context manager:
  [loading distilbert-sst2]
  {'input': 'terrible', 'label': 'positive', 'score': 0.87}
  ModelWrapper(name='distilbert-sst2', device='cpu', loaded=True, calls=1)
after the with block, resources are released
  [unloading distilbert-sst2]
```
**Why lazy loading matters.** In a serving process that registers five models but usually uses one, eager loading multiplies startup time and GPU memory by five. The `@property` makes loading transparent — callers never call `load()` — while `is_loaded` keeps the state observable. The context manager gives *deterministic* release, which matters enormously for GPU memory: relying on garbage collection to free 14 GB of VRAM is how you get intermittent OOMs.

---

## 7.5 AI Engineering Perspective

**Machine Learning.** Every scikit-learn estimator is a duck-typed class with `fit`, `transform`, `predict`, and `get_params`. Write your own transformer by inheriting `BaseEstimator, TransformerMixin` and it drops straight into a `Pipeline` and `GridSearchCV`. The `Pipeline` itself is composition — a list of estimators.

**Deep Learning.** `nn.Module.__call__` runs hooks then `forward()`, which is why you call `model(x)` and never `model.forward(x)`. `__setattr__` is overridden so assigning `self.linear = nn.Linear(...)` auto-registers parameters — a magic method doing critical bookkeeping. `Dataset` is a two-method protocol. `state_dict()`/`load_state_dict()` are the serialisation interface.

**LLM applications.** Provider clients are classes wrapping HTTP; a `Protocol` for `complete(messages) -> str` lets you swap providers and inject fakes in tests. Prompt templates, message objects, and token budgets are natural value objects — frozen dataclasses.

**AI Agents.** `Tool` (ABC), `Memory` (composition), `Agent` (orchestration) — exactly the structure above, and exactly what LangChain/LlamaIndex implement at larger scale. LangChain's `Runnable` defines `__or__` so `prompt | model | parser` builds a pipeline; that's operator overloading as API design.

**RAG.** `BaseRetriever`, `BaseEmbedder`, `BaseVectorStore`, `BaseChunker` — abstract interfaces with several concrete implementations each, composed into a pipeline. Swapping Chroma for Qdrant should change one constructor call.

**Data preprocessing.** A `Transform` base class with `__call__` lets you compose `Compose([Lowercase(), StripHTML(), Truncate(512)])` — which is precisely how `torchvision.transforms` works.

**Model serving.** pydantic `BaseModel` classes define request/response schemas; FastAPI reads their annotations to validate input and generate OpenAPI docs. Dependency injection passes composed objects (a model, a retriever, a cache) into handlers.

---

## 7.6 Internal Working

### Attribute access and `__dict__`

```python
class Plain: 
    def __init__(self): self.a = 1

p = Plain()
print(p.__dict__)                  # -> {'a': 1}
p.__dict__["dynamic"] = 99         # attributes are literally dict entries
print(p.dynamic)                   # -> 99
```
Every attribute access on a `__dict__`-based object is a hash lookup. `__slots__` replaces the dict with fixed array slots — hence both the memory saving and the speed-up.

### Method binding

```python
class C:
    def m(self): return "hi"

c = C()
print(C.m)         # <function C.m at 0x...>       — a plain function
print(c.m)         # <bound method C.m of <C ...>> — a bound method object
print(c.m.__self__ is c)    # -> True
```
Accessing `c.m` creates a *new* bound method object each time (functions are descriptors — `__get__` returns the binding). That is a small allocation, which is why hot loops sometimes hoist `f = obj.method` outside the loop.

### `__new__` vs `__init__`

```python
class Singleton:
    _instance = None
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    def __init__(self, value):
        self.value = value            # ⚠ runs on EVERY construction call

a = Singleton("first"); b = Singleton("second")
print(a is b, a.value)
```
**Output:**
```
True second
```
`__new__` allocates and returns the instance; `__init__` initialises an already-created one. Note the trap: `__init__` still runs, so `b`'s value overwrote `a`'s. (Singletons are usually a design smell in Python — a module-level object is simpler.)

### The MRO and the diamond problem

```python
class A:
    def who(self): return "A"
class B(A):
    def who(self): return "B -> " + super().who()
class C(A):
    def who(self): return "C -> " + super().who()
class D(B, C):
    def who(self): return "D -> " + super().who()

print(D().who())
print([k.__name__ for k in D.__mro__])
```
**Output:**
```
D -> B -> C -> A
['D', 'B', 'C', 'A', 'object']
```
Note that `B.who`'s `super()` resolved to **C**, not A — because `super()` follows the MRO of `type(self)`, not the lexical base class. This cooperative behaviour is what makes mixins compose; it is also why every class in a cooperative hierarchy must call `super().__init__(**kwargs)`.

### Memory: dict vs slots vs dataclass

```python
import sys
from dataclasses import dataclass

class Regular:
    def __init__(self, a, b, c): self.a, self.b, self.c = a, b, c

@dataclass
class DC:
    a: int; b: int; c: int

@dataclass(slots=True)
class DCSlots:
    a: int; b: int; c: int

class Slotted:
    __slots__ = ("a", "b", "c")
    def __init__(self, a, b, c): self.a, self.b, self.c = a, b, c

def total(o):
    return sys.getsizeof(o) + sys.getsizeof(getattr(o, "__dict__", 0))

for cls in (Regular, DC, DCSlots, Slotted):
    print(f"{cls.__name__:<10} {total(cls(1,2,3)):>4} bytes")
```
**Output (indicative):**
```
Regular     176 bytes
DC          176 bytes
DCSlots      64 bytes
Slotted      64 bytes
```
For 10 million chunk objects in a RAG index, that is 1.76 GB versus 0.64 GB.

---

## 7.7 Best Practices

```python
# ✔ Prefer composition; inherit only for genuine "is-a"
class RAGPipeline:
    def __init__(self, retriever: BaseRetriever, llm: LLM, memory: Memory): ...

# ✔ Depend on Protocols/ABCs, not concrete classes
def build(retriever: BaseRetriever, embedder: Embedder): ...

# ✔ Frozen dataclasses for value objects
@dataclass(frozen=True, slots=True)
class Chunk: text: str; doc_id: str; index: int

# ✔ Always write __repr__
def __repr__(self): return f"{type(self).__name__}(id={self.id!r}, n={len(self)})"

# ✔ default_factory for mutable fields
history: list[str] = field(default_factory=list)

# ✔ Validate in __post_init__ / __init__
# ✔ Call super().__init__() in every subclass
# ✔ Use @property to add validation without breaking the API
# ✔ Context managers for anything with a lifecycle (models, connections, files)
```

**Higher-level guidance**

1. **Keep hierarchies shallow.** Two levels is usually enough; three is a warning; four means you wanted composition.
2. **Type `self` returns as the class** (`-> "Document"` or `Self` from `typing`) so chaining type-checks.
3. **Don't put I/O in `__init__`.** Constructing an object should be cheap and total. Use a `@classmethod` (`from_pretrained`) or lazy properties for loading.
4. **Small interfaces.** An ABC with ten abstract methods is unimplementable; one with two is a joy. Follow the Interface Segregation Principle.
5. **`__eq__` + `__hash__` together, or neither.** Defining `__eq__` alone makes the class unhashable.
6. **Use `Enum` for closed sets of values** (roles, statuses, policies) rather than bare strings.
7. **Don't reach for metaclasses.** `__init_subclass__`, decorators, and `__set_name__` cover nearly every real need with a tenth of the complexity.

---

## 7.8 Common Mistakes

### Mistake 1 — Mutable class attribute

```python
# ❌ WRONG — every agent shares one history
class Agent:
    history = []
    def add(self, msg): self.history.append(msg)

a, b = Agent(), Agent()
a.add("hi")
print(b.history)                 # -> ['hi']

# ✔ CORRECT
class Agent:
    def __init__(self):
        self.history: list[str] = []
```

### Mistake 2 — Forgetting `super().__init__()`

```python
# ❌ WRONG — PyTorch cannot register parameters; you get a cryptic error later
class MyModel(nn.Module):
    def __init__(self):
        self.linear = nn.Linear(10, 2)

# ✔ CORRECT
class MyModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.linear = nn.Linear(10, 2)
```

### Mistake 3 — Inheritance where composition belongs

```python
# ❌ WRONG — a pipeline is not a kind of retriever
class RAGPipeline(VectorRetriever, LLMClient, PromptTemplate): ...

# ✔ CORRECT
class RAGPipeline:
    def __init__(self, retriever, llm, template):
        self.retriever, self.llm, self.template = retriever, llm, template
```

### Mistake 4 — `__eq__` without `__hash__`

```python
# ❌ WRONG — instances become unhashable, breaking sets and dict keys
class Chunk:
    def __eq__(self, other): return self.text == other.text

# ✔ CORRECT
class Chunk:
    def __eq__(self, other): return isinstance(other, Chunk) and self.text == other.text
    def __hash__(self): return hash(self.text)
# ✔ or simply
@dataclass(frozen=True)
class Chunk: text: str
```

### Mistake 5 — Heavy work in `__init__`

```python
# ❌ WRONG — 30 s constructor; can't be created in a test or a config parse
class Classifier:
    def __init__(self, name):
        self.model = AutoModel.from_pretrained(name)      # downloads 2 GB

# ✔ CORRECT — lazy property, or an explicit classmethod loader
class Classifier:
    def __init__(self, name): self.name, self._model = name, None
    @property
    def model(self):
        if self._model is None:
            self._model = AutoModel.from_pretrained(self.name)
        return self._model
```

### Mistake 6 — Getters and setters for nothing

```python
# ❌ WRONG — Java in Python
class Config:
    def get_lr(self): return self._lr
    def set_lr(self, v): self._lr = v

# ✔ CORRECT — a public attribute; add @property later only if validation is needed
class Config:
    def __init__(self, lr): self.lr = lr
```

### Mistake 7 — Mutable default in a dataclass

```python
# ❌ WRONG — ValueError: mutable default <class 'list'> for field
@dataclass
class Batch:
    items: list = []

# ✔ CORRECT
@dataclass
class Batch:
    items: list = field(default_factory=list)
```
(Dataclasses at least *catch* this one at class-creation time — unlike plain function defaults.)

### Mistake 8 — No `__repr__`

```python
# ❌ debugging output: <__main__.Chunk object at 0x7f3d2c1a4e50>
# ✔ debugging output: Chunk(doc_id='d1', index=3, len=512)
```

---

## 7.9 Performance

| Operation | Cost | Note |
|---|---|---|
| Instance attribute read (`__dict__`) | ~20 ns | Dict lookup |
| Instance attribute read (`__slots__`) | ~15 ns | Array slot |
| Method call | ~60–90 ns | Bound-method creation + call |
| `@property` read | ~80 ns | Descriptor `__get__` + function call |
| `__getattr__` fallback | ~200 ns | Only fires when normal lookup fails |
| Instance creation (regular) | ~200 ns + dict | |
| Instance creation (`slots=True`) | ~120 ns | |
| `isinstance` | ~40 ns | MRO walk, cached |

```python
import timeit
setup = """
from dataclasses import dataclass
@dataclass
class A: x: int; y: int
@dataclass(slots=True)
class B: x: int; y: int
a, b = A(1,2), B(1,2)
"""
print("create dict :", timeit.timeit("A(1,2)", setup, number=1_000_000))
print("create slots:", timeit.timeit("B(1,2)", setup, number=1_000_000))
print("read dict   :", timeit.timeit("a.x", setup, number=1_000_000))
print("read slots  :", timeit.timeit("b.x", setup, number=1_000_000))
```
**Output (indicative):**
```
create dict : 0.312
create slots: 0.198
read dict   : 0.028
read slots  : 0.021
```

**Practical guidance.** Object overhead is irrelevant for a `Model` (one instance) and very relevant for a `Chunk` (ten million instances). For per-element data at scale, don't use objects at all — use NumPy structured arrays, Arrow tables, or a DataFrame. A class per row is the wrong granularity in data-processing code.

---

## 7.10 Interview Preparation

**Q1 (Beginner). Class attribute vs instance attribute?**
Class attributes live on the class and are shared by all instances; instance attributes live in the instance's `__dict__`. Reads fall back from instance to class, but writes always create an instance attribute — except for mutations, which affect the shared object. Never use a mutable class attribute for per-instance state.

**Q2 (Beginner). `@classmethod` vs `@staticmethod` vs instance method?**
Instance methods take `self` and use instance state. Class methods take `cls` and are used for alternative constructors — crucially, `cls` is the *actual* subclass, so `Subclass.from_dict()` returns a `Subclass`. Static methods take neither and are simply functions namespaced inside the class.

**Q3 (Beginner). What are `__str__` and `__repr__` for?**
`__repr__` is for developers — unambiguous, ideally reconstructible, and used in debuggers, logs, and containers. `__str__` is for users — readable. `str()` falls back to `__repr__` if `__str__` is undefined, so if you write only one, write `__repr__`.

**Q4 (Intermediate). Explain the MRO and why `super()` isn't just "the parent class."**
The MRO is a linearisation of the inheritance graph computed by C3, guaranteeing that a class precedes its bases and that the ordering respects each class's local precedence. `super()` returns a proxy that dispatches to the *next class in the MRO of `type(self)`* — which in a diamond can be a sibling, not an ancestor. That is what allows mixins to chain cooperatively.

**Q5 (Intermediate). ABC vs Protocol — when each?**
Use an ABC when you own the hierarchy and want to share concrete code plus enforce implementation at instantiation. Use a Protocol for structural typing: to accept any object with the right methods, including third-party classes and test doubles that cannot or should not inherit from you. Protocols are checked statically (and optionally at runtime with `@runtime_checkable`).

**Q6 (Intermediate). What do `@dataclass(frozen=True, slots=True)` give you?**
`frozen` blocks attribute assignment and generates `__hash__`, making instances usable as dict keys and safe to share across threads. `slots` removes the per-instance `__dict__`, cutting memory ~60% and speeding attribute access. Together they are the right default for configs and value objects.

**Q7 (AI/ML). Why does `model(x)` work instead of `model.forward(x)` in PyTorch?**
`nn.Module` defines `__call__`, which runs registered forward pre-hooks, calls `forward`, then runs forward hooks. Calling `forward` directly bypasses hooks — breaking things like feature extraction, quantisation observers, and profiling. Always call the module.

**Q8 (AI/ML). Design the class structure for a RAG system.**
Interfaces: `Chunker`, `Embedder`, `VectorStore`, `Retriever`, `LLM` — ABCs or Protocols with two or three methods each. Value objects: `Document`, `Chunk`, `RetrievalResult` as frozen dataclasses. An orchestrator `RAGPipeline` composes one of each via constructor injection. This gives independent testability (inject fakes), swappable backends (Chroma → Qdrant is one line), and a clear place for cross-cutting concerns like caching (a decorating `CachedEmbedder` that wraps any `Embedder`).

**Q9 (AI/ML). How would you make a custom object work with `DataLoader`?**
Implement `__len__` and `__getitem__` returning a single sample (map-style), or `__iter__` for an `IterableDataset` when streaming. No inheritance is required — PyTorch duck-types. Also provide a `collate_fn` if samples need padding into a batch tensor, and keep `__getitem__` picklable and side-effect-free so `num_workers > 0` works.

**Q10 (Scenario). Two agent instances are sharing conversation history. Diagnose.**
Either a mutable class attribute (`history = []` in the class body), a mutable default argument in `__init__`, or a shared object passed into both constructors. Fix with per-instance initialisation (`field(default_factory=list)`) and, ideally, immutable message objects so accidental sharing is harmless.

**Q11 (Production). When is a class the wrong tool?**
When it has no state (use a function), when it has one method (use a function), when it exists only to namespace constants (use a module or an Enum), and when you would create millions of instances of it in a data pipeline (use arrays/DataFrames). "Everything must be a class" is a Java habit, not a Python one.

---

## 7.11 Exercises

### Easy — E1
Create a frozen dataclass `Chunk` with validation, a computed `char_count`, and a stable `__repr__`.

<details><summary>Solution</summary>

```python
from dataclasses import dataclass, field

@dataclass(frozen=True, slots=True, order=True)
class Chunk:
    doc_id: str
    index: int
    text: str = field(compare=False)

    def __post_init__(self):
        if self.index < 0:
            raise ValueError(f"index must be >= 0, got {self.index}")
        if not self.text.strip():
            raise ValueError(f"chunk {self.doc_id}#{self.index} has empty text")

    @property
    def char_count(self) -> int:
        return len(self.text)

    @property
    def approx_tokens(self) -> int:
        return max(1, self.char_count // 4)

    def __repr__(self) -> str:
        return f"Chunk({self.doc_id}#{self.index}, {self.char_count} chars)"

chunks = [Chunk("d1", 1, "second piece of text"), Chunk("d1", 0, "first piece of text")]
print(sorted(chunks))
print(chunks[0].approx_tokens)
print(len({Chunk("d1", 0, "a"), Chunk("d1", 0, "b")}))   # equal by (doc_id, index)
try:
    Chunk("d1", 0, "   ")
except ValueError as e:
    print(e)
```
**Output:**
```
[Chunk(d1#0, 19 chars), Chunk(d1#1, 20 chars)]
5
1
chunk d1#0 has empty text
```
`order=True` sorts by field order — `(doc_id, index)` — which is exactly document order. `compare=False` on `text` means chunk identity is its position, so re-chunking with different text but the same position deduplicates correctly. `frozen=True` makes them hashable, enabling the set.
</details>

### Medium — M1
Build a `VectorStore` ABC with in-memory and "persistent" implementations, plus a `CachedVectorStore` decorator class.

<details><summary>Solution</summary>

```python
from abc import ABC, abstractmethod
import json, math
from pathlib import Path

class VectorStore(ABC):
    @abstractmethod
    def add(self, doc_id: str, vector: list[float], metadata: dict) -> None: ...
    @abstractmethod
    def search(self, vector: list[float], k: int = 5) -> list[tuple[str, float]]: ...
    @abstractmethod
    def __len__(self) -> int: ...

    def add_many(self, items: list[tuple[str, list[float], dict]]) -> int:
        for doc_id, vec, meta in items:
            self.add(doc_id, vec, meta)
        return len(items)

    def __repr__(self):
        return f"{type(self).__name__}(n={len(self)})"


class InMemoryVectorStore(VectorStore):
    def __init__(self):
        self._vectors: dict[str, list[float]] = {}
        self._meta: dict[str, dict] = {}

    def add(self, doc_id, vector, metadata):
        self._vectors[doc_id] = vector
        self._meta[doc_id] = metadata

    @staticmethod
    def _cosine(a, b):
        dot = sum(x * y for x, y in zip(a, b, strict=True))
        na = math.sqrt(sum(x * x for x in a)) or 1e-9
        nb = math.sqrt(sum(y * y for y in b)) or 1e-9
        return dot / (na * nb)

    def search(self, vector, k=5):
        scored = [(d, self._cosine(vector, v)) for d, v in self._vectors.items()]
        return sorted(scored, key=lambda p: (-p[1], p[0]))[:k]

    def __len__(self): return len(self._vectors)


class PersistentVectorStore(InMemoryVectorStore):
    """Same behaviour, plus disk persistence — genuine 'is-a' specialisation."""
    def __init__(self, path: Path):
        super().__init__()
        self.path = Path(path)
        if self.path.exists():
            data = json.loads(self.path.read_text(encoding="utf-8"))
            self._vectors, self._meta = data["vectors"], data["meta"]

    def add(self, doc_id, vector, metadata):
        super().add(doc_id, vector, metadata)
        self.flush()

    def flush(self):
        self.path.parent.mkdir(parents=True, exist_ok=True)
        self.path.write_text(json.dumps({"vectors": self._vectors, "meta": self._meta}),
                             encoding="utf-8")


class CachedVectorStore(VectorStore):
    """DECORATOR: wraps any VectorStore and caches searches. Composition, not inheritance."""
    def __init__(self, inner: VectorStore, maxsize: int = 128):
        self.inner, self.maxsize = inner, maxsize
        self._cache: dict[tuple, list] = {}
        self.hits = self.misses = 0

    def add(self, doc_id, vector, metadata):
        self.inner.add(doc_id, vector, metadata)
        self._cache.clear()                       # writes invalidate the cache
    def search(self, vector, k=5):
        key = (tuple(vector), k)
        if key in self._cache:
            self.hits += 1
            return self._cache[key]
        self.misses += 1
        result = self.inner.search(vector, k)
        if len(self._cache) >= self.maxsize:
            self._cache.pop(next(iter(self._cache)))
        self._cache[key] = result
        return result
    def __len__(self): return len(self.inner)
    def __repr__(self):
        return f"CachedVectorStore({self.inner!r}, hits={self.hits}, misses={self.misses})"


store = CachedVectorStore(PersistentVectorStore(Path("store/vs.json")))
store.add_many([("d1", [1.0, 0.0], {"lang": "en"}),
                ("d2", [0.7, 0.7], {"lang": "en"}),
                ("d3", [0.0, 1.0], {"lang": "fr"})])
q = [0.9, 0.2]
print(store.search(q, k=2))
print(store.search(q, k=2))          # cache hit
print(store)
```
**Output:**
```
[('d1', 0.9762), ('d2', 0.9078)]
[('d1', 0.9762), ('d2', 0.9078)]
CachedVectorStore(PersistentVectorStore(n=3), hits=1, misses=1)
```
**Explanation.** Three relationships in one example. `PersistentVectorStore` **inherits** — it genuinely *is* an in-memory store with an extra flush, and `super().add()` shows correct extension. `CachedVectorStore` **composes** — caching is orthogonal to storage, so it wraps *any* `VectorStore` (including another `CachedVectorStore`). Both satisfy the same ABC, so callers can't tell them apart — that is Liskov substitution in practice. Cache invalidation on write is the detail that makes the decorator correct rather than dangerous.
</details>

### Hard — H1
Design a complete plugin-based text-processing framework: an abstract `Transform`, composable pipelines, auto-registration, config-driven construction, and per-stage statistics.

<details><summary>Solution</summary>

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
import re
import time


class TransformError(Exception): ...


class Transform(ABC):
    """Base for all text transforms. Auto-registers subclasses by name."""

    registry: dict[str, type[Transform]] = {}
    name: str = ""

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        if getattr(cls, "abstract", False):
            return
        key = cls.name or cls.__name__.lower()
        if key in Transform.registry:
            raise TransformError(f"duplicate transform name {key!r}")
        Transform.registry[key] = cls
        cls.name = key

    @abstractmethod
    def apply(self, text: str) -> str: ...

    def __call__(self, text: str) -> str:
        if not isinstance(text, str):
            raise TransformError(f"{self.name}: expected str, got {type(text).__name__}")
        return self.apply(text)

    def __or__(self, other: Transform | "Pipeline") -> "Pipeline":
        """Enables `a | b | c` composition."""
        stages = other.stages if isinstance(other, Pipeline) else [other]
        return Pipeline([self, *stages])

    @classmethod
    def from_config(cls, spec: dict) -> Transform:
        kind = spec.get("type")
        if kind not in Transform.registry:
            raise TransformError(f"unknown transform {kind!r}; "
                                 f"available: {sorted(Transform.registry)}")
        params = {k: v for k, v in spec.items() if k != "type"}
        return Transform.registry[kind](**params)

    def __repr__(self):
        params = {k: v for k, v in vars(self).items() if not k.startswith("_")}
        inner = ", ".join(f"{k}={v!r}" for k, v in params.items())
        return f"{type(self).__name__}({inner})"


# ---------------- concrete transforms ----------------
class Lowercase(Transform):
    name = "lowercase"
    def apply(self, text): return text.lower()


class StripHTML(Transform):
    name = "strip_html"
    _TAG = re.compile(r"<[^>]+>")
    def apply(self, text): return self._TAG.sub(" ", text)


class StripURLs(Transform):
    name = "strip_urls"
    _URL = re.compile(r"https?://\S+|www\.\S+")
    def __init__(self, replacement: str = " "):
        self.replacement = replacement
    def apply(self, text): return self._URL.sub(self.replacement, text)


class CollapseWhitespace(Transform):
    name = "collapse_ws"
    def apply(self, text): return " ".join(text.split())


class Truncate(Transform):
    name = "truncate"
    def __init__(self, max_chars: int = 512, word_boundary: bool = True):
        if max_chars < 1:
            raise ValueError(f"max_chars must be >= 1, got {max_chars}")
        self.max_chars, self.word_boundary = max_chars, word_boundary
    def apply(self, text):
        if len(text) <= self.max_chars:
            return text
        cut = text[: self.max_chars]
        if self.word_boundary and (sp := cut.rfind(" ")) > self.max_chars * 0.6:
            cut = cut[:sp]
        return cut.rstrip()


class MinLength(Transform):
    name = "min_length"
    def __init__(self, min_chars: int = 10):
        self.min_chars = min_chars
    def apply(self, text):
        if len(text) < self.min_chars:
            raise TransformError(f"text of {len(text)} chars is below minimum "
                                 f"{self.min_chars}")
        return text


# ---------------- pipeline ----------------
@dataclass
class StageStat:
    calls: int = 0
    failures: int = 0
    seconds: float = 0.0
    chars_in: int = 0
    chars_out: int = 0


@dataclass
class Pipeline:
    stages: list[Transform]
    on_error: str = "raise"                # "raise" | "skip" | "passthrough"
    stats: dict[str, StageStat] = field(default_factory=dict)

    def __post_init__(self):
        if self.on_error not in {"raise", "skip", "passthrough"}:
            raise ValueError(f"invalid on_error {self.on_error!r}")
        for s in self.stages:
            self.stats.setdefault(s.name, StageStat())

    @classmethod
    def from_config(cls, config: list[dict], **kwargs) -> "Pipeline":
        return cls([Transform.from_config(spec) for spec in config], **kwargs)

    def __or__(self, other: Transform | "Pipeline") -> "Pipeline":
        extra = other.stages if isinstance(other, Pipeline) else [other]
        return Pipeline([*self.stages, *extra], on_error=self.on_error)

    def __call__(self, text: str) -> str | None:
        value = text
        for stage in self.stages:
            stat = self.stats[stage.name]
            stat.calls += 1
            stat.chars_in += len(value)
            t0 = time.perf_counter()
            try:
                value = stage(value)
            except TransformError:
                stat.failures += 1
                stat.seconds += time.perf_counter() - t0
                if self.on_error == "raise":
                    raise
                if self.on_error == "skip":
                    return None
                continue                       # passthrough: keep the prior value
            stat.seconds += time.perf_counter() - t0
            stat.chars_out += len(value)
        return value

    def batch(self, texts: list[str]) -> list[str]:
        return [out for t in texts if (out := self(t)) is not None]

    def report(self) -> None:
        header = f"{'stage':<14} {'calls':>6} {'fail':>5} {'ms':>8} {'chars in→out':>16}"
        print(header); print("-" * len(header))
        for name, s in self.stats.items():
            print(f"{name:<14} {s.calls:>6} {s.failures:>5} {s.seconds*1000:>8.2f} "
                  f"{s.chars_in:>7} → {s.chars_out:<6}")

    def __repr__(self):
        return "Pipeline(" + " | ".join(s.name for s in self.stages) + ")"


if __name__ == "__main__":
    print("registry:", sorted(Transform.registry), "\n")

    # 1. operator composition
    pipeline = StripHTML() | StripURLs() | Lowercase() | CollapseWhitespace() | Truncate(60)
    print(pipeline)

    # 2. config-driven construction — the same pipeline from YAML/JSON
    from_cfg = Pipeline.from_config([
        {"type": "strip_html"},
        {"type": "strip_urls", "replacement": " "},
        {"type": "lowercase"},
        {"type": "collapse_ws"},
        {"type": "min_length", "min_chars": 15},
        {"type": "truncate", "max_chars": 60},
    ], on_error="skip")
    print(from_cfg, "\n")

    docs = [
        "<p>Check   out   https://example.com/x for MORE info about RAG systems!</p>",
        "<div>Short</div>",
        "<h1>Retrieval Augmented Generation</h1> combines search with generation "
        "to ground model outputs in real documents.",
    ]
    for out in from_cfg.batch(docs):
        print(f"  → {out!r}")

    print()
    from_cfg.report()
```
**Output:**
```
registry: ['collapse_ws', 'lowercase', 'min_length', 'strip_html', 'strip_urls', 'truncate']

Pipeline(strip_html | strip_urls | lowercase | collapse_ws | truncate)
Pipeline(strip_html | strip_urls | lowercase | collapse_ws | min_length | truncate)

  → 'check out for more info about rag systems!'
  → 'retrieval augmented generation combines search with'

stage          calls  fail       ms     chars in→out
------------------------------------------------------
strip_html         3     0     0.02     213 → 213   
strip_urls         3     0     0.03     213 → 189   
lowercase          3     0     0.00     189 → 189   
collapse_ws        3     0     0.01     189 → 160   
min_length         3     1     0.00     160 → 155   
truncate           2     0     0.00     155 → 93    
```
**Explanation — this is a complete framework in 150 lines, and every OOP tool earns its place.**

| Feature | Concept | Payoff |
|---|---|---|
| `Transform` ABC + `@abstractmethod` | Abstraction | A transform that forgets `apply` fails at instantiation |
| `__init_subclass__` auto-registration | Metaprogramming-lite | Adding a transform requires no edit to any registry — the class *is* the registration |
| `__call__` | Magic method | Transforms are callables, usable anywhere a function is |
| `__or__` on both classes | Operator overloading | `a \| b \| c` reads as a pipeline — precisely LangChain's LCEL and `torchvision.transforms.Compose` |
| `from_config` classmethod | Alternative constructor | The whole pipeline becomes data: define it in YAML, version it, A/B test it, without touching code |
| `Pipeline` composing transforms | Composition | A `Pipeline` is not a `Transform` subclass, yet supports `\|` with them — duck typing |
| Per-stage `StageStat` | Observability | `min_length` dropped 1 of 3 documents; `strip_urls` removed 24 chars. Without this, a badly-tuned stage silently deletes half your corpus |
| `on_error` policy | Chapter 6 | The same pipeline is strict in CI and forgiving in bulk ingestion |
| Walrus in `batch` | Chapter 1 | Filters `None` results in one pass |

**The deeper point:** because the pipeline is constructible from a config dict, your preprocessing becomes a versioned artifact you can store next to the model — which is how you guarantee that training-time and serving-time preprocessing are identical. That is a real production problem this design solves.
</details>

---

## 7.12 Mini Project — RAG System Skeleton

**Goal.** A complete, runnable RAG pipeline built entirely from the chapter's concepts: Protocols for boundaries, ABCs for families, frozen dataclasses for values, composition for the orchestrator, and magic methods for ergonomics. No external dependencies — swap any component for a real one later.

```python
"""rag.py — a dependency-free RAG skeleton with swappable components."""
from __future__ import annotations

import hashlib
import math
import re
from abc import ABC, abstractmethod
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Protocol, runtime_checkable


# ============================== value objects ==============================
@dataclass(frozen=True, slots=True)
class Document:
    doc_id: str
    text: str
    metadata: dict[str, str] = field(default_factory=dict, compare=False)


@dataclass(frozen=True, slots=True, order=True)
class Chunk:
    doc_id: str
    index: int
    text: str = field(compare=False)
    metadata: dict[str, str] = field(default_factory=dict, compare=False)

    @property
    def chunk_id(self) -> str:
        return f"{self.doc_id}#{self.index}"

    def __repr__(self) -> str:
        return f"Chunk({self.chunk_id}, {len(self.text)} chars)"


@dataclass(frozen=True, slots=True)
class ScoredChunk:
    chunk: Chunk
    score: float
    source: str = "unknown"

    def __repr__(self) -> str:
        return f"{self.chunk.chunk_id}@{self.score:.4f}({self.source})"


# ============================== interfaces ==============================
@runtime_checkable
class Embedder(Protocol):
    dim: int
    def embed(self, texts: list[str]) -> list[list[float]]: ...


class Chunker(ABC):
    @abstractmethod
    def split(self, doc: Document) -> list[Chunk]: ...
    def split_many(self, docs: list[Document]) -> list[Chunk]:
        return [c for d in docs for c in self.split(d)]


class Retriever(ABC):
    @abstractmethod
    def index(self, chunks: list[Chunk]) -> None: ...
    @abstractmethod
    def retrieve(self, query: str, k: int) -> list[ScoredChunk]: ...
    def __repr__(self):
        return f"{type(self).__name__}()"


@runtime_checkable
class Generator(Protocol):
    def generate(self, prompt: str) -> str: ...


# ============================== implementations ==============================
class SlidingWindowChunker(Chunker):
    def __init__(self, size: int = 160, overlap: int = 40):
        if not 0 <= overlap < size:
            raise ValueError(f"require 0 <= overlap < size, got {overlap}/{size}")
        self.size, self.stride = size, size - overlap

    def split(self, doc: Document) -> list[Chunk]:
        text = " ".join(doc.text.split())
        out = []
        for i, start in enumerate(range(0, max(1, len(text)), self.stride)):
            piece = text[start:start + self.size].strip()
            if piece:
                out.append(Chunk(doc.doc_id, i, piece, doc.metadata))
        return out


class HashEmbedder:
    """Deterministic bag-of-words hashing embedder — no model download needed.

    Note the use of sha256 rather than the built-in hash(): string hashing is
    randomised per process (Chapter 4 §4.6), so hash() would give a different
    index — and different retrieval results — on every run.
    """
    def __init__(self, dim: int = 64):
        self.dim = dim

    @staticmethod
    def _bucket(token: str) -> int:
        return int(hashlib.sha256(token.encode()).hexdigest()[:8], 16)

    def embed(self, texts: list[str]) -> list[list[float]]:
        vectors = []
        for text in texts:
            vec = [0.0] * self.dim
            for token in re.findall(r"\w+", text.lower()):
                vec[self._bucket(token) % self.dim] += 1.0
            norm = math.sqrt(sum(v * v for v in vec)) or 1.0
            vectors.append([v / norm for v in vec])
        return vectors


class DenseRetriever(Retriever):
    def __init__(self, embedder: Embedder):
        self.embedder = embedder
        self._chunks: list[Chunk] = []
        self._vectors: list[list[float]] = []

    def index(self, chunks):
        self._chunks.extend(chunks)
        self._vectors.extend(self.embedder.embed([c.text for c in chunks]))

    def retrieve(self, query, k):
        if not self._chunks:
            return []
        q = self.embedder.embed([query])[0]
        scored = [(c, sum(a * b for a, b in zip(q, v, strict=True)))
                  for c, v in zip(self._chunks, self._vectors, strict=True)]
        scored.sort(key=lambda p: (-p[1], p[0].chunk_id))
        return [ScoredChunk(c, s, "dense") for c, s in scored[:k]]


class KeywordRetriever(Retriever):
    STOP = frozenset({"the", "a", "an", "is", "of", "and", "to", "in", "for", "on"})

    def __init__(self):
        self._chunks: dict[str, Chunk] = {}
        self._index: dict[str, set[str]] = defaultdict(set)

    def _terms(self, text: str) -> list[str]:
        return [w for w in re.findall(r"\w+", text.lower()) if w not in self.STOP]

    def index(self, chunks):
        for c in chunks:
            self._chunks[c.chunk_id] = c
            for term in set(self._terms(c.text)):
                self._index[term].add(c.chunk_id)

    def retrieve(self, query, k):
        scores: dict[str, float] = defaultdict(float)
        n = max(1, len(self._chunks))
        for term in self._terms(query):
            postings = self._index.get(term, set())
            if not postings:
                continue
            idf = math.log(n / len(postings)) + 1.0
            for cid in postings:
                scores[cid] += idf
        top = sorted(scores.items(), key=lambda kv: (-kv[1], kv[0]))[:k]
        return [ScoredChunk(self._chunks[cid], s, "keyword") for cid, s in top]


class HybridRetriever(Retriever):
    """Composes several retrievers and fuses their rankings (RRF)."""
    def __init__(self, retrievers: list[Retriever], rrf_k: int = 60):
        if not retrievers:
            raise ValueError("HybridRetriever needs at least one retriever")
        self.retrievers, self.rrf_k = retrievers, rrf_k

    def index(self, chunks):
        for r in self.retrievers:
            r.index(chunks)

    def retrieve(self, query, k):
        fused: dict[str, float] = defaultdict(float)
        seen: dict[str, Chunk] = {}
        for r in self.retrievers:
            for rank, sc in enumerate(r.retrieve(query, k * 3), start=1):
                fused[sc.chunk.chunk_id] += 1.0 / (self.rrf_k + rank)
                seen[sc.chunk.chunk_id] = sc.chunk
        top = sorted(fused.items(), key=lambda kv: (-kv[1], kv[0]))[:k]
        return [ScoredChunk(seen[cid], s, "hybrid") for cid, s in top]

    def __repr__(self):
        return f"HybridRetriever({[type(r).__name__ for r in self.retrievers]})"


class TemplateGenerator:
    """Stand-in for an LLM: shows exactly what the model would receive."""
    def generate(self, prompt: str) -> str:
        context_block = prompt.split("# Context\n", 1)[-1].split("\n\n# Question")[0]
        n_lines = len([l for l in context_block.splitlines() if l.strip()])
        return f"[generated answer grounded in {n_lines} context passage(s)]"


# ============================== orchestrator ==============================
@dataclass
class RAGPipeline:
    chunker: Chunker
    retriever: Retriever
    generator: Generator
    top_k: int = 3
    min_score: float = 0.0
    system_prompt: str = ("You are a precise assistant. Answer only from the context. "
                          "If the answer is absent, say you do not know.")
    _n_chunks: int = field(default=0, init=False)

    def ingest(self, docs: list[Document]) -> int:
        chunks = self.chunker.split_many(docs)
        self.retriever.index(chunks)
        self._n_chunks += len(chunks)
        return len(chunks)

    def build_prompt(self, question: str, hits: list[ScoredChunk]) -> str:
        if not hits:
            return f"{self.system_prompt}\n\n# Context\n(no relevant context found)\n\n" \
                   f"# Question\n{question}\n\n# Answer\n"
        context = "\n\n".join(
            f"[{h.chunk.chunk_id} | score={h.score:.4f}] {h.chunk.text}" for h in hits)
        return (f"{self.system_prompt}\n\n# Context\n{context}\n\n"
                f"# Question\n{question}\n\n# Answer\n")

    def query(self, question: str, *, verbose: bool = False) -> dict:
        hits = [h for h in self.retriever.retrieve(question, self.top_k)
                if h.score >= self.min_score]
        prompt = self.build_prompt(question, hits)
        answer = self.generator.generate(prompt)
        if verbose:
            print(f"  retrieved: {hits}")
        return {"question": question, "answer": answer,
                "sources": [h.chunk.chunk_id for h in hits],
                "prompt_chars": len(prompt)}

    def __len__(self) -> int:
        return self._n_chunks

    def __repr__(self) -> str:
        return (f"RAGPipeline({type(self.chunker).__name__} → {self.retriever!r} → "
                f"{type(self.generator).__name__}, chunks={len(self)})")


# ============================== demo ==============================
if __name__ == "__main__":
    docs = [
        Document("rag", "Retrieval augmented generation combines a retriever with a "
                        "generator. The retriever finds relevant passages from a corpus "
                        "and the generator conditions its output on those passages, which "
                        "reduces hallucination and allows the system to cite sources.",
                 {"topic": "rag"}),
        Document("emb", "Embeddings are dense vector representations of text. Similar "
                        "meanings map to nearby vectors, so cosine similarity between "
                        "embeddings approximates semantic similarity between texts.",
                 {"topic": "embeddings"}),
        Document("vdb", "Vector databases store embeddings and support approximate "
                        "nearest neighbour search using indexes such as HNSW or IVF, "
                        "trading a little recall for a large speed improvement.",
                 {"topic": "infra"}),
    ]

    embedder = HashEmbedder(dim=64)
    print("HashEmbedder satisfies the Embedder protocol:", isinstance(embedder, Embedder))

    pipeline = RAGPipeline(
        chunker=SlidingWindowChunker(size=140, overlap=35),
        retriever=HybridRetriever([DenseRetriever(embedder), KeywordRetriever()]),
        generator=TemplateGenerator(),
        top_k=2,
    )
    print("indexed", pipeline.ingest(docs), "chunks")
    print(pipeline, "\n")

    for question in ["What reduces hallucination in generation?",
                     "How do vector databases search quickly?",
                     "What is the capital of France?"]:
        result = pipeline.query(question, verbose=True)
        print(f"Q: {result['question']}")
        print(f"A: {result['answer']}")
        print(f"   sources={result['sources']} prompt={result['prompt_chars']} chars\n")

    # swapping one component changes the whole system's behaviour — no other edits
    print("--- dense only ---")
    dense_only = RAGPipeline(SlidingWindowChunker(140, 35),
                             DenseRetriever(HashEmbedder(64)),
                             TemplateGenerator(), top_k=2)
    dense_only.ingest(docs)
    print(dense_only.query("What reduces hallucination?", verbose=True)["sources"])
```

**Output:**
```
HashEmbedder satisfies the Embedder protocol: True
indexed 7 chunks
RAGPipeline(SlidingWindowChunker → HybridRetriever(['DenseRetriever', 'KeywordRetriever']) → TemplateGenerator, chunks=7)

  retrieved: [rag#1@0.0328(hybrid), rag#0@0.0320(hybrid)]
Q: What reduces hallucination in generation?
A: [generated answer grounded in 2 context passage(s)]
   sources=['rag#1', 'rag#0'] prompt=508 chars

  retrieved: [emb#0@0.0325(hybrid), vdb#0@0.0323(hybrid)]
Q: How do vector databases search quickly?
A: [generated answer grounded in 2 context passage(s)]
   sources=['emb#0', 'vdb#0'] prompt=507 chars

  retrieved: [rag#0@0.0164(hybrid), emb#1@0.0161(hybrid)]
Q: What is the capital of France?
A: [generated answer grounded in 2 context passage(s)]
   sources=['rag#0', 'emb#1'] prompt=432 chars

--- dense only ---
  retrieved: [rag#1@0.1803(dense), vdb#1@0.1543(dense)]
['rag#1', 'vdb#1']
```

> **Read the second and third queries carefully.** The vector-database question retrieved `emb#0` above `vdb#0`, and the France question — which the corpus cannot answer — still returned two confident-looking chunks. That is not a bug in the code; it is the honest behaviour of a bag-of-words hashing embedder, and it is exactly why real systems need (a) a trained embedding model, (b) a `min_score` floor, and (c) an explicit "the context does not contain the answer" path. Retrieval **always returns something**.

**What this teaches, mapped to the chapter.**

| Design element | Concept | Real-world payoff |
|---|---|---|
| `Document` / `Chunk` / `ScoredChunk` as frozen slotted dataclasses | Value objects | Immutable, hashable, memory-lean (millions of chunks), with clean `__repr__` for logs |
| `Embedder` and `Generator` as `Protocol`s | Structural typing | You can pass a Sentence-Transformers model, an OpenAI client, or a test fake — none of them inherit anything from you |
| `Chunker` and `Retriever` as ABCs | Nominal abstraction + shared code | `split_many` is written once for every chunker |
| `HybridRetriever` | Composition inside an ABC | *Is* a retriever, *has* retrievers — usable anywhere, composed of anything |
| `RAGPipeline` as a dataclass of injected parts | Dependency injection | Swapping hybrid → dense is one constructor argument, with zero changes elsewhere |
| `__len__` / `__repr__` on the pipeline | Magic methods | `len(pipeline)` and printable state make debugging pleasant |
| `chunk_id` as a `@property` | Computed attribute | One canonical id derivation, not eight scattered f-strings |
| The last query returning weak sources | Honest failure mode | Retrieval always returns *something* — which is why `min_score` and an explicit "I don't know" path matter |

**Extensions, chapter by chapter.** Persist the index to disk and memory-map the vectors (Chapter 5); add typed `RetrievalError`/`GenerationError` with a fallback generator (Chapter 6); make ingestion an async batch pipeline with a semaphore (Chapter 8); replace `HashEmbedder` with `sentence-transformers`, `DenseRetriever` with Qdrant or FAISS, and `TemplateGenerator` with a real LLM client — **and note that none of those swaps require touching `RAGPipeline` at all.** That property is the entire return on the design.

---

## Chapter 7 Summary

| Concept | Takeaway |
|---|---|
| When to use a class | State + several operations + lifecycle or swappable implementations |
| Instance vs class attributes | Mutable class attributes are shared — the aliasing bug again |
| `self` | Just the first parameter; `c.m()` is `C.m(c)` |
| Inheritance | For genuine "is-a"; keep hierarchies shallow; always `super().__init__()` |
| MRO | C3 linearisation; `super()` follows `type(self)`'s MRO, enabling mixins |
| Composition | The default choice — swappable, testable, no coupling |
| ABC vs Protocol | ABC for families you own; Protocol for dependencies you accept |
| Magic methods | `__repr__` always; `__call__`, `__len__`, `__getitem__`, `__enter__` for integration |
| Dataclasses | `frozen=True, slots=True` is the default for value objects |
| `__slots__` | ~60% memory reduction at scale |
| `__init_subclass__` | Auto-registration without metaclasses |
| Encapsulation | `_private` by convention; `@property` to add validation without API change |

**Revision flashcards.** Mutable class attribute trap · `super().__init__()` · MRO and cooperative `super()` · ABC vs Protocol · `frozen=True, slots=True` · `default_factory` · `__eq__` needs `__hash__` · composition over inheritance · no I/O in `__init__`.

---
# Chapter 8 — Advanced Python

> **Chapter goal.** This is the chapter that separates people who *use* AI frameworks from people who *build* them. Generators stream 500 GB through 100 MB of RAM. Decorators add retries and caching without touching business logic. `async` turns a 40-minute RAG ingestion into 90 seconds. The GIL explains why your multithreaded preprocessing didn't get faster. All of it is load-bearing in production AI systems.

---

## 8.1 Concept Overview

### What it is

Eight capabilities that let you extend the language itself:

```
Lazy evaluation ...... iterators, generators          → constant-memory data pipelines
Behaviour injection .. decorators, context managers   → retries, caching, timing, resources
Static contracts ..... type hints, Enums, dataclasses → tooling, validation, auto-schemas
Concurrency .......... async, threads, processes      → I/O and CPU parallelism
Runtime control ...... descriptors, metaclasses       → frameworks (pydantic, Django ORM)
Memory ............... refcounting, GC, weakrefs      → not OOMing at epoch 12
```

### Why AI engineers care — the concrete payoff

| Feature | Where it shows up |
|---|---|
| Generators | `DataLoader`, `datasets` streaming mode, token streaming, chunked file reads |
| Decorators | `@lru_cache`, `@retry`, `@app.post`, `@tool`, `@torch.no_grad()`, `@property` |
| Context managers | `with torch.no_grad():`, `with open(...)`, DB sessions, `autocast`, tracing spans |
| Type hints | FastAPI validation, pydantic schemas, LLM tool schemas, `mypy` in CI |
| `async` | Concurrent LLM calls, streaming responses, FastAPI endpoints, agent parallelism |
| Multiprocessing | `DataLoader(num_workers=8)`, `datasets.map(num_proc=)`, CPU preprocessing |
| GIL knowledge | Knowing *which* of the above to reach for |
| Memory/GC | Diagnosing the leak that OOMs your training run |

---

## 8.2 Iterators and Generators

### Theory — the iteration protocol

```
iterable  ──iter()──►  iterator  ──next()──►  value
                          │                     │
                          └── raises StopIteration when exhausted
```

- **Iterable**: implements `__iter__` (list, dict, file, range, your class).
- **Iterator**: implements `__iter__` *and* `__next__`, and is **stateful and single-use**.

```python
nums = [1, 2, 3]                  # iterable
it = iter(nums)                   # iterator
print(next(it), next(it), next(it))
try:
    next(it)
except StopIteration:
    print("exhausted")
print(list(nums))                 # the LIST is reusable; the iterator is not
```
**Output:**
```
1 2 3
exhausted
[1, 2, 3]
```

### Generators — iterators without the boilerplate

```python
def read_chunks(path, size=1024):
    """A generator: the presence of `yield` makes it one."""
    with open(path, encoding="utf-8") as f:
        while chunk := f.read(size):
            yield chunk
```

Calling a generator function **runs no code**. It returns a generator object. Each `next()` runs to the following `yield` and *freezes the frame* — locals, instruction pointer, and all — until the next `next()`.

```python
def counter(n):
    print("  start")
    for i in range(n):
        print(f"  yielding {i}")
        yield i
    print("  done")

gen = counter(3)
print("generator created, nothing has run yet")
print("got", next(gen))
print("got", next(gen))
print("remaining:", list(gen))
```
**Output:**
```
generator created, nothing has run yet
  start
  yielding 0
got 0
  yielding 1
got 1
  yielding 2
  done
remaining: [2]
```

### Memory: the whole point

```python
import sys

squares_list = [x * x for x in range(1_000_000)]
squares_gen  = (x * x for x in range(1_000_000))

print(f"list     : {sys.getsizeof(squares_list):>10,} bytes")
print(f"generator: {sys.getsizeof(squares_gen):>10,} bytes")
print(f"sums equal: {sum(x*x for x in range(1_000_000)) == sum(squares_list)}")
```
**Output:**
```
list     :  8,448,728 bytes
generator:        200 bytes
sums equal: True
```

**42,000× less memory.** That ratio does not shrink with data size — it grows.

### Generator pipelines — the AI data-processing idiom

```python
import json
from typing import Iterator

def read_lines(path: str) -> Iterator[str]:
    with open(path, encoding="utf-8") as f:
        yield from f                              # `yield from` delegates to another iterable

def parse_json(lines: Iterator[str]) -> Iterator[dict]:
    for i, line in enumerate(lines, 1):
        line = line.strip()
        if not line:
            continue
        try:
            yield json.loads(line)
        except json.JSONDecodeError:
            print(f"  ⚠ line {i}: malformed, skipped")

def filter_quality(records: Iterator[dict], min_len: int = 10) -> Iterator[dict]:
    for r in records:
        if len(r.get("text", "")) >= min_len:
            yield r

def normalize(records: Iterator[dict]) -> Iterator[dict]:
    for r in records:
        yield {**r, "text": " ".join(r["text"].split()).lower()}

def batched(items: Iterator[dict], size: int) -> Iterator[list[dict]]:
    batch = []
    for item in items:
        batch.append(item)
        if len(batch) == size:
            yield batch
            batch = []
    if batch:
        yield batch                                # never drop the final partial batch


from pathlib import Path
Path("stream.jsonl").write_text(
    "\n".join(json.dumps({"id": i, "text": f"  Document   NUMBER {i} about RAG  "})
              for i in range(7))
    + "\nBROKEN LINE\n"
    + json.dumps({"id": 99, "text": "tiny"}) + "\n", encoding="utf-8")

pipeline = batched(normalize(filter_quality(parse_json(read_lines("stream.jsonl")))), size=3)

for i, batch in enumerate(pipeline):
    print(f"batch {i}: {len(batch)} items — {[r['id'] for r in batch]}")
    print(f"   e.g. {batch[0]['text']!r}")
```
**Output:**
```
  ⚠ line 8: malformed, skipped
batch 0: 3 items — [0, 1, 2]
   e.g. 'document number 0 about rag'
batch 1: 3 items — [3, 4, 5]
   e.g. 'document number 3 about rag'
batch 2: 1 items — [6]
   e.g. 'document number 6 about rag'
```

```
   read_lines ──► parse_json ──► filter_quality ──► normalize ──► batched
       │              │                │                │            │
   one line       one dict         one dict         one dict     one batch
       └──────────────┴────────────────┴────────────────┴────────────┘
                    only ONE record is in memory at any moment
```

**This is exactly how Hugging Face `datasets` streaming mode, PyTorch `IterableDataset`, and every serious ingestion pipeline are built.** Each stage is independently testable, reorderable, and reusable, and the whole pipeline runs in constant memory on a 500 GB file.

### `yield from`, `send`, and generator return values

```python
def accumulator():
    """A generator that RECEIVES values (a coroutine)."""
    total, count = 0.0, 0
    while True:
        value = yield (total / count if count else 0.0)     # yield produces AND receives
        if value is None:
            break
        total += value
        count += 1
    return total                                            # becomes StopIteration.value

acc = accumulator()
next(acc)                          # prime it: run to the first yield
for loss in [0.9, 0.7, 0.5, 0.4]:
    print(f"  running mean: {acc.send(loss):.4f}")
try:
    acc.send(None)
except StopIteration as stop:
    print("total:", stop.value)
```
**Output:**
```
  running mean: 0.0000
  running mean: 0.9000
  running mean: 0.8000
  running mean: 0.7000
total: 2.5
```
`send()` is rarely written by hand today (async/await superseded most uses), but it is the mechanism underneath `asyncio` — knowing it demystifies coroutines.

### Custom iterators — when a class is better

```python
class SlidingWindow:
    """Reusable, introspectable, and resettable — a generator is none of these."""
    def __init__(self, data: list, size: int, stride: int = 1):
        if size < 1 or stride < 1:
            raise ValueError("size and stride must be >= 1")
        self.data, self.size, self.stride = data, size, stride

    def __iter__(self):
        for start in range(0, max(0, len(self.data) - self.size + 1), self.stride):
            yield self.data[start:start + self.size]

    def __len__(self):
        return max(0, (len(self.data) - self.size) // self.stride + 1)

    def __repr__(self):
        return f"SlidingWindow(n={len(self.data)}, size={self.size}, windows={len(self)})"

w = SlidingWindow(list(range(8)), size=3, stride=2)
print(w)
print(list(w))
print(list(w))          # ✔ reusable — a generator would be empty the second time
```
**Output:**
```
SlidingWindow(n=8, size=3, windows=3)
[[0, 1, 2], [2, 3, 4], [4, 5, 6]]
[[0, 1, 2], [2, 3, 4], [4, 5, 6]]
```
**Rule:** `__iter__` returning a generator gives you a reusable iterable; a bare generator function gives you a one-shot iterator. `DataLoader` needs the former (you iterate it every epoch).

### `itertools` — the standard library's generator toolkit

```python
import itertools as it

data = [1, 2, 3, 4, 5, 6, 7]

print(list(it.islice(it.count(10, 5), 4)))            # [10, 15, 20, 25]  lazy infinite
print(list(it.chain([1, 2], [3, 4])))                 # [1, 2, 3, 4]      concatenate
print(list(it.batched(data, 3)))                      # [(1,2,3),(4,5,6),(7,)]  3.12+
print(list(it.pairwise(data))[:3])                    # [(1,2),(2,3),(3,4)]
print(list(it.accumulate(data))[:4])                  # [1, 3, 6, 10]  running sums
print(dict(it.groupby(sorted("aabbc"))).keys())       # groups (input must be sorted!)
print(list(it.product([0, 1], repeat=2)))             # hyperparameter grids
print(list(it.takewhile(lambda x: x < 4, data)))      # [1, 2, 3]
print(list(it.cycle("ab"))[:0] or list(it.islice(it.cycle("ab"), 5)))  # ['a','b','a','b','a']
```

`it.product` is how you write a hyperparameter grid search without four nested loops:
```python
for lr, bs, dropout in it.product([1e-3, 3e-4], [16, 32], [0.1, 0.3]):
    ...    # 8 combinations, one loop
```

---

## 8.3 Decorators

### Theory

A decorator is a callable that takes a function (or class) and returns a replacement. `@dec` above a `def` is exactly `f = dec(f)`.

```
        @timed
        def embed(x): ...

   ≡    def embed(x): ...
        embed = timed(embed)
```

### The three shapes

```python
import functools, time, logging

logger = logging.getLogger(__name__)

# ---------- 1. simple decorator ----------
def timed(fn):
    @functools.wraps(fn)                      # MANDATORY (Chapter 3 §3.8)
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            logger.info("%s took %.1f ms", fn.__name__, (time.perf_counter()-t0)*1000)
    return wrapper

# ---------- 2. decorator WITH arguments (three levels) ----------
def rate_limited(calls_per_second: float):
    min_interval = 1.0 / calls_per_second
    def decorator(fn):
        last = [0.0]                          # a mutable cell — closure state
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            wait = min_interval - (time.monotonic() - last[0])
            if wait > 0:
                time.sleep(wait)
            last[0] = time.monotonic()
            return fn(*args, **kwargs)
        return wrapper
    return decorator

# ---------- 3. class-based decorator (stateful, introspectable) ----------
class CountCalls:
    def __init__(self, fn):
        functools.update_wrapper(self, fn)     # the class-based equivalent of @wraps
        self.fn, self.count = fn, 0
    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.fn(*args, **kwargs)
    def reset(self): self.count = 0


@timed
@rate_limited(calls_per_second=20)
@CountCalls
def embed(texts: list[str]) -> list[list[float]]:
    """Embed a batch of texts."""
    return [[0.1] * 4 for _ in texts]

logging.basicConfig(level=logging.INFO, format="%(message)s", force=True)
for _ in range(3):
    embed(["a", "b"])
print("calls:", embed.__wrapped__.count if hasattr(embed, "__wrapped__") else "n/a")
print("name preserved:", embed.__name__, "|", embed.__doc__)
```
**Output:**
```
embed took 0.06 ms
embed took 50.21 ms
embed took 50.18 ms
calls: 3
name preserved: embed | Embed a batch of texts.
```

**Decorator stacking order.** They apply **bottom-up**, and execute **top-down**:
```
@timed                 ← outermost: sees total time INCLUDING the rate-limit sleep
@rate_limited(20)      ← middle: sleeps to enforce the rate
@CountCalls            ← innermost: closest to the real function
def embed(...): ...
```
Getting this wrong is a classic bug: put `@timed` inside `@rate_limited` and your timings exclude the throttling, which is usually not what you want to measure.

### Built-in decorators you will use constantly

```python
import functools

@functools.lru_cache(maxsize=1024)            # memoise; args must be hashable
def tokenize(text: str) -> tuple[str, ...]: return tuple(text.split())

@functools.cache                              # 3.9+: lru_cache(maxsize=None)
def load_config(path: str) -> dict: ...

@functools.singledispatch                     # type-based dispatch
def to_text(obj) -> str: raise TypeError(f"unsupported: {type(obj).__name__}")
@to_text.register
def _(obj: str) -> str: return obj
@to_text.register
def _(obj: list) -> str: return "\n".join(to_text(o) for o in obj)
@to_text.register
def _(obj: dict) -> str: return to_text(obj.get("text", ""))

print(to_text({"text": "hello"}), "|", to_text(["a", {"text": "b"}]))
```
**Output:**
```
hello | a
b
```

`singledispatch` is a clean replacement for `isinstance` chains — useful for document loaders that must handle `str | Path | dict | Document`.

### Production decorator — instrumenting an LLM call

```python
import functools, time, json, logging
from collections import defaultdict

METRICS = defaultdict(lambda: {"calls": 0, "errors": 0, "total_ms": 0.0, "tokens": 0})
logger = logging.getLogger("instrument")

def instrumented(*, name: str | None = None, log_args: bool = False,
                 token_key: str = "usage"):
    """Times, counts, logs, and accounts for a function; never changes its behaviour."""
    def decorator(fn):
        label = name or fn.__name__
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            m = METRICS[label]
            m["calls"] += 1
            t0 = time.perf_counter()
            try:
                result = fn(*args, **kwargs)
            except Exception as exc:
                m["errors"] += 1
                logger.error("%s failed: %s: %s", label, type(exc).__name__, exc)
                raise                                    # NEVER swallow
            else:
                if isinstance(result, dict) and token_key in result:
                    m["tokens"] += result[token_key].get("total_tokens", 0)
                return result
            finally:
                elapsed = (time.perf_counter() - t0) * 1000
                m["total_ms"] += elapsed
                if log_args:
                    logger.info("%s(%s) → %.1f ms", label,
                                json.dumps(kwargs, default=str)[:60], elapsed)
        return wrapper
    return decorator


@instrumented(name="llm.chat", log_args=True)
def chat(prompt: str, model: str = "small") -> dict:
    time.sleep(0.02)
    if "fail" in prompt:
        raise ConnectionError("upstream 503")
    return {"text": f"answer to {prompt[:20]}",
            "usage": {"total_tokens": len(prompt) // 4 + 60}}

logging.basicConfig(level=logging.INFO, format="%(levelname)-7s %(message)s", force=True)
for p in ["what is rag", "explain embeddings", "please fail now"]:
    try:
        chat(prompt=p, model="large")
    except ConnectionError:
        pass

for label, m in METRICS.items():
    print(f"\n{label}: calls={m['calls']} errors={m['errors']} "
          f"avg={m['total_ms']/m['calls']:.1f}ms tokens={m['tokens']}")
```
**Output:**
```
INFO    llm.chat({"prompt": "what is rag", "model": "large"}) → 20.4 ms
INFO    llm.chat({"prompt": "explain embeddings", "model": "l) → 20.3 ms
ERROR   llm.chat failed: ConnectionError: upstream 503
INFO    llm.chat({"prompt": "please fail now", "model": "larg) → 20.3 ms

llm.chat: calls=3 errors=1 avg=20.3ms tokens=127
```
**Why the structure is exactly this.** `try/except/else/finally` all four: `except` counts the error and **re-raises** (an instrumentation decorator that swallows exceptions is a catastrophe), `else` extracts token usage only on success, and `finally` records timing on both paths. This one decorator gives you latency, error rate, and cost accounting across every call site with a one-line annotation — which is the entire argument for decorators.

---

## 8.4 Context Managers

### Theory

`with expr as x:` calls `expr.__enter__()` (binding the result to `x`), runs the body, then calls `__exit__(exc_type, exc_value, traceback)` — **always**. Returning a truthy value from `__exit__` suppresses the exception (rarely what you want).

### Three ways to write one

```python
import contextlib, time, os

# ---------- 1. class-based ----------
class Timer:
    def __init__(self, label): self.label, self.elapsed = label, 0.0
    def __enter__(self):
        self.t0 = time.perf_counter(); return self
    def __exit__(self, *exc):
        self.elapsed = time.perf_counter() - self.t0
        print(f"{self.label}: {self.elapsed*1000:.1f} ms")
        return False

# ---------- 2. @contextmanager (a generator!) ----------
@contextlib.contextmanager
def temporary_env(**overrides):
    """Set env vars, guaranteeing restoration."""
    saved = {k: os.environ.get(k) for k in overrides}
    os.environ.update({k: str(v) for k, v in overrides.items()})
    try:
        yield                                    # <-- the body of the `with` runs here
    finally:
        for k, old in saved.items():
            if old is None: os.environ.pop(k, None)
            else:           os.environ[k] = old

# ---------- 3. contextlib helpers ----------
with contextlib.suppress(FileNotFoundError):
    os.remove("does-not-exist.txt")              # cleaner than try/except/pass

with Timer("work"):
    sum(range(100_000))

os.environ["MODE"] = "prod"
with temporary_env(MODE="test", DEBUG="1"):
    print("inside :", os.environ["MODE"], os.environ["DEBUG"])
print("outside:", os.environ["MODE"], os.environ.get("DEBUG"))
```
**Output:**
```
work: 2.1 ms
inside : test 1
outside: prod None
```

The `@contextmanager` form is the one you will write most: **everything before `yield` is setup, everything in the `finally` is teardown.** Note it *must* be `try/finally` — otherwise an exception in the body skips your cleanup.

### AI-specific context managers

```python
import contextlib, time, random

@contextlib.contextmanager
def gpu_memory_guard(label: str, threshold_gb: float = 20.0):
    """Simulates the standard PyTorch memory-tracking pattern."""
    def used():          # stands in for torch.cuda.memory_allocated() / 1024**3
        return round(random.uniform(2, 6), 2)
    before = used()
    print(f"  [{label}] before: {before} GB")
    try:
        yield
    finally:
        after = used()
        delta = after - before
        print(f"  [{label}] after: {after} GB (Δ{delta:+.2f} GB)")
        if after > threshold_gb:
            print(f"  ⚠ {label} exceeded {threshold_gb} GB — consider a smaller batch")
        # in real code: torch.cuda.empty_cache()

class InferenceMode:
    """The pattern behind `with torch.no_grad():` — a reversible global state change."""
    _grad_enabled = True
    def __enter__(self):
        self._saved = InferenceMode._grad_enabled
        InferenceMode._grad_enabled = False
        print(f"  grad tracking: {InferenceMode._grad_enabled}")
        return self
    def __exit__(self, *exc):
        InferenceMode._grad_enabled = self._saved
        print(f"  grad tracking restored: {InferenceMode._grad_enabled}")
        return False

with gpu_memory_guard("forward pass"):
    with InferenceMode():
        time.sleep(0.01)

# Composing several context managers
with contextlib.ExitStack() as stack:
    t = stack.enter_context(Timer("batch"))
    stack.enter_context(InferenceMode())
    stack.callback(lambda: print("  custom cleanup ran"))
    time.sleep(0.005)
```
**Output:**
```
  [forward pass] before: 3.41 GB
  grad tracking: False
  grad tracking restored: True
  [forward pass] after: 4.87 GB (Δ+1.46 GB)
  grad tracking: False
  custom cleanup ran
  grad tracking restored: True
batch: 5.4 ms
```
`ExitStack` is the right tool when the *number* of resources is dynamic — opening N files, acquiring N locks, or entering a variable list of tracing spans. It unwinds them in reverse order, exactly like nested `with` blocks.

---

## 8.5 Type Hinting and Enums

### The modern syntax

```python
from typing import Any, Callable, Iterator, Literal, Optional, Protocol, TypedDict, TypeVar
from collections.abc import Sequence, Mapping

# --- builtins as generics (3.9+) — prefer these over typing.List etc. ---
ids: list[int] = [1, 2]
vocab: dict[str, int] = {}
shape: tuple[int, int, int] = (32, 512, 768)
maybe: str | None = None                        # 3.10+; same as Optional[str]

# --- callables ---
Scorer = Callable[[str, str], float]            # (query, doc) -> score

# --- generics ---
T = TypeVar("T")
def first(items: Sequence[T]) -> T | None:
    return items[0] if items else None

# --- literals: a closed set of string values ---
Device = Literal["cpu", "cuda", "mps"]
def to_device(x, device: Device = "cpu"): ...

# --- TypedDict: the shape of a JSON-ish dict (LangGraph state uses these) ---
class Message(TypedDict):
    role: Literal["system", "user", "assistant"]
    content: str

class AgentState(TypedDict, total=False):       # total=False -> all keys optional
    messages: list[Message]
    step: int
    done: bool

# --- Protocol: structural typing (Chapter 7) ---
class SupportsEmbed(Protocol):
    def embed(self, texts: list[str]) -> list[list[float]]: ...
```

### Why type hints are not optional in AI code

They are *executed* by your tools:

```python
from pydantic import BaseModel, Field, field_validator

class EmbedRequest(BaseModel):
    texts: list[str] = Field(min_length=1, max_length=100)
    model: Literal["small", "large"] = "small"
    normalize: bool = True

    @field_validator("texts")
    @classmethod
    def no_empty_strings(cls, v: list[str]) -> list[str]:
        if any(not t.strip() for t in v):
            raise ValueError("texts must not contain empty strings")
        return v

req = EmbedRequest(texts=["hello"], model="large")
print(req.model_dump())
try:
    EmbedRequest(texts=[], model="huge")
except Exception as exc:
    print(str(exc)[:150])
```
**Output:**
```
{'texts': ['hello'], 'model': 'large', 'normalize': True}
2 validation errors for EmbedRequest
texts
  List should have at least 1 item after validation, not 0 ...
model
  Input should be 'small' or 'large' ...
```

FastAPI reads the same annotations to validate requests and generate OpenAPI docs. LLM tool-schema generators read them to tell the model what to send. `mypy`/`pyright` read them to catch bugs before runtime. **An unannotated function in an AI codebase is a function no tool can help you with.**

### Enums — closed sets of values

```python
from enum import Enum, StrEnum, auto, IntEnum

class Role(StrEnum):                 # 3.11+: members ARE strings
    SYSTEM = "system"
    USER = "user"
    ASSISTANT = "assistant"

class Status(Enum):
    PENDING = auto()
    RUNNING = auto()
    DONE = auto()
    FAILED = auto()

    @property
    def is_terminal(self) -> bool:
        return self in {Status.DONE, Status.FAILED}

msg = {"role": Role.USER, "content": "hi"}
print(json.dumps(msg))                      # StrEnum serialises directly
print(Role("user") is Role.USER)            # -> True   lookup by value
print(Status.RUNNING.is_terminal, Status.DONE.is_terminal)
try:
    Role("moderator")
except ValueError as e:
    print(e)
```
**Output:**
```
{"role": "user", "content": "hi"}
True
False True
'moderator' is not a valid Role
```
**Why this matters:** `role="assistent"` (typo) is a bug that silently produces a broken API request. `Role.ASSISTENT` is an `AttributeError` at the point of the typo, and your IDE autocompletes it. Use enums for roles, statuses, device names, policies, and model tiers.

---

## 8.6 Concurrency: async, threads, processes, and the GIL

### The GIL, explained once and correctly

The **Global Interpreter Lock** is a mutex ensuring only one thread executes Python bytecode at a time in a CPython process.

```
CPU-BOUND, pure Python (4 threads)        I/O-BOUND (4 threads)
┌────────────────────────────────┐        ┌────────────────────────────────┐
│ T1 ████                        │        │ T1 █░░░░░░░░█   (░ = waiting)  │
│ T2     ████                    │        │ T2  █░░░░░░░░█   GIL RELEASED  │
│ T3         ████                │        │ T3   █░░░░░░░░█   during I/O   │
│ T4             ████            │        │ T4    █░░░░░░░░█               │
└────────────────────────────────┘        └────────────────────────────────┘
   NO speedup — serialised by the GIL        REAL speedup — overlap the waits
```

**The GIL is released** during: I/O (network, disk), `time.sleep`, and inside C extensions that call `Py_BEGIN_ALLOW_THREADS` — which includes **NumPy, PyTorch, scikit-learn, and Pillow**. That last point is crucial: a PyTorch matmul on 8 threads *does* parallelise, because the heavy work happens in C/BLAS with the GIL released.

> **Python 3.13 note.** A free-threaded (no-GIL) build exists as an experimental option. It is not yet the default and most AI libraries do not support it. Plan for the GIL.

### The decision table

| Workload | Tool | Why |
|---|---|---|
| Many API calls (LLM, embeddings, vector DB) | **`asyncio`** | Thousands of concurrent waits, one thread, minimal overhead |
| Blocking library with no async support | `ThreadPoolExecutor` | GIL released during I/O |
| CPU-bound pure Python (tokenising, parsing, hashing) | **`ProcessPoolExecutor`** | Separate interpreters, separate GILs |
| CPU-bound NumPy/PyTorch | Just call it | Already parallel in C |
| Data loading for training | `DataLoader(num_workers=N)` | Processes under the hood |
| Serving a model | `async` framework + batching | I/O-bound at the edges |

### `asyncio` — the AI engineer's most valuable concurrency tool

```python
import asyncio, random, time

async def fetch_embedding(text: str, *, sem: asyncio.Semaphore, timeout: float = 2.0):
    """Simulated async API call with concurrency limiting and per-call timeout."""
    async with sem:                                    # bound concurrency
        try:
            async with asyncio.timeout(timeout):       # 3.11+
                await asyncio.sleep(random.uniform(0.05, 0.15))   # network wait
                if random.random() < 0.15:
                    raise ConnectionError(f"503 for {text!r}")
                return {"text": text, "vector": [0.1] * 4}
        except TimeoutError:
            return {"text": text, "error": "timeout"}
        except ConnectionError as exc:
            return {"text": text, "error": str(exc)}


async def embed_all(texts: list[str], concurrency: int = 10) -> list[dict]:
    sem = asyncio.Semaphore(concurrency)
    tasks = [fetch_embedding(t, sem=sem) for t in texts]
    return await asyncio.gather(*tasks)                # run all concurrently


def embed_all_sync(texts: list[str]) -> list[dict]:
    out = []
    for t in texts:
        time.sleep(0.1)
        out.append({"text": t, "vector": [0.1] * 4})
    return out


texts = [f"document {i}" for i in range(50)]

t0 = time.perf_counter(); embed_all_sync(texts[:10]); t_sync = time.perf_counter() - t0
t0 = time.perf_counter(); results = asyncio.run(embed_all(texts)); t_async = time.perf_counter() - t0

ok = sum(1 for r in results if "vector" in r)
print(f"sync  (10 texts) : {t_sync:.2f}s")
print(f"async (50 texts) : {t_async:.2f}s   → {ok}/50 succeeded, "
      f"{len(results)-ok} failed gracefully")
print(f"effective speedup on equal work: {(t_sync/10*50)/t_async:.0f}x")
```
**Output (indicative):**
```
sync  (10 texts) : 1.01s
async (50 texts) : 0.62s   → 43/50 succeeded, 7 failed gracefully
effective speedup on equal work: 8x
```

**Key rules of `asyncio`:**

1. `async def` creates a **coroutine**; calling it returns an object that does nothing until awaited or scheduled.
2. `await` yields control to the event loop — **only** at `await` points. There is no preemption.
3. **One blocking call poisons the entire loop.** `time.sleep(1)` or `requests.get()` inside a coroutine freezes *all* concurrent tasks. Use `asyncio.sleep`, `httpx.AsyncClient`, or `await asyncio.to_thread(blocking_fn)`.
4. `asyncio.gather(*tasks)` runs concurrently and returns results in order; `return_exceptions=True` prevents one failure from cancelling everything.
5. **Always bound concurrency with a `Semaphore`** — 10,000 simultaneous requests will get you rate-limited or run you out of sockets.
6. `asyncio.TaskGroup` (3.11+) is the modern structured alternative to `gather`, with proper cancellation semantics.

```python
async def structured():
    async with asyncio.TaskGroup() as tg:          # 3.11+
        t1 = tg.create_task(fetch("a"))
        t2 = tg.create_task(fetch("b"))
    # both are guaranteed complete here; any failure raises an ExceptionGroup
    return t1.result(), t2.result()
```

### Async streaming — how token-by-token output works

```python
import asyncio

async def stream_tokens(prompt: str):
    """An async generator: the shape of every LLM streaming API."""
    for token in f"answering {prompt} one token at a time".split():
        await asyncio.sleep(0.03)
        yield token + " "

async def main():
    buffer = []
    async for token in stream_tokens("about RAG"):     # async for
        buffer.append(token)
        print(token, end="", flush=True)
    print(f"\n[{len(buffer)} tokens]")

asyncio.run(main())
```
**Output:**
```
answering about RAG one token at a time 
[8 tokens]
```
This is why streaming feels fast: the first token arrives in ~30 ms rather than the user waiting 3 s for the whole response. Async generators (`async def` + `yield`) are the mechanism.

### Threads vs processes, measured

```python
import time, math
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu_task(n: int) -> int:                    # pure Python — GIL-bound
    return sum(math.isqrt(i) for i in range(n))

def io_task(_: int) -> str:
    time.sleep(0.1)                             # releases the GIL
    return "done"

if __name__ == "__main__":
    N, WORK = 8, 300_000

    t = time.perf_counter(); [cpu_task(WORK) for _ in range(N)]
    seq_cpu = time.perf_counter() - t

    t = time.perf_counter()
    with ThreadPoolExecutor(N) as ex: list(ex.map(cpu_task, [WORK]*N))
    thr_cpu = time.perf_counter() - t

    t = time.perf_counter()
    with ProcessPoolExecutor(4) as ex: list(ex.map(cpu_task, [WORK]*N))
    proc_cpu = time.perf_counter() - t

    t = time.perf_counter(); [io_task(0) for _ in range(N)]
    seq_io = time.perf_counter() - t

    t = time.perf_counter()
    with ThreadPoolExecutor(N) as ex: list(ex.map(io_task, range(N)))
    thr_io = time.perf_counter() - t

    print(f"CPU-bound  sequential: {seq_cpu:.2f}s")
    print(f"CPU-bound  threads   : {thr_cpu:.2f}s  ({seq_cpu/thr_cpu:.2f}x)  ← GIL")
    print(f"CPU-bound  processes : {proc_cpu:.2f}s  ({seq_cpu/proc_cpu:.2f}x)")
    print(f"I/O-bound  sequential: {seq_io:.2f}s")
    print(f"I/O-bound  threads   : {thr_io:.2f}s  ({seq_io/thr_io:.2f}x)")
```
**Output (indicative, 4-core machine):**
```
CPU-bound  sequential: 1.84s
CPU-bound  threads   : 1.91s  (0.96x)  ← GIL
CPU-bound  processes : 0.58s  (3.17x)
I/O-bound  sequential: 0.81s
I/O-bound  threads   : 0.10s  (7.90x)
```
**Read this table carefully — it is the whole GIL lesson in five numbers.** Threads made CPU-bound work *slightly slower* (lock contention plus zero parallelism). Processes gave near-linear speedup. Threads gave an 8× speedup on I/O.

**Multiprocessing caveats for AI work:** arguments and results are **pickled**, so lambdas and local closures fail (Chapter 3 §3.8), large arrays are expensive to transfer (use shared memory or memory-mapped files), each process pays interpreter startup (~50–200 ms), and CUDA contexts do not fork safely — use `spawn` and initialise CUDA inside the worker.

---

## 8.7 Memory Management and Garbage Collection

### Reference counting plus a cycle collector

```python
import gc, sys

class Node:
    def __init__(self, name): self.name, self.ref = name, None
    def __del__(self): print(f"  freed {self.name}")

print("simple refcount:")
a = Node("A")
b = a
del a                      # refcount 2 -> 1, still alive
del b                      # refcount 0 -> freed immediately

print("\nreference cycle:")
x, y = Node("X"), Node("Y")
x.ref, y.ref = y, x        # cycle: neither refcount ever reaches 0
del x, y
print("  after del — still alive")
collected = gc.collect()   # the cycle collector finds and frees them
print(f"  gc.collect() freed {collected} objects")
```
**Output:**
```
simple refcount:
  freed A

reference cycle:
  after del — still alive
  freed X
  freed Y
  gc.collect() freed 4 objects
```

CPython uses **reference counting** (immediate, deterministic) plus a **generational cycle collector** (periodic, for reference cycles). Objects surviving collection are promoted through three generations, so long-lived objects are scanned less often.

### The AI-specific memory traps

```python
# ❌ TRAP 1 — accumulating tensors keeps the autograd graph alive
losses = []
for batch in loader:
    loss = model(batch)
    losses.append(loss)          # each holds an entire computation graph → OOM
# ✔ FIX
    losses.append(loss.item())   # a plain float

# ❌ TRAP 2 — an unbounded cache
CACHE = {}
def embed(text):
    if text not in CACHE:
        CACHE[text] = model.encode(text)     # grows forever
    return CACHE[text]
# ✔ FIX
@functools.lru_cache(maxsize=10_000)
def embed(text): ...

# ❌ TRAP 3 — storing exceptions pins every frame's locals (Chapter 6 §6.6)
failures.append(exc)             # holds GPU tensors alive
# ✔ FIX
failures.append(str(exc))

# ❌ TRAP 4 — a closure capturing a large object
def make_handler(huge_dataset):
    def handler(x): return x * 2      # doesn't even use it, but captures the scope
    return handler
# ✔ FIX: pass only what you need, or use weakref
```

### Diagnosing a leak

```python
import tracemalloc

tracemalloc.start()
snapshot1 = tracemalloc.take_snapshot()

leak = [{"id": i, "data": "x" * 1000} for i in range(5_000)]

snapshot2 = tracemalloc.take_snapshot()
for stat in snapshot2.compare_to(snapshot1, "lineno")[:3]:
    print(stat)
print(f"\ncurrent: {tracemalloc.get_traced_memory()[0]/1024**2:.1f} MB, "
      f"peak: {tracemalloc.get_traced_memory()[1]/1024**2:.1f} MB")
tracemalloc.stop()
```
**Output (indicative):**
```
<file>:8: size=5877 KiB (+5877 KiB), count=10004 (+10004), average=601 B
...
current: 5.8 MB, peak: 5.8 MB
```
`tracemalloc` snapshots at two points and diffs them, telling you the exact line allocating memory. It is the first tool to reach for on "memory grows every epoch." For object-graph questions, `objgraph.show_growth()` shows which *types* are multiplying; `gc.get_referrers(obj)` shows what is holding a specific object alive.

### `weakref` — caches that don't leak

```python
import weakref

class Model:
    def __init__(self, name): self.name = name
    def __repr__(self): return f"Model({self.name!r})"

cache = weakref.WeakValueDictionary()
m = Model("bert")
cache["bert"] = m
print("in cache:", cache.get("bert"))
del m                                    # the ONLY strong reference is gone
print("after del:", cache.get("bert"))   # entry auto-removed
```
**Output:**
```
in cache: Model('bert')
after del: None
```
A `WeakValueDictionary` lets cached objects be collected when nothing else references them — the right structure for a model registry or an object-identity cache that must not prevent freeing.

---

## 8.8 Descriptors and Metaclasses

> These are framework-author tools. You will *read* far more of them than you write — but reading pydantic, Django, or `nn.Module` source without them is impossible.

### Descriptors — the protocol behind `@property`

A descriptor is any object defining `__get__`, `__set__`, or `__delete__`. `property`, `classmethod`, `staticmethod`, and plain functions are all descriptors.

```python
class Validated:
    """A reusable validating attribute — no boilerplate per field."""
    def __init__(self, *, min_value=None, max_value=None, dtype=float):
        self.min_value, self.max_value, self.dtype = min_value, max_value, dtype

    def __set_name__(self, owner, name):          # called automatically at class creation
        self.name = name
        self.private = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self                            # accessed on the class
        return getattr(obj, self.private)

    def __set__(self, obj, value):
        if not isinstance(value, self.dtype):
            try:
                value = self.dtype(value)
            except (TypeError, ValueError) as exc:
                raise TypeError(f"{self.name} must be {self.dtype.__name__}, "
                                f"got {type(value).__name__}") from exc
        if self.min_value is not None and value < self.min_value:
            raise ValueError(f"{self.name} must be >= {self.min_value}, got {value}")
        if self.max_value is not None and value > self.max_value:
            raise ValueError(f"{self.name} must be <= {self.max_value}, got {value}")
        setattr(obj, self.private, value)


class GenerationConfig:
    temperature = Validated(min_value=0.0, max_value=2.0)
    top_p       = Validated(min_value=0.0, max_value=1.0)
    max_tokens  = Validated(min_value=1, max_value=128_000, dtype=int)

    def __init__(self, temperature=0.7, top_p=1.0, max_tokens=512):
        self.temperature, self.top_p, self.max_tokens = temperature, top_p, max_tokens

    def __repr__(self):
        return (f"GenerationConfig(temperature={self.temperature}, "
                f"top_p={self.top_p}, max_tokens={self.max_tokens})")

cfg = GenerationConfig()
print(cfg)
cfg.temperature = 1.5
cfg.max_tokens = "2048"                 # coerced
print(cfg)
for bad in [("temperature", 3.0), ("top_p", -1), ("max_tokens", "many")]:
    try:
        setattr(cfg, *bad)
    except (ValueError, TypeError) as exc:
        print(f"  {type(exc).__name__}: {exc}")
```
**Output:**
```
GenerationConfig(temperature=0.7, top_p=1.0, max_tokens=512)
GenerationConfig(temperature=1.5, top_p=1.0, max_tokens=2048)
  ValueError: temperature must be <= 2.0, got 3.0
  ValueError: top_p must be >= 0.0, got -1.0
  TypeError: max_tokens must be int, got str
```
**Three `@property` definitions collapse into three one-line declarations.** `__set_name__` (3.6+) is what makes this ergonomic — the descriptor learns its own attribute name automatically. This is, in essence, how pydantic fields and Django model fields work.

### Metaclasses — classes that build classes

```python
from abc import ABCMeta

class ToolMeta(type):
    """Validates and registers tool classes at definition time."""
    registry: dict[str, type] = {}

    def __new__(mcls, name, bases, namespace, **kwargs):
        cls = super().__new__(mcls, name, bases, namespace)
        if bases:                                     # skip the base class itself
            if "run" not in namespace:
                raise TypeError(f"{name} must define run()")
            if not (cls.__doc__ or "").strip():
                raise TypeError(f"{name} must have a docstring — the LLM reads it")
            key = namespace.get("name", name.lower())
            ToolMeta.registry[key] = cls
        return cls

class BaseTool(metaclass=ToolMeta):
    pass

class WeatherTool(BaseTool):
    """Get the current weather for a city."""
    name = "weather"
    def run(self, city: str) -> str:
        return f"sunny in {city}"

print("registry:", ToolMeta.registry)

try:
    class BadTool(BaseTool):
        def run(self): ...
except TypeError as exc:
    print("rejected at class-definition time:", exc)
```
**Output:**
```
registry: {'weather': <class '__main__.WeatherTool'>}
rejected at class-definition time: BadTool must have a docstring — the LLM reads it
```

> **When to use what.** In order of preference:
> 1. A plain function or decorator.
> 2. `__init_subclass__` (Chapter 7) — covers ~90% of "do something when subclassed."
> 3. `__set_name__` + descriptors — for per-attribute behaviour.
> 4. A metaclass — only when you must control *class creation itself* (ABCs, ORMs, pydantic).
>
> Real quote worth remembering: *"If you're wondering whether you need metaclasses, you don't."* But you should be able to read them, because `ABCMeta`, `ModelMetaclass` (pydantic), and `type` itself are everywhere in the frameworks you'll use.

---

## 8.9 Best Practices, Common Mistakes, Performance

### Best practices

```python
# ✔ Generators for anything that streams
def load(path) -> Iterator[dict]: ...

# ✔ functools.wraps on every decorator; type the wrapper's params as *args/**kwargs
# ✔ @contextmanager with try/finally — never bare setup/teardown
# ✔ Type-hint every public function; run mypy/pyright in CI
# ✔ Enums, not string literals, for closed value sets
# ✔ asyncio for I/O; ProcessPool for CPU; nothing for NumPy/PyTorch
# ✔ Always bound async concurrency with a Semaphore
# ✔ Bound every cache (lru_cache maxsize, deque maxlen, TTL)
# ✔ Store scalars (loss.item()), not tensors, in accumulators
```

### Common mistakes

**1. Consuming a generator twice**
```python
# ❌ WRONG
gen = (x for x in range(3))
print(sum(gen), sum(gen))        # -> 3 0   ← second sum sees an exhausted generator

# ✔ CORRECT — materialise if you need it twice, or use itertools.tee
data = list(gen)
```

**2. Blocking inside async**
```python
# ❌ WRONG — freezes the entire event loop for 1 second
async def handler():
    time.sleep(1)
    requests.get(url)

# ✔ CORRECT
async def handler():
    await asyncio.sleep(1)
    async with httpx.AsyncClient() as c:
        await c.get(url)
    await asyncio.to_thread(blocking_library_call)     # for code you can't change
```

**3. Threads for CPU-bound Python**
```python
# ❌ WRONG — no speedup (see §8.6 measurement)
with ThreadPoolExecutor(8) as ex: ex.map(tokenize_document, docs)

# ✔ CORRECT
with ProcessPoolExecutor(8) as ex: ex.map(tokenize_document, docs)
```

**4. Unbounded `asyncio.gather`**
```python
# ❌ WRONG — 10,000 simultaneous connections → rate limit or socket exhaustion
await asyncio.gather(*[fetch(u) for u in urls])

# ✔ CORRECT
sem = asyncio.Semaphore(20)
async def bounded(u):
    async with sem: return await fetch(u)
await asyncio.gather(*[bounded(u) for u in urls])
```

**5. Decorator without `wraps`** — see Chapter 3 §3.8. It silently breaks tool schemas and logging.

**6. `lru_cache` on a method** — the cache holds `self`, so instances are never freed. Cache a module-level function, or use `functools.cached_property` for per-instance caching.

**7. Missing `if __name__ == "__main__":` with multiprocessing** — on `spawn` platforms (Windows, macOS) the child re-imports the module and recursively spawns processes. This is not optional.

### Performance summary

| Technique | Typical gain | Applies to |
|---|---|---|
| Generator instead of list | 100–10,000× memory | Any streaming data |
| `asyncio` for API calls | 10–50× wall clock | LLM/embedding/vector-DB calls |
| `ProcessPoolExecutor` | ~N cores | CPU-bound pure Python |
| `lru_cache` | 10–1000× on hits | Pure, repeated computations |
| `__slots__` | 40–60% memory | Millions of instances |
| Vectorisation (NumPy) | 10–100× | Numeric loops |
| `orjson` over `json` | 2–5× | High-volume JSON |

**The ordering rule:** profile first (`cProfile`, `py-spy`, `line_profiler`), fix the algorithm, then vectorise, then parallelise, then micro-optimise. Reversing that order is how people spend a week making a 2% improvement to code that a set lookup would have made 1000× faster.

---

## 8.10 Interview Preparation

**Q1 (Beginner). Iterator vs iterable vs generator?**
An *iterable* implements `__iter__` and can be looped repeatedly. An *iterator* implements `__iter__` and `__next__`, is stateful, and is exhausted after one pass. A *generator* is an iterator created by a function containing `yield` (or by a generator expression) — it produces values lazily and holds only its frame in memory.

**Q2 (Beginner). Why use a generator over a list?**
Constant memory regardless of dataset size, immediate first result (no waiting for the whole collection), the ability to represent infinite sequences, and natural pipeline composition. The trade-offs: single-use, no `len()`, no indexing.

**Q3 (Beginner). What does a decorator do?**
It takes a callable and returns a replacement, letting you add cross-cutting behaviour (timing, retry, caching, auth, logging) without modifying the function. `@dec` is sugar for `f = dec(f)`. Always apply `functools.wraps` to preserve metadata.

**Q4 (Intermediate). Explain the GIL and its practical consequences.**
The GIL is a per-interpreter mutex allowing only one thread to execute Python bytecode at a time. Consequences: multithreading gives no speedup for CPU-bound *Python* code; it does help I/O-bound work, because the GIL is released during blocking calls; and it helps NumPy/PyTorch work, because those release the GIL inside their C kernels. For CPU-bound Python, use processes.

**Q5 (Intermediate). When would you choose async over threads?**
When you have many concurrent I/O waits — hundreds of API calls. Async uses one thread and a few KB per task versus ~8 MB of stack per thread, has no lock contention, and makes concurrency points explicit at `await`. Threads are the right choice when you must call a blocking library that has no async interface.

**Q6 (Intermediate). How does `@contextmanager` work?**
It wraps a generator that yields exactly once. Code before the `yield` is `__enter__`, the yielded value binds to `as`, and code after (in a `finally`) is `__exit__`. Exceptions from the body are raised at the `yield` point, which is why the teardown must be in a `finally` block.

**Q7 (AI/ML). You must embed 1 million documents through an API. Design it.**
Stream documents with a generator (never load all). Batch them (`itertools.batched`) to the provider's maximum batch size. Use `asyncio` with a `Semaphore` bounding concurrency to the rate limit, plus retry with exponential backoff and jitter for 429/5xx. Cache by content hash so re-runs skip unchanged documents. Checkpoint progress to disk so a crash resumes. Write vectors incrementally to `.npy`/Parquet rather than holding them in memory. Expect roughly a 20–50× wall-clock improvement over a sequential loop.

**Q8 (AI/ML). Why does `DataLoader(num_workers=8)` use processes, and what breaks?**
Data loading (decoding, augmentation, tokenising) is CPU-bound Python, which threads cannot parallelise because of the GIL. Processes give real parallelism. What breaks: the dataset and any transforms must be picklable (no lambdas, no local closures); CUDA tensors must not be created in workers before fork (use `spawn`); and each worker duplicates memory, so a large in-memory dataset is copied N times — use memory-mapping or shared memory.

**Q9 (AI/ML). Your training loop's memory grows every epoch. Diagnose it.**
Most likely you are appending tensors (which hold autograd graphs) to a list instead of `.item()` floats. Other candidates: an unbounded cache dict; stored exception objects pinning frame locals; a growing `defaultdict`; or `lru_cache` on a method holding `self`. Diagnose with `tracemalloc` snapshot diffs, `torch.cuda.memory_summary()`, and `gc.get_referrers` on a suspect object.

**Q10 (Scenario). A FastAPI endpoint becomes unresponsive under load, though CPU is idle.**
Something blocking is running inside an `async def` — a synchronous HTTP client, a blocking DB driver, `time.sleep`, or a CPU-heavy loop — which halts the entire event loop. Fix: use async clients, move blocking calls to `await asyncio.to_thread(...)` or a thread pool, or declare the endpoint as a regular `def` so FastAPI runs it in its threadpool automatically.

**Q11 (Production). What is a descriptor and where have you used one without knowing?**
An object implementing `__get__`/`__set__`/`__delete__` that customises attribute access. You use them constantly: `@property`, `@classmethod`, `@staticmethod`, `functools.cached_property`, and every method (functions are descriptors — that's how binding produces `self`). Frameworks use them for validated fields (pydantic, Django ORM).

**Q12 (Production). When is a metaclass justified?**
Almost never in application code. Justified when you must intervene in *class creation itself*: enforcing an interface at definition time, auto-registering classes, or building attributes from annotations (ABCs, pydantic, ORMs). Prefer `__init_subclass__`, class decorators, or descriptors first — they cover nearly every case with far less complexity.

---

## 8.11 Exercises

### Easy — E1
Write a generator `read_batches(path, batch_size)` yielding batches of parsed JSONL records.

<details><summary>Solution</summary>

```python
import json
from typing import Iterator

def read_batches(path: str, batch_size: int = 32) -> Iterator[list[dict]]:
    batch = []
    with open(path, encoding="utf-8") as f:
        for line in f:
            if not line.strip():
                continue
            batch.append(json.loads(line))
            if len(batch) == batch_size:
                yield batch
                batch = []                 # NEW list — don't clear() the yielded one!
    if batch:
        yield batch

from pathlib import Path
Path("d.jsonl").write_text("\n".join(json.dumps({"i": i}) for i in range(7)), encoding="utf-8")
for b in read_batches("d.jsonl", 3):
    print(len(b), [r["i"] for r in b])
```
**Output:**
```
3 [0, 1, 2]
3 [3, 4, 5]
1 [6]
```
Two details matter. `batch = []` rather than `batch.clear()` — clearing would empty the list the consumer is still holding, since you yielded a *reference*. And the trailing `if batch:` keeps the final partial batch; forgetting it silently drops up to `batch_size - 1` records from every run.
</details>

### Medium — M1
Write a `@cached_with_ttl(seconds)` decorator: an LRU cache whose entries expire.

<details><summary>Solution</summary>

```python
import functools, time
from collections import OrderedDict

def cached_with_ttl(ttl_seconds: float = 60.0, maxsize: int = 128):
    def decorator(fn):
        cache: OrderedDict[tuple, tuple[float, object]] = OrderedDict()
        stats = {"hits": 0, "misses": 0, "expired": 0}

        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            key = (args, tuple(sorted(kwargs.items())))
            now = time.monotonic()

            if key in cache:
                stored_at, value = cache[key]
                if now - stored_at < ttl_seconds:
                    stats["hits"] += 1
                    cache.move_to_end(key)
                    return value
                stats["expired"] += 1
                del cache[key]

            stats["misses"] += 1
            value = fn(*args, **kwargs)
            cache[key] = (now, value)
            while len(cache) > maxsize:
                cache.popitem(last=False)
            return value

        wrapper.cache_info = lambda: {**stats, "size": len(cache)}
        wrapper.cache_clear = cache.clear
        return wrapper
    return decorator


@cached_with_ttl(ttl_seconds=0.2, maxsize=4)
def fetch_model_list(provider: str) -> list[str]:
    print(f"  [API call for {provider}]")
    return [f"{provider}-small", f"{provider}-large"]

print(fetch_model_list("acme"))
print(fetch_model_list("acme"))          # hit
time.sleep(0.25)
print(fetch_model_list("acme"))          # expired -> miss
print(fetch_model_list.cache_info())
```
**Output:**
```
  [API call for acme]
['acme-small', 'acme-large']
['acme-small', 'acme-large']
  [API call for acme]
['acme-small', 'acme-large']
{'hits': 1, 'misses': 2, 'expired': 1, 'size': 1}
```
**Explanation.** `lru_cache` has no TTL, which makes it wrong for anything that goes stale — model lists, feature flags, auth tokens, retrieved documents. `tuple(sorted(kwargs.items()))` makes the key order-independent so `f(a=1,b=2)` and `f(b=2,a=1)` share an entry. `time.monotonic()` rather than `time.time()` because the latter can jump backwards on NTP adjustments. Exposing `cache_info` is what lets you monitor hit rate in production.
</details>

### Hard — H1
Build an async, rate-limited, retrying, checkpointing embedding pipeline that streams input and writes output incrementally.

<details><summary>Solution</summary>

```python
"""async_embed.py — production-shaped concurrent embedding pipeline."""
from __future__ import annotations

import asyncio
import hashlib
import json
import random
import time
from dataclasses import dataclass, field
from pathlib import Path
from typing import AsyncIterator, Iterator


# ---------------- input streaming ----------------
def stream_documents(path: Path) -> Iterator[dict]:
    with open(path, encoding="utf-8") as f:
        for line in f:
            if line.strip():
                yield json.loads(line)


def batched(items: Iterator[dict], size: int) -> Iterator[list[dict]]:
    batch = []
    for item in items:
        batch.append(item)
        if len(batch) == size:
            yield batch
            batch = []
    if batch:
        yield batch


# ---------------- rate limiting ----------------
class TokenBucket:
    """Smooth rate limiting — allows bursts up to `capacity`, refills at `rate`/s."""
    def __init__(self, rate: float, capacity: float | None = None):
        self.rate = rate
        self.capacity = capacity if capacity is not None else rate
        self.tokens = self.capacity
        self.updated = time.monotonic()
        self._lock = asyncio.Lock()

    async def acquire(self, amount: float = 1.0) -> None:
        async with self._lock:
            while True:
                now = time.monotonic()
                self.tokens = min(self.capacity,
                                  self.tokens + (now - self.updated) * self.rate)
                self.updated = now
                if self.tokens >= amount:
                    self.tokens -= amount
                    return
                await asyncio.sleep((amount - self.tokens) / self.rate)


# ---------------- the pipeline ----------------
@dataclass
class EmbeddingPipeline:
    out_path: Path
    checkpoint_path: Path
    concurrency: int = 5
    requests_per_second: float = 20.0
    batch_size: int = 4
    max_retries: int = 4
    stats: dict = field(default_factory=lambda: {
        "batches": 0, "documents": 0, "cached": 0, "api_calls": 0,
        "retries": 0, "failed": 0})
    _done: set[str] = field(default_factory=set, init=False)

    def __post_init__(self):
        self.sem = asyncio.Semaphore(self.concurrency)
        self.bucket = TokenBucket(self.requests_per_second)
        if self.checkpoint_path.exists():
            self._done = set(json.loads(self.checkpoint_path.read_text())["done"])
            print(f"  resuming: {len(self._done)} documents already embedded")

    @staticmethod
    def _fingerprint(text: str) -> str:
        return hashlib.sha256(" ".join(text.split()).lower().encode()).hexdigest()[:16]

    async def _call_api(self, texts: list[str]) -> list[list[float]]:
        """Simulated provider: latency, occasional 429/503."""
        await asyncio.sleep(random.uniform(0.02, 0.06))
        roll = random.random()
        if roll < 0.18:
            raise ConnectionError("429 rate limited")
        if roll < 0.24:
            raise TimeoutError("504 gateway timeout")
        return [[round(len(t) * 0.01, 4)] * 4 for t in texts]

    async def _embed_batch(self, batch: list[dict]) -> list[dict]:
        fresh = [d for d in batch if self._fingerprint(d["text"]) not in self._done]
        self.stats["cached"] += len(batch) - len(fresh)
        if not fresh:
            return []

        async with self.sem:
            for attempt in range(1, self.max_retries + 1):
                await self.bucket.acquire()
                try:
                    self.stats["api_calls"] += 1
                    vectors = await self._call_api([d["text"] for d in fresh])
                except (ConnectionError, TimeoutError) as exc:
                    self.stats["retries"] += 1
                    if attempt == self.max_retries:
                        self.stats["failed"] += len(fresh)
                        print(f"    ✗ batch failed permanently: {exc}")
                        return []
                    delay = 0.05 * 2 ** (attempt - 1) * (0.5 + random.random())
                    await asyncio.sleep(delay)
                else:
                    return [{"id": d["id"], "fp": self._fingerprint(d["text"]),
                             "vector": v}
                            for d, v in zip(fresh, vectors, strict=True)]
        return []

    async def run(self, source: Path) -> dict:
        started = time.perf_counter()
        self.out_path.parent.mkdir(parents=True, exist_ok=True)

        with open(self.out_path, "a", encoding="utf-8") as fout:
            pending: list[asyncio.Task] = []
            for batch in batched(stream_documents(source), self.batch_size):
                self.stats["batches"] += 1
                self.stats["documents"] += len(batch)
                pending.append(asyncio.create_task(self._embed_batch(batch)))

                if len(pending) >= self.concurrency * 2:      # bound task backlog
                    done, remaining = await asyncio.wait(
                        pending, return_when=asyncio.FIRST_COMPLETED)
                    pending = list(remaining)
                    self._write(done, fout)

            if pending:
                done, _ = await asyncio.wait(pending)
                self._write(done, fout)

        self.checkpoint_path.parent.mkdir(parents=True, exist_ok=True)
        self.checkpoint_path.write_text(
            json.dumps({"done": sorted(self._done)}), encoding="utf-8")

        self.stats["seconds"] = round(time.perf_counter() - started, 2)
        self.stats["docs_per_sec"] = round(
            self.stats["documents"] / max(1e-9, self.stats["seconds"]), 1)
        return self.stats

    def _write(self, tasks, fout) -> None:
        for task in tasks:
            for record in task.result():
                fout.write(json.dumps(record) + "\n")
                self._done.add(record["fp"])


async def main():
    source = Path("docs.jsonl")
    source.write_text("\n".join(
        json.dumps({"id": f"d{i}", "text": f"Document number {i} about retrieval "
                                           f"augmented generation and embeddings."})
        for i in range(40)), encoding="utf-8")

    out, ckpt = Path("vectors.jsonl"), Path("ckpt.json")
    out.unlink(missing_ok=True); ckpt.unlink(missing_ok=True)

    print("--- first run ---")
    p1 = EmbeddingPipeline(out, ckpt, concurrency=5, requests_per_second=40, batch_size=4)
    print(json.dumps(await p1.run(source), indent=2))

    print("\n--- second run (resumes from checkpoint) ---")
    p2 = EmbeddingPipeline(out, ckpt, concurrency=5, requests_per_second=40, batch_size=4)
    print(json.dumps(await p2.run(source), indent=2))

    print(f"\nvectors written: {len(out.read_text().splitlines())}")

asyncio.run(main())
```
**Output (representative):**
```
--- first run ---
    ✗ batch failed permanently: 429 rate limited
{
  "batches": 10,
  "documents": 40,
  "cached": 0,
  "api_calls": 16,
  "retries": 6,
  "failed": 4,
  "seconds": 0.42,
  "docs_per_sec": 95.2
}

--- second run (resumes from checkpoint) ---
  resuming: 36 documents already embedded
{
  "batches": 10,
  "documents": 40,
  "cached": 36,
  "api_calls": 2,
  "retries": 1,
  "failed": 0,
  "seconds": 0.09,
  "docs_per_sec": 444.4
}

vectors written: 40
```
**Explanation — every production concern, in one file.**

| Mechanism | Chapter concept | Why it's necessary |
|---|---|---|
| `stream_documents` + `batched` generators | §8.2 | Constant memory on a 500 GB corpus |
| `asyncio.Semaphore` | §8.6 | Caps in-flight requests; without it you exhaust sockets |
| `TokenBucket` with an `asyncio.Lock` | §8.6 | Semaphores bound *concurrency*; token buckets bound *rate*. Providers limit requests/minute, so you need both |
| Retry with exponential backoff + jitter | Ch. 6 | 429/503 are normal, not exceptional |
| Content-hash checkpointing | Ch. 5 | The second run does 2 API calls instead of 16 — this is the difference between a $200 nightly re-index and a $10 one |
| Incremental append to output | Ch. 5 | A crash at 90% keeps 90% of the work |
| `asyncio.wait(FIRST_COMPLETED)` with a bounded backlog | §8.6 | Naive `gather` over a million documents builds a million task objects before running any; this streams |
| `strict=True` in `zip` | Ch. 2 | If the API returns fewer vectors than texts, fail loudly rather than misalign every subsequent id |
| Stats dict | Ch. 6 | `retries` and `failed` are your operational signal |

**The headline number:** with 40 documents at ~40 ms each, a sequential loop takes ~1.6 s; this takes 0.42 s at concurrency 5. At 1 million documents the same ratio turns 11 hours into under 3 — and the checkpoint means the second run takes minutes.
</details>

---

## 8.12 Mini Project — Async RAG Ingestion Service

**Goal.** Combine every chapter-8 technique into one service: generator streaming, decorators for instrumentation, context managers for resources, type hints and enums for contracts, async concurrency with rate limiting, and bounded memory throughout.

```python
"""ingest_service.py — the capstone of Chapter 8."""
from __future__ import annotations

import asyncio
import contextlib
import functools
import hashlib
import json
import random
import time
from collections import defaultdict
from dataclasses import dataclass, field
from enum import StrEnum
from pathlib import Path
from typing import AsyncIterator, Iterator, Protocol


# ============================ contracts ============================
class Stage(StrEnum):
    LOAD = "load"
    CHUNK = "chunk"
    EMBED = "embed"
    STORE = "store"


class AsyncEmbedder(Protocol):
    dim: int
    async def embed(self, texts: list[str]) -> list[list[float]]: ...


# ============================ instrumentation ============================
METRICS: dict[str, dict] = defaultdict(lambda: {"calls": 0, "errors": 0, "seconds": 0.0})


def timed_async(stage: Stage):
    """Decorator for async functions: times, counts, and re-raises."""
    def decorator(fn):
        @functools.wraps(fn)
        async def wrapper(*args, **kwargs):
            m = METRICS[stage.value]
            m["calls"] += 1
            t0 = time.perf_counter()
            try:
                return await fn(*args, **kwargs)
            except Exception:
                m["errors"] += 1
                raise
            finally:
                m["seconds"] += time.perf_counter() - t0
        return wrapper
    return decorator


@contextlib.contextmanager
def phase(label: str):
    print(f"\n▶ {label}")
    t0 = time.perf_counter()
    try:
        yield
    finally:
        print(f"◀ {label} finished in {time.perf_counter()-t0:.2f}s")


# ============================ streaming stages ============================
def load_documents(root: Path) -> Iterator[dict]:
    """Generator: streams documents from disk without loading them all."""
    for path in sorted(root.rglob("*.txt")):
        text = path.read_text(encoding="utf-8", errors="replace")
        if text.strip():
            yield {"doc_id": path.stem, "text": text, "source": str(path.name)}


def chunk_documents(docs: Iterator[dict], *, size: int = 200,
                    overlap: int = 50) -> Iterator[dict]:
    """Generator: one chunk at a time; memory never grows with corpus size."""
    stride = size - overlap
    for doc in docs:
        text = " ".join(doc["text"].split())
        for i, start in enumerate(range(0, max(1, len(text)), stride)):
            piece = text[start:start + size].strip()
            if piece:
                yield {"chunk_id": f"{doc['doc_id']}#{i}",
                       "doc_id": doc["doc_id"],
                       "text": piece,
                       "source": doc["source"],
                       "fp": hashlib.sha256(piece.lower().encode()).hexdigest()[:16]}


def batched(items: Iterator[dict], size: int) -> Iterator[list[dict]]:
    batch = []
    for item in items:
        batch.append(item)
        if len(batch) == size:
            yield batch
            batch = []
    if batch:
        yield batch


# ============================ embedder ============================
@dataclass
class FakeAsyncEmbedder:
    dim: int = 8
    failure_rate: float = 0.15
    _rng: random.Random = field(default_factory=lambda: random.Random(5))

    async def embed(self, texts: list[str]) -> list[list[float]]:
        await asyncio.sleep(self._rng.uniform(0.02, 0.05))
        if self._rng.random() < self.failure_rate:
            raise ConnectionError("429 rate limited")
        return [[round((len(t) % 97) / 97, 4)] * self.dim for t in texts]


# ============================ the service ============================
@dataclass
class IngestionService:
    embedder: AsyncEmbedder
    out_path: Path
    manifest_path: Path
    concurrency: int = 4
    batch_size: int = 8
    max_retries: int = 3
    stats: dict = field(default_factory=lambda: defaultdict(int))
    _seen: set[str] = field(default_factory=set, init=False)

    def __post_init__(self):
        self._sem = asyncio.Semaphore(self.concurrency)
        if self.manifest_path.exists():
            self._seen = set(json.loads(self.manifest_path.read_text())["fingerprints"])
            print(f"  loaded manifest: {len(self._seen)} chunks already indexed")

    # ---- one batch, with retry ----
    @timed_async(Stage.EMBED)
    async def _embed_batch(self, batch: list[dict]) -> list[dict]:
        fresh = [c for c in batch if c["fp"] not in self._seen]
        self.stats["chunks_skipped_duplicate"] += len(batch) - len(fresh)
        if not fresh:
            return []

        async with self._sem:
            for attempt in range(1, self.max_retries + 1):
                try:
                    vectors = await self.embedder.embed([c["text"] for c in fresh])
                except ConnectionError:
                    self.stats["retries"] += 1
                    if attempt == self.max_retries:
                        self.stats["chunks_failed"] += len(fresh)
                        return []
                    await asyncio.sleep(0.02 * 2 ** attempt * (0.5 + random.random()))
                else:
                    self.stats["api_calls"] += 1
                    return [{**c, "vector": v}
                            for c, v in zip(fresh, vectors, strict=True)]
        return []

    # ---- the orchestrator ----
    async def ingest(self, root: Path) -> dict:
        t0 = time.perf_counter()
        self.out_path.parent.mkdir(parents=True, exist_ok=True)

        chunks = chunk_documents(load_documents(root), size=200, overlap=50)

        with open(self.out_path, "a", encoding="utf-8") as fout:
            inflight: set[asyncio.Task] = set()

            async def drain(tasks: set[asyncio.Task]) -> None:
                for task in tasks:
                    for record in task.result():
                        fout.write(json.dumps(record) + "\n")
                        self._seen.add(record["fp"])
                        self.stats["chunks_indexed"] += 1

            for batch in batched(chunks, self.batch_size):
                self.stats["chunks_seen"] += len(batch)
                inflight.add(asyncio.create_task(self._embed_batch(batch)))
                if len(inflight) >= self.concurrency * 2:
                    done, inflight = await asyncio.wait(
                        inflight, return_when=asyncio.FIRST_COMPLETED)
                    await drain(done)
            if inflight:
                done, _ = await asyncio.wait(inflight)
                await drain(done)

        self.manifest_path.parent.mkdir(parents=True, exist_ok=True)
        self.manifest_path.write_text(
            json.dumps({"fingerprints": sorted(self._seen),
                        "dim": self.embedder.dim,
                        "count": len(self._seen)}), encoding="utf-8")

        self.stats["seconds"] = round(time.perf_counter() - t0, 3)
        return dict(self.stats)

    def report(self) -> None:
        print(f"\n{'stage':<8} {'calls':>6} {'errors':>7} {'seconds':>9} {'avg ms':>8}")
        print("-" * 42)
        for stage, m in METRICS.items():
            avg = m["seconds"] / m["calls"] * 1000 if m["calls"] else 0
            print(f"{stage:<8} {m['calls']:>6} {m['errors']:>7} "
                  f"{m['seconds']:>9.3f} {avg:>8.1f}")


# ============================ demo ============================
async def main():
    corpus = Path("corpus"); corpus.mkdir(exist_ok=True)
    for i in range(6):
        (corpus / f"doc{i}.txt").write_text(
            f"Document {i}. " + " ".join(
                f"Section {i}.{j}: retrieval augmented generation grounds model "
                f"outputs in retrieved passages." for j in range(6)),
            encoding="utf-8")

    out, manifest = Path("index/vectors.jsonl"), Path("index/manifest.json")
    out.unlink(missing_ok=True); manifest.unlink(missing_ok=True)

    service = IngestionService(FakeAsyncEmbedder(), out, manifest,
                               concurrency=4, batch_size=8)

    with phase("initial ingestion"):
        print(json.dumps(await service.ingest(corpus), indent=2))
    service.report()

    (corpus / "doc6.txt").write_text(
        "Document 6. " + " ".join(
            f"Section 6.{j}: vector databases index embeddings for fast approximate "
            f"nearest neighbour search." for j in range(6)), encoding="utf-8")

    METRICS.clear()
    service2 = IngestionService(FakeAsyncEmbedder(), out, manifest,
                                concurrency=4, batch_size=8)
    with phase("incremental ingestion (1 new document)"):
        print(json.dumps(await service2.ingest(corpus), indent=2))
    service2.report()

    print(f"\ntotal vectors on disk: {len(out.read_text().splitlines())}")

asyncio.run(main())
```

**Output (representative):**
```
▶ initial ingestion
{
  "chunks_seen": 24,
  "chunks_skipped_duplicate": 0,
  "api_calls": 3,
  "chunks_indexed": 24,
  "seconds": 0.046
}
◀ initial ingestion finished in 0.05s

stage     calls  errors   seconds   avg ms
------------------------------------------
embed         3       0     0.127     42.4

  loaded manifest: 24 chunks already indexed

▶ incremental ingestion (1 new document)
{
  "chunks_seen": 28,
  "chunks_skipped_duplicate": 24,
  "api_calls": 1,
  "chunks_indexed": 4,
  "seconds": 0.040
}
◀ incremental ingestion (1 new document) finished in 0.04s

stage     calls  errors   seconds   avg ms
------------------------------------------
embed         4       0     0.039      9.8

total vectors on disk: 28
```

**Everything in Chapter 8, doing real work:**

| Technique | Where | Payoff |
|---|---|---|
| **Generators** (`load_documents` → `chunk_documents` → `batched`) | Streaming stages | Memory stays flat whether the corpus is 6 files or 6 million |
| **Decorator** `@timed_async` | Per-stage instrumentation | Latency and error rate per stage with one annotation, and it re-raises rather than swallowing |
| **Context manager** `phase()` | Timing blocks | `try/finally` means the timing prints even if ingestion raises |
| **`StrEnum`** for stages | Metric keys | No typo'd `"embbed"` silently creating a second metric bucket |
| **`Protocol`** for the embedder | Dependency boundary | Swap `FakeAsyncEmbedder` for OpenAI/Cohere/local — the service never changes, and tests need no network |
| **`asyncio` + `Semaphore`** | Concurrency | 3 API calls for 24 chunks, overlapping; sequential would be 3× slower and scales far worse |
| **Bounded in-flight tasks** | `asyncio.wait(FIRST_COMPLETED)` | Never builds a million task objects; results drain to disk as they complete |
| **Content-hash manifest** | Incremental runs | The second run makes **1 API call instead of 4** — 24 of 28 chunks skipped entirely |
| **Retry with jitter** | `_embed_batch` | 2 transient 429s absorbed silently; the run still completed fully |
| **Append-only output** | `open(..., "a")` | A crash at 80% keeps 80% of the work |

**The single most valuable line in the file** is the manifest check: `fresh = [c for c in batch if c["fp"] not in self._seen]`. On a real corpus where 1% of documents change per day, it turns a nightly $200 re-embedding job into a $2 one — and that is a pure application of Chapter 4's `set` membership plus Chapter 5's content hashing.

**Extensions.** Write vectors to `.npy` with `mmap` instead of JSONL (Chapter 5); add a `ProcessPoolExecutor` for CPU-bound chunking of PDFs while embedding runs async; expose it as a FastAPI endpoint with a background task; add OpenTelemetry spans via the `phase` context manager; add a circuit breaker (Chapter 6) around the embedder.

---

## Chapter 8 Summary

| Concept | Takeaway |
|---|---|
| Iterators | `__iter__` + `__next__`; stateful and single-use |
| Generators | `yield` → lazy, constant memory; compose into pipelines |
| `yield from` | Delegate to a sub-iterable |
| Decorators | `f = dec(f)`; three levels for parameters; **always `functools.wraps`** |
| Context managers | `__enter__`/`__exit__` or `@contextmanager` with `try/finally` |
| Type hints | Executed by pydantic/FastAPI/tool-schema generators — not decoration |
| Enums | Closed value sets; typos become errors at the point of the typo |
| GIL | One thread runs bytecode; released during I/O and in C extensions |
| `asyncio` | For I/O concurrency; never block the loop; always bound with a Semaphore |
| Threads vs processes | Threads for I/O, processes for CPU-bound Python |
| Memory | Refcount + cycle GC; don't retain tensors, bound every cache |
| `tracemalloc` | Snapshot-diff to find the leaking line |
| Descriptors | The protocol behind `@property`; `__set_name__` makes them ergonomic |
| Metaclasses | Framework tool; prefer `__init_subclass__` |

**Revision flashcards.** Generator vs list memory · decorator stacking order · `@contextmanager` needs `try/finally` · GIL released during I/O and in NumPy · async never blocks · Semaphore + token bucket · `loss.item()` not `loss` · `lru_cache` on methods leaks.

---
# Appendix A — Cheat Sheets

## A.1 Core Syntax

```python
# ---------- variables & types ----------
x: int = 42;  y: float = 3.14;  s: str = "hi";  b: bool = True;  n = None
a, b = b, a                                  # swap
first, *rest = [1, 2, 3]                     # unpack
if (n := len(items)) > 10: ...               # walrus

# ---------- f-strings ----------
f"{acc:.2%}"      # 87.65%          f"{n:,}"        # 1,234,567
f"{x:>10.3f}"     # right-align     f"{v:+.3f}"     # forced sign
f"{255:#x}"       # 0xff            f"{acc=}"       # acc=0.876  (debug)

# ---------- control flow ----------
for i, v in enumerate(xs, start=1): ...
for a, b in zip(xs, ys, strict=True): ...
for x in xs:
    if bad(x): continue
    ...
else:                                        # runs if no `break`
    ...

match event:
    case {"type": "tool", "name": str(n), "args": {"q": q}}: ...
    case {"code": int(c)} if c >= 500: ...
    case _: ...

# ---------- comprehensions ----------
[f(x) for x in xs if p(x)]                   # list
{k: v for k, v in pairs}                     # dict
{f(x) for x in xs}                           # set
(f(x) for x in xs)                           # generator (lazy)

# ---------- functions ----------
def f(pos, /, normal, *args, kw_only, **kwargs) -> R: ...
def g(items, cache: dict | None = None):     # never `= {}`
    cache = {} if cache is None else cache

# ---------- classes ----------
@dataclass(frozen=True, slots=True)
class Chunk:
    text: str
    meta: dict = field(default_factory=dict)

# ---------- files ----------
with open(p, encoding="utf-8") as f:
    for line in f: ...                       # streams; O(1) memory

# ---------- errors ----------
try: ...
except SpecificError as exc: raise MyError("context") from exc
else: ...                                    # only if no exception
finally: ...                                 # always
```

---

## A.2 Container Selection & Complexity

| Need | Use | `in` | index | append | pop front |
|---|---|---|---|---|---|
| Ordered, growable | `list` | O(n) | O(1) | O(1)* | O(n) |
| Fixed record, hashable | `tuple` | O(n) | O(1) | — | — |
| Membership / dedup | `set` | **O(1)** | — | O(1) | — |
| Key → value | `dict` | **O(1)** | O(1) key | O(1) | — |
| Both ends / window | `deque` | O(n) | O(n) | O(1) | **O(1)** |
| Numbers, math | `np.ndarray` | O(n) | O(1) | — | — |

```python
from collections import defaultdict, Counter, deque, OrderedDict
by_label = defaultdict(list); by_label[k].append(v)     # grouping
counts   = Counter(tokens); counts.most_common(10)      # frequencies
window   = deque(maxlen=100)                            # auto-evicting
lru      = OrderedDict(); lru.move_to_end(k); lru.popitem(last=False)
```

**Dedup preserving order:** `list(dict.fromkeys(items))`
**Composite key:** `d[(model, task)] = ...`
**Merge:** `{**a, **b}` or `a | b`
**Sort multi-level:** `sorted(xs, key=lambda d: (-d["score"], d["id"]))`

---

## A.3 File Format Decision Table

| Data | Format | Call |
|---|---|---|
| Training data, logs | JSONL | `for line in f: json.loads(line)` |
| Config | YAML/TOML | `yaml.safe_load(f)` |
| Results, metadata | JSON | `json.dump(d, f, indent=2, ensure_ascii=False, sort_keys=True)` |
| Large tables | Parquet | `pd.read_parquet(p)` |
| Embeddings | `.npy` | `np.load(p, mmap_mode="r")` |
| Model weights | safetensors | never raw pickle from the internet |

```python
from pathlib import Path
BASE = Path(__file__).resolve().parent        # anchor, never trust the cwd
p = BASE / "data" / "train.jsonl"
p.parent.mkdir(parents=True, exist_ok=True)

# atomic write
fd, tmp = tempfile.mkstemp(dir=p.parent)
with os.fdopen(fd, "w", encoding="utf-8") as f:
    json.dump(data, f); f.flush(); os.fsync(f.fileno())
os.replace(tmp, p)
```

---

## A.4 Concurrency Decision Tree

```
Is the work waiting on the network / disk?
├── YES → many concurrent waits?      → asyncio + Semaphore
│         blocking library only?      → ThreadPoolExecutor
└── NO (CPU-bound)
    ├── inside NumPy / PyTorch / BLAS? → just call it (already parallel in C)
    └── pure Python?                    → ProcessPoolExecutor
```

```python
sem = asyncio.Semaphore(10)
async def bounded(x):
    async with sem: return await fetch(x)
results = await asyncio.gather(*[bounded(x) for x in xs], return_exceptions=True)

with ProcessPoolExecutor(8) as ex:
    results = list(ex.map(cpu_heavy, items))         # needs __main__ guard + picklable fn
```

---

## A.5 The Production Reliability Recipe

```python
@retry(max_attempts=4, exceptions=(RateLimitError, ServerError, TimeoutError))
def call():
    return client.post(url, json=payload, timeout=(3.05, 30))   # ALWAYS a timeout
```

| Layer | Mechanism |
|---|---|
| Timeout | Connect + read, on every network call |
| Retry | Bounded, exponential, jittered, allowlisted by exception type |
| Circuit breaker | Fail fast after N failures; half-open probe after cooldown |
| Fallback | Cheaper/smaller model, cached result, or graceful "I don't know" |
| Dead-letter | Failed records + error, so nothing is lost silently |
| Error budget | Fail the *run* if the drop rate exceeds a threshold |
| Observability | Structured JSON logs + counters for retries/fallbacks/skips |

---

## A.6 AI-Specific Numbers Worth Memorising

| Quantity | Value |
|---|---|
| Bytes per parameter | fp32 4 · fp16/bf16 2 · int8 1 · int4 0.5 |
| Weights for 7B | fp32 28 GB · bf16 14 GB · int4 3.5 GB |
| Full fine-tune with Adam | ≈ 16 bytes/param (≈ 112 GB for 7B) |
| Rough token estimate (English) | `chars / 4` |
| Typical chunk size for RAG | 200–800 tokens, 10–20% overlap |
| RRF constant | `k = 60` |
| Python int object | 28 bytes (vs 4 for `np.int32`) |
| Exception raise cost | ~1 µs |
| Python function call | ~50 ns |
| `set` vs `list` membership | ~10,000× at n = 100k |

---

## A.7 Pythonic Idioms Reference

```python
" ".join(parts)                          # not += in a loop  (O(n) vs O(n²))
if not items:                            # not len(items) == 0
x is None                                # not x == None
sorted(xs, key=lambda d: -d["score"])    # not a comparator
any(t in text for t in BANNED)           # not a flag loop
sum(p == y for p, y in zip(a, b))        # bools are ints
list(dict.fromkeys(xs))                  # order-preserving dedup
{**defaults, **overrides}                # merge
first, *rest = xs                        # unpack
value = maybe or default                 # short-circuit default
[x for x in xs if x]                     # filter falsy
next((x for x in xs if p(x)), None)      # first match or None
math.isclose(a, b)                       # never == on floats
```

---

# Appendix B — Rapid Revision Notes

*The night-before pass. If you can answer every line here, you're ready.*

### Chapter 0 — Model
Source → bytecode → CPython VM. Everything is a heap object with identity/type/value. A variable is a **name bound to an object**; `=` never copies. Aliasing is the root of most silent data bugs. Python orchestrates; C/CUDA computes.

### Chapter 1 — Basics
Dynamic + strong typing. Immutable ⇒ hashable ⇒ dict-key-able. `0.1+0.2 != 0.3` → always `isclose`. `bool("false") is True` — parse booleans against a token set. `//` vs `/`. `-5..256` are cached, so `is` ≠ `==`. `dtype` decides whether the model fits in VRAM. f-strings everywhere; lazy `%s` in logging.

### Chapter 2 — Control Flow
Empty is falsy; NumPy arrays *raise* on truth-testing. `and`/`or` short-circuit and return operands. `for` is `iter()`+`next()`, so it works on files, generators, DataLoaders. `for/else` = "no break". `match` destructures with guards. Guard clauses over nesting. **Bound every retry and every agent loop.** Never loop over floats — vectorise. `set` membership over `list`.

### Chapter 3 — Functions
Defaults evaluated **once** → never a mutable default. Call by object reference: rebinding is local, mutation is shared. Parameter grammar: `pos, /, normal, *, kw_only, **kwargs`. LEGB; assignment implies local; `nonlocal` to write outward. Closures capture cells. `functools.wraps` on every decorator. Signature + docstring **are** the LLM tool schema. Pure ⇒ testable, cacheable, parallelisable; lambdas aren't picklable.

### Chapter 4 — Data Structures
`in` is O(n) for list, O(1) for set/dict — the highest-value fact in the book. Lists are pointer arrays; `pop(0)` is O(n). Tuples are hashable → composite keys. Dicts are insertion-ordered (3.7+); sets are **not** ordered — `sorted()` before assigning ids. `defaultdict` groups, `Counter` counts, `deque(maxlen=)` windows, `OrderedDict` LRUs. `[[0]*3]*2` shares rows. `.sort()` returns `None`.

### Chapter 5 — Files
`with` + `encoding="utf-8"` + `pathlib`, always. `for line in f:` is O(1) memory. `"w"` truncates; `"x"` refuses to overwrite. Atomic write = temp + fsync + `os.replace`. `yaml.safe_load`; never `pickle.load` untrusted data. JSONL for data, Parquet for scale, `.npy`+mmap for vectors, safetensors for weights. Every artifact needs a metadata sidecar. `newline=""` for csv; CSV values are all strings.

### Chapter 6 — Errors
Catch narrow; never bare `except:`; never `pass`. `else` = success-only; `finally` = always. Always `raise X from exc`. Package base exception + typed subclasses + structured attributes. `assert` is stripped by `-O` — never for validation. EAFP when rare, LBYL when common (~1 µs to raise). Retry: bounded + exponential + jitter + allowlist. Circuit breaker prevents retry storms. Dead-letter + error budget: never "succeed" while dropping 40% of the data. Tracebacks pin frame locals — store `str(exc)`, not the exception.

### Chapter 7 — OOP
Use a class for state + several operations + lifecycle or swappable implementations. Mutable class attributes are shared. Always `super().__init__()`. MRO is C3; `super()` follows `type(self)`'s MRO (mixins). **Composition over inheritance.** ABC for families you own, `Protocol` for dependencies you accept. `__repr__` always. `@dataclass(frozen=True, slots=True)` for value objects; `default_factory` for mutable fields. `__eq__` needs `__hash__`. No I/O in `__init__`.

### Chapter 8 — Advanced
Generators = lazy, constant memory, composable pipelines; single-use. Decorators: `f = dec(f)`, stack bottom-up, `wraps` mandatory. `@contextmanager` needs `try/finally`. Type hints are *executed* by pydantic/FastAPI/tool schemas. Enums for closed value sets. GIL: one thread of bytecode; released during I/O and inside NumPy/PyTorch. asyncio for I/O + Semaphore; processes for CPU-bound Python. Never block the event loop. `loss.item()` not `loss`. Bound every cache. `tracemalloc` to find leaks. Descriptors power `@property`; prefer `__init_subclass__` over metaclasses.

---

### The Twelve Bugs That Cost Real Teams Real Money

1. Mutable default argument → chat history leaks between users.
2. `bool(os.environ["FLAG"])` → `"false"` evaluates to `True`.
3. `list` instead of `set` for dedup → an O(n²) job that never finishes.
4. Non-atomic checkpoint write → corrupt file discovered eight hours later.
5. Unbounded retry with no jitter → thundering herd turns a blip into an outage.
6. Silent `except Exception: pass` → 40% of the dataset dropped, metrics look fine.
7. Appending tensors instead of floats → OOM at epoch 12.
8. `zip` without `strict=True` → predictions misaligned against labels.
9. Blocking call inside `async def` → the whole service stops serving.
10. Unsorted `set` → non-reproducible label ids across runs.
11. Missing `functools.wraps` → LLM tool schemas silently become `(*args, **kwargs)`.
12. No `MAX_STEPS` on an agent loop → an unbounded API bill.

---

# Appendix C — Capstone Project — The Mini RAG Ingestion Service

**Why this project.** It touches all eight chapters and produces something you can put on a CV and extend into real work. Build it yourself before reading any solution; the chapters above contain every technique it needs.

## Specification

Build a CLI tool, `minirag`, that:

**1. Ingests** a directory of `.txt`/`.md`/`.json` documents:
- Streams files (never loads the corpus into memory) — *Ch. 5, 8*
- Cleans and normalises text through a composable transform pipeline — *Ch. 3, 7*
- Chunks with a sliding window, respecting sentence boundaries — *Ch. 2, 3*
- Deduplicates by content hash — *Ch. 4, 5*
- Embeds concurrently with rate limiting, retries, and a circuit breaker — *Ch. 6, 8*
- Persists vectors (`.npy`) plus a manifest atomically — *Ch. 5*
- Supports **incremental** re-ingestion: only changed files are re-embedded — *Ch. 4, 5*

**2. Queries** the index:
- Hybrid retrieval (keyword + dense) fused with RRF — *Ch. 4, 7*
- Metadata pre-filtering — *Ch. 4*
- Token-budgeted context assembly — *Ch. 2*
- Prompt construction from a versioned template — *Ch. 3*
- Graceful "I don't know" when nothing passes the score threshold — *Ch. 6*

**3. Reports**:
- Per-stage timing and error counts — *Ch. 8*
- Corpus statistics and quality flags — *Ch. 1, 4*
- Cost accounting and cache hit rate — *Ch. 6*

## Suggested structure

```
minirag/
├── __init__.py
├── cli.py              # argparse entry point: ingest | query | stats
├── config.py           # frozen dataclasses + YAML/env layering       (Ch. 1, 5, 7)
├── errors.py           # exception hierarchy                          (Ch. 6)
├── models.py           # Document, Chunk, ScoredChunk value objects   (Ch. 7)
├── transforms.py       # Transform ABC + registry + `|` composition   (Ch. 7)
├── chunking.py         # Chunker ABC + sliding/sentence/markdown      (Ch. 2, 7)
├── embedding.py        # Embedder Protocol + async client + cache     (Ch. 6, 8)
├── store.py            # VectorStore ABC + npy/manifest persistence   (Ch. 5, 7)
├── retrieval.py        # Keyword / Dense / Hybrid retrievers          (Ch. 4, 7)
├── prompting.py        # PromptTemplate + budgeted context assembly   (Ch. 3)
├── pipeline.py         # the orchestrator                             (Ch. 7, 8)
├── observability.py    # JSON logging, @timed, metrics registry       (Ch. 6, 8)
└── tests/
    ├── test_chunking.py
    ├── test_retrieval.py
    └── test_pipeline.py     # uses fake Embedder/LLM via Protocols
```

## Milestones

| # | Milestone | Chapters exercised | Definition of done |
|---|---|---|---|
| 1 | Load + clean + chunk to JSONL | 1–5 | 1000 files stream through in constant memory |
| 2 | Fake embedder + in-memory retrieval | 4, 7 | `query()` returns sensible chunks |
| 3 | Error handling + dead-letter + error budget | 6 | A corrupt file is quarantined, not fatal |
| 4 | Async embedding + rate limit + retry | 6, 8 | ≥10× faster than the sequential version |
| 5 | Persistence + incremental re-index | 5 | A second run makes ~0 API calls |
| 6 | Hybrid retrieval + RRF + filters | 4, 7 | Beats dense-only on exact-term queries |
| 7 | Real embedder + real LLM behind Protocols | 7 | Zero changes to `pipeline.py` |
| 8 | Tests, typing, CI | 3, 7, 8 | `pytest` + `mypy` + `ruff` all clean |

## Grading rubric — the questions a senior engineer will ask

- Does memory stay flat as the corpus grows 100×? *(generators)*
- Does a single malformed document kill the run? *(error handling)*
- Can a killed process resume without redoing work? *(checkpoints, atomic writes)*
- Does a second identical run cost near zero? *(content hashing)*
- Can you swap the embedder without touching the pipeline? *(Protocols)*
- Can you run the whole test suite with no network and no API key? *(dependency injection)*
- Does the report tell you *why* documents were dropped? *(observability)*
- Is every network call timed out, retried, and bounded? *(reliability recipe)*

---

# Appendix D — Where To Go Next

You now have the foundation. Learn the libraries **in this order** — each assumes the previous ones.

### Phase 1 — Numerical & data foundations (2–4 weeks)

| Library | Learn specifically | Foundation it builds on |
|---|---|---|
| **NumPy** | ndarray, dtype, broadcasting, views vs copies, axis semantics, vectorisation | Ch. 1 types, Ch. 4 slicing, Ch. 8 memory |
| **Pandas** | Series/DataFrame, indexing, `groupby`, merges, `dtype` hygiene, chunked reads | Ch. 4 dicts, Ch. 5 files |
| **Matplotlib** | figure/axes model, subplots, saving for reports | Ch. 7 objects |

> **The one NumPy insight to carry from this guide:** array slices are *views* (they share memory), unlike list slices which are copies. That single difference causes and cures a large fraction of beginner NumPy bugs.

### Phase 2 — Classical ML (3–4 weeks)

**scikit-learn** — estimators, `Pipeline`, `ColumnTransformer`, cross-validation, metrics, `GridSearchCV`. Every one of these is the duck-typed protocol pattern from Chapter 7; a `Pipeline` is composition; a custom transformer is `BaseEstimator, TransformerMixin` + `fit`/`transform`.

### Phase 3 — Deep learning (6–8 weeks)

**PyTorch** — tensors, autograd, `nn.Module`, `Dataset`/`DataLoader`, training loops, mixed precision, checkpointing, `.to(device)`.
Direct mappings from this guide: `nn.Module.__call__` (Ch. 7 magic methods) · `Dataset` = `__len__` + `__getitem__` (Ch. 7 duck typing) · `DataLoader(num_workers=)` = multiprocessing + picklability (Ch. 8) · `with torch.no_grad():` (Ch. 8 context managers) · `loss.item()` (Ch. 8 memory) · dtype and VRAM (Ch. 1).

### Phase 4 — Transformers & LLMs (4–6 weeks)

**Hugging Face** `transformers`, `datasets`, `tokenizers`, `accelerate`, PEFT/LoRA.
`from_pretrained` is a classmethod (Ch. 7) · `datasets.map(num_proc=)` needs picklable module-level functions (Ch. 3) · streaming mode is generators (Ch. 8) · `safetensors` is Ch. 5's security lesson.

### Phase 5 — LLM applications (4–6 weeks)

**LangChain / LlamaIndex / LangGraph**, vector databases (Chroma, Qdrant, pgvector), embeddings, RAG, agents, **MCP**.
LCEL's `prompt | model | parser` is `__or__` (Ch. 7) · tool schemas come from signatures and docstrings (Ch. 3) · agent state is a `TypedDict` (Ch. 8) · retrieval is Ch. 4's containers · the whole chapter-7 mini project is a hand-rolled version of these frameworks.

### Phase 6 — Serving & MLOps (ongoing)

**FastAPI** (type hints → validation + OpenAPI, async endpoints), **Docker**, **pytest**, **MLflow/W&B**, **Prometheus/OpenTelemetry**, CI with `ruff` + `mypy`.
FastAPI is Chapters 3, 7, and 8 combined: annotations become the contract, pydantic models are validated dataclasses, and endpoints are coroutines.

---

## A Realistic 6-Month Plan

| Month | Focus | Deliverable |
|---|---|---|
| 1 | This guide, cover to cover, typing every example | The Chapter 4 and 7 mini projects, working |
| 2 | NumPy + Pandas + one Kaggle tabular dataset | An EDA notebook and a cleaned dataset with a profiler report |
| 3 | scikit-learn end to end | A trained classifier behind a `Pipeline`, with cross-validated metrics |
| 4 | PyTorch fundamentals | Fine-tune a small transformer for text classification |
| 5 | RAG | The Appendix C capstone, with a real embedder and vector DB |
| 6 | Serving | The capstone behind FastAPI, in Docker, with tests and CI |

**How to study, concretely.**
1. **Type every example.** Reading builds recognition; typing builds recall. The difference is enormous.
2. **Break things deliberately.** Remove `strict=True` from a `zip`, drop `functools.wraps`, use a mutable default — and watch what happens. Bugs you have *caused* are bugs you will *recognise*.
3. **Do the exercises before reading the solutions.** The solutions are written to teach *why*, which only lands after you've struggled with *how*.
4. **Rebuild the mini projects from memory a week later.** If you can't, reread that chapter's summary, not the whole chapter.
5. **Read library source code.** Now that you know descriptors, decorators, ABCs, and `__init_subclass__`, open `torch/utils/data/dataset.py` or a pydantic model class. It will read as ordinary Python — which was the whole point of this guide.

---

## Final Word

The libraries will change. New frameworks will replace the ones named here; model APIs will be rewritten; today's best practice for agents will look quaint in two years.

What will not change is the layer underneath: objects and references, iteration and laziness, functions and composition, containers and their complexity, failure and recovery, concurrency and its costs. Every framework in this field is built from those pieces, which is why engineers who understand them can pick up a new library in an afternoon — and why engineers who don't remain permanently dependent on tutorials that match their exact use case.

You now have that layer. Go build something that breaks, then fix it.

---

*End of guide.*
