# 10 — How Vector Databases Work: ANN, IVF, HNSW and Product Quantization

**Video:** [FDE Full Course #10 — How Vector Databases Actually Work](https://www.youtube.com/watch?v=YitjkFRN20Q) (1h 34m)
**Instructor notes:** [Lecture 10 PDF](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2010)

Lecture 9 ended on the problem: exact search over millions of high-dimensional vectors is too slow and too big, and tree indexes collapse in high dimensions. This lecture covers how real vector databases get around it: **don't look at most vectors** (ANN via IVF or HNSW), and **store vectors smaller** (quantization, PQ).

---

## Contents

1. [Recap: exact KNN is correct but expensive](#recap-exact-knn-is-correct-but-expensive)
2. [ANN: approximate nearest neighbour](#ann-approximate-nearest-neighbour)
3. [Measuring ANN quality: Recall@K](#measuring-ann-quality-recallk)
4. [IVF — divide the space into regions](#ivf--divide-the-space-into-regions)
5. [HNSW — navigate a layered graph](#hnsw--navigate-a-layered-graph)
6. [IVF vs HNSW](#ivf-vs-hnsw)
7. [Compression: quantization and product quantization](#compression-quantization-and-product-quantization)
8. [What makes a vector database](#what-makes-a-vector-database)
9. [Products: Pinecone, Qdrant, Weaviate, pgvector](#products-pinecone-qdrant-weaviate-pgvector)

---

## Recap: exact KNN is correct but expensive

[0:00](https://www.youtube.com/watch?v=YitjkFRN20Q&t=0s)

Key property: **similar meaning → similar vectors → nearby locations in vector space.** Interstellar, The Martian and Gravity cluster together. Titanic and Avengers are elsewhere.

10,000,000 movie embeddings. Query "movies about humans exploring deep space" → query vector → **find the top 10 nearest**. Brute force:

```
for every vector V:
    similarity = cosine(query, V)
return highest 10
```

That's **exact nearest-neighbour search**. Its big advantage: **it doesn't guess.** With a fixed dataset and distance function, it finds the mathematically nearest items. Nothing is wrong with exact KNN except the cost: `10,000,000 × 1,536 ≈ 15 billion` operations per query, with thousands of users querying at once.

---

## ANN: approximate nearest neighbour

[11:00](https://www.youtube.com/watch?v=YitjkFRN20Q&t=660s)

**The coffee-shop analogy:** Rohit is in Delhi and wants the nearest coffee shop. You don't compute his distance to every coffee shop in India. Delhi is promising. Mumbai, Bangalore, Chennai and Assam are obviously irrelevant. Huge areas get eliminated *before* any detailed comparison.

> Exact search asks: how do I compute the distance to **every** vector faster?
> ANN asks: how do I **avoid considering most vectors** in the first place?

### Exact vs approximate

| Exact search | ANN |
|---|---|
| 1. Interstellar | 1. Interstellar |
| 2. The Martian | 2. The Martian |
| 3. Gravity | 3. **Apollo 13** |

ANN missed Gravity and returned Apollo 13. For movie recommendations, does the user care? Not at all. Apollo 13 is still a great space movie. It would only be a real failure if it returned Titanic for a space query.

| | Exact | ANN |
|---|---|---|
| Accuracy | Perfect | Very high |
| Compute | High | Much lower |

### What exactly is "approximate"?

ANN usually doesn't compute distances *incorrectly*. The approximation happens **earlier**, in candidate selection. Instead of 10,000,000 candidates (or 100M), the index picks maybe 10,000 promising ones and computes **proper** distances only for those.

```
  ● A
       ● B   ← just outside the searched region
  Q
  ● C
```

If the index decides Q belongs to A's region and B sits just outside it, ANN can miss B even if B is slightly closer, because **`distance(Q, B)` was never calculated**. Not because it was calculated wrong. That's where "approximate" comes from.

---

## Measuring ANN quality: Recall@K

[21:00](https://www.youtube.com/watch?v=YitjkFRN20Q&t=1260s)

Exact top 10: `A B C D E F G H I J`. ANN top 10: `A B C D E F G H X Y`.

```
Recall@K = (true nearest neighbours recovered) / K
Recall@10 = 8 / 10 = 0.8 = 80%
```

ANN isn't simply "inaccurate". It's a **dial**:

```
Search almost everything     → very high recall → more work   (≈ exact)
Search a tiny region         → lower recall     → very fast
Explore more candidates      → higher recall    → a bit more work
```

The video's benchmark idea: if 20,000 comparisons out of 100 million get you 95% recall, that's an excellent ANN. Most ANN algorithms expose a knob for *how much of the space to explore*.

> **The central ANN trade-off: Speed ↔ Recall.**

Two big families pick candidates differently:

- **IVF** → divide the vector space into regions
- **HNSW** → navigate a graph of vectors

---

## IVF — divide the space into regions

[25:00](https://www.youtube.com/watch?v=YitjkFRN20Q&t=1500s)

### Intuition

Embeddings naturally form neighbourhoods: space movies here, romance there, superhero over there. Instead of one giant list, carve the space into regions. Real vectors have 1,536 or 3,072 dimensions, but the geometry is the same idea.

1,000,000 vectors → 1,000 clusters → ~1,000 vectors each (if balanced). Query Q arrives: **which cluster looks closest?** Search only that one.

```
1,000,000 comparisons   →   find promising cluster + ~1,000 comparisons
```

### Centroids

How do you represent a cluster? With its **centroid**, the average/central point, *"the rough address of a neighbourhood"*. A centroid is itself a vector with coordinates.

```
1,000,000 vectors → 1,000 regions → 1,000 centroids
```

At query time you compare the query against the **centroids** first, not all million vectors.

### Index build (done ahead of time, not per query)

```
INDEX BUILD
1,000,000 vectors → group into regions (clustering, e.g. k-means)
  → compute centroids → assign every vector to a region → store the groups
```

Reused for every search.

### Mini example (2-D)

| Movie | Vector |
|---|---|
| Interstellar | [8,8] |
| Gravity | [8,7] |
| The Martian | [7,8] |
| Titanic | [2,2] |
| The Notebook | [2,3] |
| La La Land | [3,2] |
| Avengers | [8,2] |
| Iron Man | [7,2] |
| Thor | [8,3] |

```
List 1 (C1 — romance):   Titanic, The Notebook, La La Land
List 2 (C2 — space):     Interstellar, Gravity, The Martian
List 3 (C3 — superhero): Avengers, Iron Man, Thor
```

Query "astronauts travelling through space" → `Q = [7.7, 7.5]`.

1. `distance(Q, C1)` far, `distance(Q, C2)` very close, `distance(Q, C3)` moderately far.
2. Search **List 2 only** → Interstellar 0.94, The Martian 0.91, Gravity 0.86.

The romance and superhero movies were never evaluated. That's the speed-up.

### Why "IVF"?

**Inverted File Index.** Like a book's index (word → pages), it maps each centroid to the list of vectors that belong to it:

```
Centroid A → [V3, V9, V81, V400, ...]
Centroid B → [V2, V7, V33, V101, ...]
Centroid C → [V1, V4, V18, V900, ...]
```

```
Query → find promising centroid → fetch its vector list → search that list
```

> **IVF: divide the world, then search the most promising neighbourhoods.**

### The boundary problem

```
   Cluster A   |   Cluster B
     ●         |
     ●         |
     ●         |  ● ← actual nearest
       Q       |
```

Q is just on A's side, so only A gets searched. The true nearest neighbour sits just across the line in B and is missed. **Vector space has no real walls.** Clusters are an optimisation the index imposes.

### Fix: search several clusters (probes)

Search the 5 or 20 closest clusters instead of 1:

```
more clusters searched → more candidates → lower chance of missing → higher recall → more work
```

In **IVFFlat** (e.g. pgvector) this knob is **probes**. Few probes = fast, possibly lower recall. More probes = more work, higher recall.

At scale: 1M vectors, 1,000 lists, ~1,000 per list. Exact search ≈ 1,000,000 vectors inspected. IVF with 5 probes ≈ 5 × 1,000 = **~5,000** candidates (+ 1,000 centroid checks).

---

## HNSW — navigate a layered graph

[39:00](https://www.youtube.com/watch?v=YitjkFRN20Q&t=2340s)

**Hierarchical Navigable Small World.** Sounds scary, but the idea is simple. IVF divides space. HNSW asks:

> What if nearby vectors were **connected** to each other? Then search becomes **navigation**.

### Start with a graph

```
  A ─── B
  │  /  │
  C ─── D ─── E
         \
          F   × Q
```

Each vector is a **node**. Links between nearby vectors are **edges**. A knows B and C. B knows A, C and D. D knows B, C, E and F.

### Search by walking

Q is near F. Start somewhere, say A:

1. `distance(Q, A)`. Check A's neighbours, B is closer → move to B.
2. B's neighbours: D is closer → move to D.
3. D's neighbours (B, E, F): F is closest → move to F.
4. No neighbour of F is closer → done.

```
Start somewhere → inspect neighbours → move toward the closest → repeat
```

### Problem: flat graphs need many hops

`A-B-C-D-E-…-Z` with only local links. Start near A, query near Z → `A → B → C → D → …`, a lot of hops. The more complicated the graph, the more hops.

### The road analogy → hierarchy

A long trip isn't all local streets:

```
local street → main road → HIGHWAY → main road → local street
```

Highways give **big jumps**. Local roads give **precision**, like finding one specific house on a street. HNSW builds the same thing as layers:

```
Layer 3:  A ──────────────── M ──────────────── Z           (very few nodes, huge jumps)
Layer 2:  A ─────── F ─────── M ─────── S ─────── Z
Layer 1:  A ── C ── F ── H ── K ── M ── P ── S ── V ── Z
Layer 0:  A-B-C-D-E-F-G-H-I-J-K-L-M-N-O-P-Q-R-S-T-U-V-W-X-Y-Z   (every vector)
```

Higher layers have fewer nodes and longer jumps. Lower layers have more nodes and fine-grained moves.

### Query walk-through (query near X)

1. **Layer 3:** start at entry node A. M is much closer → `A → M`. (Z isn't closer than M.)
2. **Drop to layer 2:** from M, S is closer → `M → S`.
3. **Drop to layer 1:** from S, V is closer → `S → V`.
4. **Layer 0:** local search `V → W → X`. Found it.

```
Start high → big jumps → reach roughly the right area
→ drop a layer → smaller jumps → drop again → fine local search
```

In the video's example, about 9 comparisons instead of 26 for A–Z. At millions of nodes, the saving is enormous.

### Details worth knowing

- **What a node stores:** its vector plus references to nearby nodes, possibly different per layer. `Interstellar → {The Martian, Gravity, Ad Astra, Arrival}`. HNSW stores the **graph**, not just vectors.
- **HNSW doesn't understand meaning.** The embedding model creates the semantic geometry. HNSW only sees `vector A`, `vector B`, `distance(A, B)` and navigates efficiently.
- **Why not connect everything to everything?** With 1M vectors that's about 1M edges per node, the scalability problem again. HNSW builds a **sparse** graph with strategically chosen neighbours.
- **Layer membership is probabilistic.** Every vector is in layer 0. Progressively fewer, randomly chosen ones get promoted to higher layers. That creates the hierarchy.
- **Not purely greedy.** Always taking the single best-looking edge can trap you in a local region (a slightly closer node now, but a dead end). Real HNSW keeps **several promising candidates** during exploration to avoid that.
- **Build cost.** Inserting vector X = find good neighbours → choose connections → insert. More effort gives a better graph but slower builds. Trade-offs exist at **build time and query time**.
- **`M` parameter** ≈ max connections per node. 2 neighbours: less memory but harder navigation. 100: easier navigation but more edges, memory, build work, and neighbours to inspect.
- **Memory hungry.** HNSW stores vectors **plus** graph edges **plus** multiple layers. With 10M vectors those edges add up. Part of HNSW's speed is paid for in RAM.

> **HNSW: navigate the vector space instead of scanning it.**

---

## IVF vs HNSW

```
IVF                                        HNSW
├─ divide space into regions               ├─ vectors are graph nodes
├─ represent regions by centroids          ├─ connect useful nearby nodes
├─ find promising regions (probes)         ├─ build hierarchical layers
└─ search vectors inside them              └─ navigate broad → local
```

| | IVF | HNSW |
|---|---|---|
| Core idea | Partition + search a few partitions | Layered proximity graph |
| Main knob | number of lists, **probes** | **M** (connections), search breadth |
| Memory | Lower (mostly grouping info) | Higher (graph edges, layers) |
| Weak spot | Neighbours across cluster boundaries | Build time, RAM |

Both avoid checking every vector. They just do it differently.

---

## Compression: quantization and product quantization

[1:09:00](https://www.youtube.com/watch?v=YitjkFRN20Q&t=4140s)

ANN fixes **search cost**. **Storage cost** is a separate problem. 100M vectors × 1,536 dims × 4 bytes (float32) = 6,144 bytes each ≈ **614 GB** raw, before metadata, IDs, indexes, HNSW edges, DB overhead and replication.

### Do we need full precision?

`[0.183742, -0.729182, 0.029117, ...]`. Each value is stored very precisely. For nearest-neighbour search, maybe you don't need that.

**Quantization:** represent high-precision values with lower-precision ones.

| Format | Bytes per dim | One 1,536-dim vector | vs float32 |
|---|---|---|---|
| float32 | 4 | 6,144 B | 1× |
| float16 | 2 | 3,072 B | 2× smaller |
| 1 byte (int8) | 1 | 1,536 B | 4× smaller |

### Product Quantization (PQ) goes much further

Tiny example: `V = [0.2, 0.8, -0.4, 0.1, 0.7, -0.2, 0.9, 0.3]`. 8 dims × 4 B = 32 bytes.

**1. Split into sub-vectors:**

```
[0.2, 0.8] | [-0.4, 0.1] | [0.7, -0.2] | [0.9, 0.3]
  part 1       part 2        part 3        part 4
```

8 dims → 4 sub-vectors of 2 dims.

**2. Learn common patterns per sub-space.** Look at just the *first* sub-vector of all 1M vectors: `[0.20,0.80]`, `[0.22,0.79]`, `[0.18,0.84]`, `[0.21,0.77]`… Lots of near-duplicates. Cluster them:

```
Cluster 0: [0.20,0.80] [0.22,0.79] [0.18,0.84]   → centroid C0 = [0.20, 0.81]
Cluster 1: [-0.42,0.12] [-0.39,0.08] [-0.45,0.14] → centroid C1 = [-0.42, 0.11]
```

Each representative is a **centroid** or **codeword**. All of them for a sub-space form a **codebook**.

**3. Store IDs instead of floats.** Instead of `[0.22, 0.79]`, store `0` (= "use codeword 0 from this codebook"). The whole vector:

```
[0.2, 0.8 | -0.4, 0.1 | 0.7, -0.2 | 0.9, 0.3]
   → #17      → #203     → #81      → #6
stored as: [17, 203, 81, 6]
```

**Why 256 centroids?** 256 = 2⁸ → one ID fits in **8 bits = 1 byte**. So each sub-vector costs 1 byte.

### PQ on a real 1,536-dim embedding

```
1,536 dims → 96 sub-vectors × 16 dims each
each sub-vector → 1 of 256 centroids → 1 byte
encoded vector = 96 × 1 byte = 96 bytes   (vs 6,144 bytes)
compression    = 6,144 / 96 = 64×
```

```
Original  6,144 B  ████████████████████████████████████████████████████████████
PQ           96 B  █
```

### PQ is lossy

Original sub-vector `[0.217, 0.794]` → stored as centroid `[0.200, 0.810]`. Close, not identical. The reconstruction is an approximation.

> Less storage ↔ less numerical precision. Another trade-off.

> **PQ: instead of storing every coordinate exactly, store which representative pattern each chunk most closely resembles.**

---

## What makes a vector database

[1:29:00](https://www.youtube.com/watch?v=YitjkFRN20Q&t=5340s)

It's more than a pile of arrays. Five layers:

1. **Original data** — `ID: 101, Title: Interstellar, Description: …, Genre: Sci-Fi, Year: 2014`.
2. **Vector representation** — `embedding: [0.21, -0.45, 0.81, ...]` stored with the record.
3. **Similarity/distance function** — cosine, dot product, or Euclidean (L2).
4. **Vector index** — an ANN index (HNSW, IVF/IVFFlat) for efficient candidate search instead of comparing against everything.
5. **Metadata** — for filtering.

### Metadata filtering

Real apps almost never search on similarity alone. *"Recent Hindi comedy movies similar to this one"*:

```
Find top 10 vectors nearest to Q
WHERE language = 'Hindi' AND genre = 'Comedy' AND year >= 2020
```

Also called **scalar filtering** or **payload filtering**. A record is really `ID + VECTOR + METADATA`, not just `[0.21, 0.54, ...]`.

Semantic results: Interstellar (Sci-Fi, 2014), The Martian (Sci-Fi, 2015), Gravity (Sci-Fi, 2013), Apollo 13 (Drama, 1995). Filter `genre = Sci-Fi AND year >= 2014` → Interstellar, The Martian.

How filtering interacts with ANN (filter before? after? during?) differs between systems, and combining the two efficiently is a real engineering problem of its own.

---

## Products: Pinecone, Qdrant, Weaviate, pgvector

**Dedicated vector databases:** Pinecone, Qdrant, Weaviate (also Milvus). Built around storing, indexing and searching vectors.

**Add vectors to a database you already have:** PostgreSQL + **pgvector**:

```sql
CREATE TABLE movies (
    id        BIGSERIAL PRIMARY KEY,
    title     TEXT,
    embedding VECTOR(1536)
);

SELECT *
FROM movies
ORDER BY embedding <=> :queryVector     -- <=> is cosine distance
LIMIT 10;
```

You can create **HNSW** or **IVFFlat** indexes on the `embedding` column. Vector search doesn't always mean adopting a whole new database product.

---

## The complete mental model

```
Original data → embedding model → vector → vector storage
  → distance/similarity function → ANN index → candidate retrieval
  → metadata filtering → top K results

At scale, optionally: original vector → quantization / PQ → smaller representation
```

| Problem | Solved by |
|---|---|
| **Search cost** | ANN: IVF, HNSW |
| **Storage cost** | Quantization, Product Quantization |
| **Real application constraints** | IDs, metadata, filtering, application data |

> Vector databases don't magically understand meaning. **The embedding model creates the vector geometry. The vector database stores it and searches it efficiently.**

## Key takeaways

1. Exact KNN is mathematically correct but scales with N × D per query.
2. ANN avoids looking at most vectors. The "approximation" is in **which candidates get checked**, not in the distance maths.
3. Recall@K = fraction of the true top-K that ANN returned. The core trade-off is speed ↔ recall.
4. **IVF:** cluster vectors (centroids), map centroid → list (inverted file), search the nearest few lists (**probes**). Boundary misses are fixed with more probes.
5. **HNSW:** sparse proximity graph in layers. Start at the sparse top layer with big jumps, descend to the dense bottom layer for precise local search. Tune with **M**. Memory-hungry.
6. Quantization (float32 → float16/int8) cuts storage 2–4×. **PQ** splits vectors into sub-vectors, replaces each with a 1-byte codebook ID, and can get ~64× smaller. It's lossy.
7. A vector DB = data + vectors + distance function + ANN index + metadata filtering.
8. Options: dedicated (Pinecone, Qdrant, Weaviate, Milvus) or bolt-on (pgvector in PostgreSQL).

## Interview questions

**Q: Why is it called *approximate* nearest neighbour?**
The index only examines a subset of candidates. If a true neighbour falls outside that subset, it's never scored and can be missed. Distances that are computed are exact.

**Q: How do you measure ANN quality?**
Recall@K: run exact search for ground truth, then measure what fraction of the true top-K the ANN index returned. Trade it off against latency.

**Q: Explain IVF in one breath.**
Cluster vectors with k-means, store an inverted list per centroid, compare the query with the centroids, then exhaustively search only the vectors in the `nprobe` nearest lists.

**Q: Explain HNSW in one breath.**
A multi-layer proximity graph. Sparse top layers allow long jumps, the dense bottom layer has every vector. Search greedily descends from the top, keeping a candidate list, refining at each layer.

**Q: IVF or HNSW?**
HNSW usually gives better recall/latency but uses more memory and builds slower. IVF is lighter, simpler to shard and compress (IVF-PQ), but needs tuning of lists/probes and can miss boundary neighbours.

**Q: How does product quantization get 64× compression on 1,536-dim vectors?**
Split into 96 sub-vectors of 16 dims, learn 256 centroids per sub-space, and store each sub-vector as a 1-byte centroid ID: 96 bytes vs 6,144.

**Q: Do I need a dedicated vector DB?**
Not necessarily. pgvector adds a `VECTOR` type, distance operators and HNSW/IVFFlat indexes to Postgres. Dedicated DBs make sense at very large scale or when you need their specific features.
