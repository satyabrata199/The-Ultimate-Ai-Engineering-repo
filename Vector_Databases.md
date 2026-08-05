# Vector Databases

> Finding the nearest vectors among a hundred million, in ten milliseconds, without exact search. This phase is about the index parameters that decide whether your retrieval is fast, accurate, or cheap — you get to pick two.

**After this phase you can:** choose a vector store for a given scale and budget, tune HNSW without guessing, size memory before you deploy, and avoid the filtering mistake that silently destroys recall.

---

## 1. The Problem

```
 EXACT nearest-neighbour search over 10M vectors of 1536 dimensions:
   10,000,000 × 1536 multiply-adds per query ≈ 15 GFLOPs
   → hundreds of milliseconds per query, single-threaded. Unusable.

 ANN (Approximate Nearest Neighbour) trades a little RECALL for
 orders of magnitude of SPEED.

   recall@10 = 0.95  →  you find 95% of the true top-10
                        and it takes 5 ms instead of 500 ms

 THE ONLY QUESTION THAT MATTERS:
   how much recall are you willing to lose, for how much speed and memory?
```

**And the answer is usually: more than you think.** For RAG, the difference between recall 0.95 and 0.99 is rarely visible in answer quality — a reranker over 50 candidates absorbs it. The difference between 5 ms and 500 ms is very visible.

---

## 2. Similarity Metrics

| Metric | Formula | Range | Use for |
|---|---|---|---|
| **Cosine** | `A·B / (‖A‖‖B‖)` | −1 to 1 | **text embeddings — the default** |
| **Dot product** | `A·B` | unbounded | normalised vectors (identical ranking to cosine, faster) |
| **Euclidean (L2)** | `√Σ(Aᵢ−Bᵢ)²` | 0 to ∞ | image embeddings, coordinates |

**The rule that saves confusion:**

```
 If your vectors are UNIT-NORMALISED (‖v‖ = 1) — and most text
 embedding models return normalised vectors — then:

   cosine similarity == dot product        (identical RANKING)
   L2 distance is a monotonic function of cosine  (identical RANKING)

 → All three give the SAME ORDER. Pick dot product; it's the fastest.

 ⚠ If vectors are NOT normalised, they differ substantially:
   dot product favours LONG vectors regardless of direction.
   Normalise at ingest and stop thinking about it.
```

```python
import numpy as np
def normalise(v: np.ndarray) -> np.ndarray:
    return v / np.linalg.norm(v)
# Then use dot product / "IP" as your metric everywhere.
```

**Score interpretation:** cosine scores are model-relative. 0.82 means nothing on its own — calibrate any threshold against your own labelled data.

---

## 3. Index Types

### Flat (brute force)

```
 Compare against every vector. Exact — recall = 1.0 by definition.
 O(n) per query.

 ✔ Under ~100k vectors: fast enough (few ms) and perfectly accurate
 ✔ The baseline you measure other indexes against
 ✘ Linear scaling: 10M vectors ≈ 500 ms+
```

**Start here.** Many teams build an HNSW index for 20,000 vectors, which is pure complexity for no gain.

### HNSW — Hierarchical Navigable Small World

**The default for most production systems.** A multi-layer graph: sparse long-range links on top for coarse navigation, dense short-range links at the bottom for precision.

```
  Layer 2   ●─────────────────●              few nodes, long hops
             \               /
  Layer 1    ●───●───────●──●                more nodes
              \   \     /   /
  Layer 0    ●─●─●─●─●─●─●─●─●               ALL vectors, short hops

  Search: enter at the top, greedily walk toward the query,
          descend a layer, repeat. O(log n).
```

| Parameter | Meaning | Effect |
|---|---|---|
| **M** | edges per node (layer 0 gets 2M) | ↑ recall, ↑ **memory**, ↑ build time |
| **ef_construction** | candidate list size while building | ↑ index quality, ↑ build time only |
| **ef_search** | candidate list size at query time | ↑ recall, ↑ **latency** — tunable live |

```
 STARTING VALUES
   M = 16                 general purpose
   M = 32–48              when recall matters more than memory
   ef_construction = 200  (higher costs build time only — be generous)
   ef_search = 64–128     TUNE THIS ONE against your recall target

 ⚠ ef_search must be ≥ top_k, and realistically ≥ 2–4× top_k.
   Requesting top_k=50 with ef_search=64 gives poor recall.

 MEMORY
   vectors:  n × dims × 4 bytes (fp32)
   graph:    n × M × 2 × ~4 bytes (bidirectional links)

   10M × 1536-d, M=16:
     vectors ≈ 61 GB          ← dominates
     graph   ≈ 1.3 GB
     → HNSW is MEMORY-HUNGRY. This is its main cost.
```

```
 ✔ Best recall/latency trade-off available
 ✔ Supports incremental insert
 ✘ Memory-hungry (everything resident)
 ✘ Deletes are usually tombstones — needs periodic compaction
 ✘ Slow to build at scale
```

### IVF — Inverted File Index

Cluster the vectors (k-means) into `nlist` cells; at query time search only the `nprobe` nearest cells.

```
   ●●●  ●●●     Query lands near centroid C2.
  ● C1● ●C2●    With nprobe=2, search C2 and C1 only —
   ●●●  ●●●     skip the other 98 cells entirely.
  ●●●   ●●●
 ● C3●  ●C4●
```

| Parameter | Guidance |
|---|---|
| **nlist** | ≈ `4 × √n` (1M vectors → ~4,000 cells) |
| **nprobe** | 1–5% of nlist. The recall/speed dial. |

```
 ✔ Much lower memory than HNSW
 ✔ Fast to build
 ✘ REQUIRES TRAINING on a representative sample before inserting
 ✘ Recall degrades if the data distribution shifts after training
 ✘ The boundary problem: a true neighbour just across a cell border is missed
```

Use IVF when memory is the binding constraint and you can tolerate retraining.

### PQ — Product Quantization

Compression, not a search structure — usually combined with IVF (`IVF_PQ`).

```
 Split a 1536-d vector into m=96 subvectors of 16 dims each.
 Learn a 256-entry codebook per subvector.
 Store each subvector as ONE BYTE (its codebook index).

   Before: 1536 × 4 bytes = 6,144 bytes
   After:  96 × 1 byte    =    96 bytes        ← 64× smaller

 10M vectors: 61 GB → 0.96 GB. Fits in RAM instead of needing a fleet.
```

```
 ✔ Massive memory reduction — often the difference between viable and not
 ✘ Lossy: recall drops (typically 0.95 → 0.80–0.90 depending on m)
 ✘ Needs training

 ✔ THE STANDARD FIX: rerank with exact distances.
   Retrieve top-100 with PQ (approximate), rescore those 100 against the
   full-precision vectors, return top-10. Recovers most of the lost recall
   for the cost of 100 exact comparisons.
```

**Alternatives worth knowing:** scalar quantization (fp32 → int8, 4× smaller, ~1% recall loss — a much gentler trade than PQ) and binary quantization (32× smaller, large recall loss, usable only with reranking). **Scalar quantization is the best default compression** — near-free memory savings.

### DiskANN / Vamana

Graph index designed to live on SSD with a small in-memory cache.

```
 ✔ Billion-scale on one machine at a fraction of the RAM cost
 ✘ Higher latency (10–50 ms — SSD reads)
 → Use when the corpus is enormous and latency budget is generous.
```

### Choosing

```
 < 100k vectors            FLAT. Exact, simple, fast enough.
 100k – 10M                HNSW. The default.
 10M – 100M, RAM-limited   HNSW + scalar quantization, or IVF_PQ + rerank
 > 100M                    DiskANN, or shard HNSW across nodes
 Frequent full rebuilds    IVF (fast to build)
 Heavy incremental writes  HNSW (incremental insert)
```

---

## 4. The Databases

| | **Type** | **Best for** | **Watch out** |
|---|---|---|---|
| **pgvector** | Postgres extension | **you already run Postgres** | slower at >10M; index build is heavy |
| **Qdrant** | dedicated, Rust | best all-round default | another service to operate |
| **Chroma** | embedded / server | prototypes, notebooks | not for production scale |
| **FAISS** | a *library*, not a DB | research, custom pipelines | no persistence, no filtering, no server |
| **Milvus** | distributed | billion-scale | heavy: etcd + MinIO + several components |
| **Weaviate** | dedicated + modules | built-in hybrid search, GraphQL | opinionated schema model |
| **Pinecone** | fully managed | zero ops, fast to ship | cost at scale, vendor lock-in |
| **LanceDB** | embedded, columnar | local/edge, versioned data | younger ecosystem |

### pgvector — start here if you can

```sql
CREATE EXTENSION vector;

CREATE TABLE chunks (
    id          uuid PRIMARY KEY,
    org_id      uuid NOT NULL,
    kb_id       uuid NOT NULL,
    document_id uuid NOT NULL,
    content     text NOT NULL,
    embedding   vector(1536) NOT NULL,
    metadata    jsonb NOT NULL DEFAULT '{}'
);

-- HNSW index. vector_ip_ops = inner product (use with normalised vectors).
CREATE INDEX ON chunks USING hnsw (embedding vector_ip_ops)
    WITH (m = 16, ef_construction = 200);

-- Filter columns need their own indexes — the ANN index doesn't cover them.
CREATE INDEX ON chunks (org_id, kb_id);

SET hnsw.ef_search = 100;                      -- per-session recall dial

SELECT id, content, 1 - (embedding <=> :q) AS score
FROM chunks
WHERE org_id = :org AND kb_id = :kb           -- ← pgvector 0.8+ filters well
ORDER BY embedding <=> :q
LIMIT 20;
```

**Why pgvector is usually the right first choice:**
```
 ✔ ONE database — transactions across your chunks and your business data
 ✔ No sync problem between "the DB" and "the vector DB"
 ✔ Row-level security works, so tenant isolation is enforced at the database
 ✔ Hybrid search in one query (vector + tsvector BM25)
 ✔ Your existing backups, monitoring, and migrations already cover it

 ✘ Slower than dedicated stores above ~10M vectors
 ✘ HNSW index builds are slow and memory-intensive
 ✘ Vector search competes with OLTP traffic for the same resources
   → use a read replica for search once volume is real
```

### Qdrant — the best dedicated default

```python
from qdrant_client import AsyncQdrantClient, models

await client.create_collection(
    collection_name=f"org_{org_id.hex}",
    vectors_config=models.VectorParams(size=1536, distance=models.Distance.COSINE),
    hnsw_config=models.HnswConfigDiff(m=16, ef_construct=200),
    quantization_config=models.ScalarQuantization(          # 4× memory saving
        scalar=models.ScalarQuantizationConfig(type=models.ScalarType.INT8,
                                               always_ram=True)),
    optimizers_config=models.OptimizersConfigDiff(indexing_threshold=20_000),
)

# ⚠ Payload indexes are REQUIRED for filters to be fast.
await client.create_payload_index(collection_name=coll, field_name="kb_id",
                                  field_schema=models.PayloadSchemaType.KEYWORD)

hits = await client.query_points(
    collection_name=f"org_{org_id.hex}",
    query=embedding,
    query_filter=models.Filter(must=[
        models.FieldCondition(key="kb_id", match=models.MatchValue(value=str(kb_id))),
        models.FieldCondition(key="acl_groups", match=models.MatchAny(any=groups)),
    ]),
    limit=20,
    search_params=models.SearchParams(hnsw_ef=128),
)
```

**Multi-tenancy:** a collection per tenant is the strongest isolation but costs memory per collection (bad past a few thousand tenants). Qdrant's **shard keys** give hard partitioning inside one collection — the right choice at high tenant counts.

---

## 5. Filtering — The Part That Silently Breaks Recall

```
 THREE STRATEGIES, AND ONLY ONE IS RIGHT

 ❌ POST-FILTERING
    search top-10 → discard non-matching results
    Problem 1: you asked for 10 and get 2. Recall collapses.
    Problem 2: one forgotten filter is a cross-tenant data leak.

 ⚠ PRE-FILTERING (brute force over the matching subset)
    filter first → exact search the survivors
    Correct, and fast ONLY when the filter is very selective.

 ✔ FILTERED ANN (what modern stores do)
    the filter is evaluated DURING graph traversal — non-matching nodes
    are skipped as the search walks. Correct AND fast.
    → Verify your store does this. Qdrant, Weaviate, and pgvector 0.8+ do.
```

**The selectivity problem, which surprises people:**

```
 A filter matching 0.1% of vectors can make HNSW SLOWER, not faster.
 The graph was built over all vectors; traversal keeps landing on
 excluded nodes and has to keep walking.

 Good stores detect this and switch strategy below a cardinality threshold
 (Qdrant: "payload index cardinality" → falls back to filtered brute force).

 ⚠ MEASURE RECALL WITH YOUR FILTERS APPLIED, not just latency.
   A filter that quietly drops recall from 0.95 to 0.60 raises no error
   and makes every answer worse.
```

**Multi-tenancy done right:**
```
 < 1,000 tenants        collection per tenant — strongest, simplest
 > 1,000 tenants        one collection + shard key / partition on org_id
 Always                 org_id in the payload AND in the filter AND a
                        redundant assertion on results (Phase 15 §12)
```

---

## 6. Sizing

```python
def size_index(n_vectors: int, dims: int, *, m: int = 16,
               quantization: str = "none") -> dict:
    bytes_per_dim = {"none": 4, "scalar": 1, "binary": 0.125}[quantization]
    vectors = n_vectors * dims * bytes_per_dim
    graph   = n_vectors * m * 2 * 4              # HNSW links
    payload = n_vectors * 300                    # metadata, rough
    total   = vectors + graph + payload
    return {"vectors_gb": vectors / 1e9, "graph_gb": graph / 1e9,
            "total_gb": total / 1e9,
            "recommended_ram_gb": total * 1.5 / 1e9}   # + working memory

# 10M chunks, 1536-d, HNSW m=16:
#   fp32:      61 GB vectors +  1.3 GB graph → ~94 GB RAM recommended
#   int8:      15 GB vectors +  1.3 GB graph → ~25 GB RAM  ← usually the answer
```

**Chunk-count estimation:**
```
 documents × pages/doc × chunks/page
 10,000 docs × 20 pages × 3 chunks = 600,000 chunks
 → comfortably HNSW in RAM on a single modest node.

 Most teams overestimate. Do this arithmetic before choosing a distributed store.
```

**Cost sanity check:** managed vector databases often cost more per month than the LLM calls they serve. Compute it before committing — self-hosted Qdrant or pgvector on a machine you already run is frequently 10× cheaper.

---

## 7. Operations

```python
# ── Batch upserts. One vector per request is 100× slower. ──
BATCH = 500
for i in range(0, len(points), BATCH):
    await client.upsert(collection, points=points[i:i + BATCH], wait=False)

# ── Deterministic IDs → upsert is idempotent (re-ingestion is safe) ──
point_id = str(uuid5(NAMESPACE, f"{document_id}:{chunk_index}"))
```

```
 DELETES
   Most graph indexes tombstone rather than remove. Deleted vectors still
   occupy memory and are still traversed until compaction.
   → Schedule optimisation/compaction. Monitor the deleted-vector ratio.
   → For "delete a whole tenant", dropping a collection beats deleting
     millions of points.

 RE-EMBEDDING (the migration you will eventually do)
   Changing the embedding model invalidates every vector.
   ✔ Build a NEW collection alongside the old one
   ✔ Backfill in batches; both models live simultaneously
   ✔ Switch reads behind a flag; keep the old collection until confident
   ✘ Never re-embed in place — you'd have a half-migrated index serving
     nonsense comparisons between two vector spaces

 BACKUPS
   Snapshot the index AND keep the source text. An index is rebuildable
   from source; source lost is lost forever.

 MONITORING
   search latency p50/p95/p99   ·   recall against a fixed golden query set
   memory / RSS                 ·   deleted-vector ratio
   index build lag              ·   filter cardinality distribution
```

**Measure recall continuously, not once:**

```python
async def measure_recall(golden: list[Query], k: int = 10) -> float:
    """Compare ANN results against exact search on a fixed sample.
    Run nightly — recall degrades silently as data grows and shifts."""
    hits = 0
    for q in golden:
        approx = {h.id for h in await store.search(q.vector, top_k=k)}
        exact  = {h.id for h in await store.search_exact(q.vector, top_k=k)}
        hits += len(approx & exact) / k
    return hits / len(golden)
```

---

## 8. Tuning Playbook

```
 SYMPTOM                      →  FIX
 ─────────────────────────────────────────────────────────────────────────
 Recall too low               →  raise ef_search (query-time, free to try)
                              →  then raise M (requires rebuild + memory)
                              →  check ef_search ≥ 2–4× top_k
 Latency too high             →  lower ef_search
                              →  add quantization (int8 first)
                              →  add a metadata filter to shrink the space
 Memory too high              →  scalar quantization (4×, ~1% recall loss)
                              →  lower M
                              →  PQ + exact rerank (64×, bigger loss)
 Filtered queries slow        →  add payload/column indexes on filter fields
                              →  check the store does FILTERED ANN, not post-filter
 Recall dropped after growth  →  IVF needs retraining, or ef_search no longer
                                 sufficient at the new scale
 Slow index build             →  lower ef_construction (build-time only)
                              →  build offline, then swap the collection
```

**Tune in this order: `ef_search` (free, live) → quantization (cheap, big memory win) → `M` (expensive rebuild).**

---

## Common Mistakes

```
❌ Reaching for a distributed vector DB at 50,000 vectors — flat search is fine
❌ Running a separate vector DB when pgvector would remove a whole sync problem
❌ Not normalising vectors, then wondering why dot product ranks strangely
❌ ef_search < top_k → poor recall with no error
❌ No payload/column indexes on filter fields → filtered queries crawl
❌ Post-filtering instead of filtered ANN → recall collapse and leak risk
❌ Measuring only latency, never recall — the failure is silent
❌ One shared collection with no tenant filter → cross-tenant retrieval
❌ Never compacting after deletes → memory grows, latency grows
❌ Re-embedding in place instead of building a parallel collection
❌ Random point IDs → re-ingestion duplicates instead of upserting
❌ Backing up the index but not the source text
❌ Choosing a managed vector DB without pricing it against the LLM spend
```

---

## Checklist

```
[ ] Vector count estimated before choosing a store
[ ] Flat index used below ~100k vectors
[ ] Vectors unit-normalised; dot product / IP as the metric
[ ] HNSW: M and ef_construction set; ef_search tuned to a recall target
[ ] ef_search ≥ 2–4× top_k
[ ] Scalar quantization considered before adding hardware
[ ] Memory sized (vectors + graph + payload) × 1.5
[ ] Payload/column indexes on every filter field
[ ] Filtering verified as in-search, not post-search
[ ] Per-tenant collections or shard keys; org_id filter + redundant assertion
[ ] Deterministic point IDs so upserts are idempotent
[ ] Batched upserts (500+ per request)
[ ] Compaction scheduled; deleted-vector ratio monitored
[ ] Recall measured nightly against a golden query set and exact search
[ ] Re-embedding runbook: parallel collection, backfill, flagged switchover
[ ] Source text backed up independently of the index
```

---

**Next:** Phase 17 — agents, where retrieval becomes one tool among many and the control flow stops being a straight line.
