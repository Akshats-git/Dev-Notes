# 09 — Why We Need Vector Databases: KNN and the Curse of Dimensionality

**Video:** [FDE Full Course #9 — Why We Need Vector Databases? KNN & Vector Search](https://www.youtube.com/watch?v=5mXvdCDv2eM) (1h 05m)
**Instructor notes:** [Lecture 09 PDF](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2009)

The movie recommender kept 30 vectors in a Java `List` and compared the query against all of them. That's fine for 30. This lecture asks what happens at Netflix, Spotify or Google scale, and shows why the usual database tricks stop working for vectors.

---

## Contents

1. [It worked because the dataset was tiny](#it-worked-because-the-dataset-was-tiny)
2. [Two separate problems: storing and searching](#two-separate-problems-storing-and-searching)
3. [Can't MySQL just store vectors?](#cant-mysql-just-store-vectors)
4. [Exact match vs similarity](#exact-match-vs-similarity)
5. [Build the simplest vector DB yourself](#build-the-simplest-vector-db-yourself)
6. [KNN — K-Nearest Neighbours](#knn--k-nearest-neighbours)
7. [Why brute force doesn't scale: compute and memory](#why-brute-force-doesnt-scale-compute-and-memory)
8. [Why normal database indexes work](#why-normal-database-indexes-work)
9. [Indexing in more than one dimension: k-d trees](#indexing-in-more-than-one-dimension-k-d-trees)
10. [The curse of dimensionality](#the-curse-of-dimensionality)

---

## It worked because the dataset was tiny

[1:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=60s)

```java
class Movie {
    String title;
    String description;
    float[] embedding;     // e.g. Interstellar → [0.21, -0.47, 0.83, ...]
}
List<Movie> movies;        // all in application memory
```

Query: "movie about space exploration and survival" → same embedding model → `[0.19, -0.44, 0.81, ...]` → compare with **every** movie:

```
Interstellar → 0.94
The Martian  → 0.91
Gravity      → 0.87
Inception    → 0.72
Titanic      → 0.31
```

Sort and return the top few. For 30 movies, completely fine.

---

## Two separate problems: storing and searching

[6:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=360s)

Real scale:

- Spotify: millions of songs
- E-commerce: hundreds of millions of products
- The web: billions of pages (the video checks: roughly 3–4 billion public web pages)

`List<float[]> vectors` in RAM stops being an option. Two related but **different** problems appear:

1. **How do we STORE millions or billions of vectors?**
2. **How do we SEARCH them efficiently?**

---

## Can't MySQL just store vectors?

[9:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=540s)

```
movies
id | title        | description        | embedding
1  | Interstellar | Space exploration  | [...]
2  | Inception    | Dream manipulation | [...]
3  | Titanic      | Romantic tragedy   | [...]
```

Nothing prevents a normal database from holding an array of numbers: as JSON, binary, serialized data, or a dedicated vector type (many SQL/NoSQL databases now have vector extensions). So **"traditional databases can't store vectors" is wrong.** They can.

The hard part is: **how do you efficiently *search* millions of vectors by similarity?**

---

## Exact match vs similarity

[12:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=720s)

Traditional queries ask **"which rows satisfy my condition?"**:

```sql
SELECT * FROM movies WHERE id = 10;           -- exact
SELECT * FROM movies WHERE genre = 'Sci-Fi';  -- exact
SELECT * FROM movies WHERE year >= 2010;      -- ordered comparison
```

Operators: `=`, `>`, `<`, `BETWEEN`, `LIKE`, `JOIN`, `GROUP BY`. The answer for each row is **match / no match**.

Vector search asks something else. Query `Q = [0.2, 0.8, 0.3]`. You almost never want a vector *exactly equal* to Q. Two vectors are only identical if the inputs were identical, and even a near-perfect semantic match scores something like 0.94, not 1. You want:

```
A = [0.21, 0.79, 0.31]   ← close
B = [0.19, 0.82, 0.30]   ← close
C = [-0.7, 0.1, 0.5]     ← far

"Give me the 10 vectors closest to Q."
```

That's **nearest-neighbour search**.

| | Traditional | Vector |
|---|---|---|
| Query | `id = 42` | `[0.21, -0.42, 0.75, ...]` |
| Question | Is this value equal to 42? | How close is this vector to the query? |
| Answer | Yes / No | A degree of similarity (more → less similar) |

Netflix example: a user likes Interstellar. You don't want Interstellar back. You want movies **similar to** it.

---

## Build the simplest vector DB yourself

[19:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=1140s)

Forget Pinecone, Qdrant, Weaviate, Milvus and pgvector for a moment. What would a minimal one need?

### Step 1 — Store the vectors (plus ID and metadata)

```
ID: M1
Vector: [0.21, 0.72, 0.13, ...]
Metadata: title = Interstellar, genre = Sci-Fi, year = 2014
```

Three pieces:

- **ID** — when a vector matches, you need to know what it represents.
- **Vector**
- **Metadata / reference to the original data** — you can't reverse an embedding back into the description. Embedding is one-way, so you must store the original info separately. Metadata also enables **filtered** search:

```
Find movies similar to Interstellar
BUT only where genre = Sci-Fi AND year > 2010
```

Similarity search and traditional filters work together.

### Step 2 — Embed the query with the **same** model

"mind-bending movie about space" → embedding model → `Q = [0.23, 0.70, 0.16, ...]`.

| Component | Job |
|---|---|
| **Embedding model** | meaning → numbers |
| **Vector database** | numbers → efficient retrieval |

The vector DB doesn't know Interstellar is about space. The embedding model created that meaning. The DB just searches it fast.

### Step 3 — Compare against every vector, sort, take top K

```
similarity(Q, M1), similarity(Q, M2), similarity(Q, M3), ...

0.95 Interstellar
0.91 The Martian
0.88 Gravity        ← K = 3 stops here
0.29 Titanic
0.21 Notebook
```

That's a basic **K-Nearest Neighbour (KNN) search**.

---

## KNN — K-Nearest Neighbours

```
        A ●
    B ●     ● C
          X  ← query
      ● D
            ● E
```

**K = how many nearest neighbours you want.** K = 1 → the closest point. K = 3 → the three closest. K = 10 → ten closest.

### Exact nearest-neighbour search

Brute force checks **100% of vectors**, so it always returns the **true** nearest K.

```
Query → search every vector → compute all distances → return the true top K
```

**Excellent accuracy, poor scalability.** Which leads to the real question:

> If checking every vector is accurate but too expensive, can we skip vectors that clearly can't be good results?

That's **indexing**.

---

## Why brute force doesn't scale: compute and memory

[26:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=1560s)

### Compute

100 vectors → 100 comparisons per query. Trivial. 100,000,000 vectors → 100 million similarity calculations, for **one** query, while thousands of users search at the same time.

And one comparison isn't one operation. Cosine similarity touches every dimension (`A[i] × B[i]`, summed up). So:

```
work ≈ N × D        →  O(N × D)
N = number of stored vectors
D = number of dimensions
```

Real numbers: N = 10,000,000, D = 1,536:

```
10,000,000 × 1,536 = 15,360,000,000
```

About **15.36 billion** dimension-level operations for one naive query. Brute-force work grows linearly with the dataset.

### Memory

One float32 = 4 bytes. One 1,536-dim vector:

```
1,536 × 4 = 6,144 bytes ≈ 6 KB
```

| Vectors | Raw size |
|---|---|
| 1 million | ≈ 6 GB |
| 10 million | ≈ 60 GB |
| 100 million | ≈ 600 GB |

And that excludes IDs, metadata, indexes, DB overhead, replication and temp data. So vector search is a **compute problem + storage/memory problem**.

### "Just put them on disk"?

That fixes capacity, but then each naive query reads huge amounts of vector data from disk/SSD and computes millions of similarities. Disk is far slower than RAM. Moving to disk doesn't solve *search*.

### Even picking the top K costs something

After computing scores (`M1 → 0.71, M2 → 0.23, M3 → 0.94, …`), you need the highest K. You don't have to fully sort everything. A **min-heap** of size K keeps the current best K as you go (DSA heap knowledge pays off here). It's still extra work.

**Three brute-force costs:** (1) read/fetch vectors, (2) compute similarities, (3) select the best K.

---

## Why normal database indexes work

[37:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=2220s)

Databases solved search speed long ago with **indexes**. Why not just index the embedding column? First, understand why ordinary indexes work.

`SELECT * FROM employee WHERE age = 32;` on 10 million rows. Without an index, scan: `23, 41, 19, 32 ✓, 56, 32 ✓ …`. With an index, the values are **ordered**:

```
18 20 23 27 32 36 41 50
```

Looking for 32 and currently at 27? Everything smaller is irrelevant, so go right. This is binary-search thinking (B-trees/B+ trees in real DBs). Finding 9 in 1..10 linearly takes 9 comparisons. Binary search (5 → 7 → 8 → 9) takes 4.

> Indexes are powerful because they let you **eliminate large parts of the search space** without inspecting them.

### Nearest neighbour in 1-D is easy

Give each movie one number (Interstellar 12, Superman 35, Titanic 42, Batman 67, …). Sorted: `12, 35, 42, 67, 80, 86, 99`. A query at 40 → binary search lands between 35 and 42. The nearest neighbours **must** be right around that position. Just look left and right.

```
            30
          /    \
        10      50
       /  \    /  \
      0   20  40   60

Nearest to 42: 42 > 30 → right; 42 < 50 → left → 40.
Large parts of the tree are discarded.
```

It works because 1-D numbers have a **natural order**.

---

## Indexing in more than one dimension: k-d trees

[45:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=2700s)

### 2-D has no single order

```
A = [2, 8]
B = [5, 3]
```

Which is "smaller"? By x, A < B. By y, A > B. "Smaller" and "bigger" lose their meaning once you leave one dimension.

### Split space one dimension at a time

Split on x first (`x < 5` | `x ≥ 5`), then inside each half split on y, then x again, alternating:

```
+----------------+----------------+
|       E        |       B        |
|                |                |
+----------------+-------F--------+
|       C        |                |
|       A        |       D        |
+----------------+----------------+
```

That's a **k-d tree** (k-dimensional tree), basically binary search extended to many dimensions.

### Build one

Points: `A=[2,3] B=[8,7] C=[4,5] D=[7,2] E=[1,8] F=[6,6]`.

1. Sort by x: `E[1,8] A[2,3] C[4,5] F[6,6] D[7,2] B[8,7]`. Pick the median, **F[6,6]**, as the root (split on x).
2. Left (x < 6): E, A, C. Right (x > 6): D, B.
3. Next level splits on **y**.

```
                F [6,6]  (x split)
              /          \
        C [4,5]          B [8,7]
       (y split)        (y split)
       /      \          /
      A        E        D
```

### Search it

Query `Q = [7, 6]`.

1. At root F (split on x): `7 > 6` → go right first. On the way, F itself is a candidate: `dist(Q, F) = 1`.
2. Continue down the right side: `dist(Q, B[8,7]) ≈ 1.41`, `dist(Q, D[7,2]) = 4`. Best so far: F at 1.
3. **Can we skip the left side entirely?**

### When pruning works

Split at `x = 5`, query at `x = 8`, best distance so far = 1. Anything on the left is at least 3 units away in x alone, so it can't beat 1. **Skip the entire left region.** That's **pruning**: search right, skip left. Exactly what an index should do.

### When pruning fails

Query at `x = 5.1`, only 0.1 from the boundary, best distance so far = 2. A point just across the line could easily be closer. You **must** search both sides. The pruning opportunity is gone.

In 2-D this is still manageable. You might be near the x boundary but far from the y boundaries, so you can still prune other regions. k-d trees work fine in **low** dimensions.

---

## The curse of dimensionality

[1:02:00](https://www.youtube.com/watch?v=5mXvdCDv2eM&t=3720s)

In 3-D there are boundaries for x, y **and** z, so more chances for the query to sit near *some* boundary. Every time you can't prove the other region is irrelevant, you have to search it. More branches survive.

With 1,000+ dimensions there's a boundary in dimension 1, 2, 3, … 1000. It becomes very hard for the index to ever say *"this whole region definitely cannot contain a closer vector"*. The tree you hoped would go `ROOT → B → E → done` turns into

```
          ROOT
        ↙      ↘
       A        B
     ↙  ↘     ↙  ↘
    D    E   F    …
```

and visits 70%, 80%, 90% of the data. You're back to brute force.

> The **curse of dimensionality** doesn't mean adding one dimension suddenly breaks search. It means every extra dimension adds more ways for points to differ and more regions that *might* hold a neighbour, so pruning keeps getting weaker.

---

## The core problem

```
Huge number of vectors
  → large storage requirements
  → millions of expensive similarity calculations
  → lots of memory/disk access
  → slow search
```

Traditional indexes win by eliminating big chunks of an **ordered** space. Embeddings live in **high-dimensional** space with no natural order, and geometric structures like k-d trees lose their pruning power as dimensions grow.

> **The central challenge of large-scale vector search:** how do we search only a *small fraction* of vectors while still finding results *very close* to the true nearest neighbours?

The answer, approximate nearest-neighbour search (ANN, HNSW, IVF, product quantization), is the next lecture.

## Key takeaways

1. Traditional DBs **can** store vectors. Storage alone isn't the main problem.
2. Vector search asks for **similarity**, not exact equality.
3. A vector record needs an ID, the vector, and metadata/original data, since embeddings can't be reversed.
4. The embedding model creates meaning. The vector DB only retrieves efficiently.
5. Returning the K closest vectors is **K-Nearest Neighbour** search.
6. Brute-force exact search is perfectly accurate but costs `O(N × D)` per query.
7. 1,536-dim float32 ≈ 6 KB per vector. 1M ≈ 6 GB, 100M ≈ 600 GB raw.
8. Moving vectors to disk saves memory but makes naive search slower.
9. Ordinary indexes work because 1-D values are ordered and let you prune.
10. Vectors have no single ordering. k-d trees split one dimension at a time and prune well in low dimensions.
11. In high dimensions, pruning collapses (**curse of dimensionality**) and trees approach brute force.

## Interview questions

**Q: Why can't we just put a B-tree index on an embedding column?**
B-trees rely on a total ordering of values. High-dimensional vectors have no meaningful single order, and similarity search needs "closest", not "equal" or "in range".

**Q: What's KNN, and what's its cost by brute force?**
Return the K stored vectors closest to the query. Brute force compares against all N vectors across D dimensions: `O(N·D)` per query.

**Q: How much memory do 10M OpenAI `text-embedding-3-small` vectors need?**
1,536 × 4 bytes ≈ 6 KB each → about 60 GB raw, before IDs, metadata and indexes.

**Q: What is the curse of dimensionality in vector search?**
As dimensions grow, a query is almost always near some partition boundary, so tree indexes can't safely prune and end up visiting most of the data.

**Q: How do you get top-K without sorting all N scores?**
Keep a min-heap of size K. Push each score and pop the smallest when size exceeds K: `O(N log K)`.
