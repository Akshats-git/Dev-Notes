# 12 — Advanced RAG: Chunking, Filtering, Hybrid Search, Reranking

**Video:** [FDE Full Course #12 — Advanced RAG Explained](https://www.youtube.com/watch?v=_FygCwbpreU) (1h 12m)
**Instructor notes:** [Lecture 12 PDF](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2012)

Basic RAG (lecture 11) works for simple questions. It quietly assumes **the retrieved chunks contain everything needed to answer**. This lecture is about what to do when they don't: improve each stage of the pipeline instead of throwing the whole thing out.

---

## Contents

1. [Why basic RAG isn't enough](#why-basic-rag-isnt-enough)
2. [Retrieval failure vs generation failure](#retrieval-failure-vs-generation-failure)
3. [Better chunking](#better-chunking)
4. [Better retrieval: Top-K, threshold, metadata filters](#better-retrieval-top-k-threshold-metadata-filters)
5. [Hybrid search: semantic + keyword](#hybrid-search-semantic--keyword)
6. [Reranking](#reranking)
7. [Context engineering (intro)](#context-engineering-intro)
8. [Quick reference](#quick-reference)

---

## Why basic RAG isn't enough

[0:00](https://www.youtube.com/watch?v=_FygCwbpreU&t=0s)

```
Question → query embedding → vector similarity search → top-K chunks
         → question + context → LLM → answer
```

Fictional company policies:

| Document | Policy |
|---|---|
| General Return Policy | Eligible products may be returned within **30 days** of delivery. |
| Electronics Return Policy | Electronic products may be returned within **7 days**, subject to inspection. |
| Damaged Product Policy | Damaged products must be reported within **48 hours** of delivery. |

Customer: **"Can I return my damaged headphones after 10 days?"**

The right answer needs three facts: headphones are electronics (7 days), the product is damaged (48-hour reporting), and 10 days have passed. So the answer is **no**.

If the search only retrieves the general policy, the LLM sees "30 days" and says *"Yes, you can return it."* That's a perfectly reasonable answer **given what it saw**. The LLM isn't incapable. It got the wrong context.

> A good LLM can't reliably use a policy that never reached its context.

Similarity search, embeddings and LLMs are all probabilistic. The right documents won't always come back.

---

## Retrieval failure vs generation failure

[9:00](https://www.youtube.com/watch?v=_FygCwbpreU&t=540s)

| Failure | What went wrong | Example | Where to look |
|---|---|---|---|
| **Retrieval failure** | The needed info wasn't retrieved | Found the general policy, missed the electronics one | Chunking, search strategy, retrieval params, filters |
| **Generation failure** | Info was retrieved but used wrongly or incompletely | Got both policies, still followed the general 30-day rule | Context arrangement, instructions, generation |

They need **different fixes**. A stronger system prompt can't recover a document that was never retrieved. Retrieving 10 or 20 more documents doesn't guarantee the model interprets them correctly.

### "Advanced RAG" = improve individual stages

| Problem | Improvement |
|---|---|
| Important info gets split apart or loses context | **Better chunking** |
| Irrelevant documents show up | **Retrieval tuning and filtering** |
| Exact identifiers/keywords are missed | **Hybrid search** |
| Useful documents are present but ranked too low | **Reranking** |
| Context is repetitive or contains competing info | **Context engineering** |

### Practical first step: look at what was retrieved

Before judging the LLM's answer, print what the vector DB returned:

```java
List<Document> relevantChunks = vectorStore.similaritySearch(
        SearchRequest.builder().query(question).topK(4).build());

for (int i = 0; i < relevantChunks.size(); i++) {
    System.out.println("DOCUMENT " + (i + 1));
    System.out.println(relevantChunks.get(i).getText());
    System.out.println("--------------------");
}
```

Ask questions you know the answers to. That gives you a **baseline** to compare every later improvement against.

> **Evaluate retrieval and generation separately.** First check what the DB returned, then check how the model used it.

---

## Better chunking

[12:00](https://www.youtube.com/watch?v=_FygCwbpreU&t=720s)

**Why chunk at all?** A 100-page (or 5,000-page) document covering returns, refunds, shipping, warranties and exchanges as **one** vector forces 1,536 numbers to represent many unrelated topics. A movie description is 3–4 lines, and that fits fine in one vector. A whole handbook doesn't. For "What's the return period for electronics?" it's too broad to match precisely. The real design question is **where to split without losing meaning**.

### 1. Fixed-size chunking

Split every N characters or tokens. Simple and predictable, but the split point ignores meaning:

```
Electronics Return Policy
Electronic products can be returned within 7 days of delivery.
Products must be in their original packaging. Items showing signs
of physical damage caused by the customer are not eligible for
return. Customers must provide the original purchase invoice.
```

→

```
Chunk 1: Electronic products can be returned within 7 days of delivery.
         Products must be in their original packaging.
Chunk 2: Items showing signs of physical damage caused by the customer
         are not eligible for return. Customers must provide the invoice.
```

A question about a physically damaged electronic product needs **both**: the window (chunk 1) and the exception (chunk 2). Retrieve only one and the answer is incomplete.

**Overlap** repeats some text between neighbours (e.g. take 200 tokens, then step back 20 and take the next 200) so boundary-spanning conditions survive. But too much overlap means more embeddings, more storage and repetitive results. It's a trade-off. A starting point to experiment with: **400–600 tokens, 10–20% overlap**, compared against no overlap. These aren't universal numbers.

### 2. Structural / document-aware chunking

Most documents already have meaningful boundaries: title → heading → subheading → paragraph → table. **Split on those** instead of blindly. Since you're not cutting mid-thought, overlap is usually not needed.

**Keep the heading in the chunk:**

```
Return Policy → Electronics
Electronic products can be returned within 7 days.
```

Without the heading, "within 7 days" could be about returns, cancellations, warranty registration or refunds.

For a small set of policy sections, add each self-contained section as its own `Document` with metadata:

```java
List<Document> documents = List.of(
    new Document("""
            Return Policy → Electronics
            Electronic products can be returned within 7 days of delivery.
            """,
        Map.of("source", "return-policy", "category", "electronics", "policyType", "returns")),
    new Document("""
            Return Policy → Clothing
            Clothing products can be returned within 30 days of delivery.
            """,
        Map.of("source", "return-policy", "category", "clothing", "policyType", "returns"))
);
vectorStore.add(documents);
```

For long sections: split by headings **first**, then run a token splitter on anything still too big. Meaningful boundaries first, size limits second.

### 3. Semantic chunking

For long, **unstructured** text with no headings, where one paragraph drifts from returns to shipping:

```
Customers can return eligible products within 30 days.
Electronic products must be returned within 7 days.
Returned products must be inspected before approval.
We offer standard shipping across India.
Orders are generally dispatched within 24 hours.
Express delivery is available in selected cities.
```

```
Split into sentences / small groups
  → embed neighbouring groups
  → compare similarity of neighbours
  → find sharp drops (topic transitions)
  → cut there → topic-coherent chunks
```

| Adjacent pair | Similarity (illustrative) |
|---|---|
| return 1 ↔ return 2 | 0.92 |
| return 2 ↔ return 3 | 0.87 |
| **return 3 ↔ shipping 1** | **0.31** ← boundary |
| shipping 1 ↔ shipping 2 | 0.94 |

→ chunk 1 = the three return sentences, chunk 2 = the three shipping sentences. Easy to implement, but it costs extra embedding calls and indexing time, and it doesn't guarantee tables, cross-references or exceptions stay intact.

### The chunk-size trade-off

| Smaller chunks | Larger chunks |
|---|---|
| More specific searchable units | More surrounding context per result |
| Risk losing surrounding conditions | Risk mixing unrelated topics (vector gets blurry) |
| May need several chunks per answer | More prompt tokens per retrieved chunk |

No universally right size. Test against your documents' structure and the questions people actually ask.

### Spring AI: token-based splitter

```java
TokenTextSplitter splitter = TokenTextSplitter.builder()
        .withChunkSize(400)            // target ~400 tokens per chunk
        .withMinChunkSizeChars(150)    // min chars considered when looking for a split point
        .withKeepSeparator(true)       // keep newlines etc.
        .build();

List<Document> chunks = splitter.apply(originalDocuments);
vectorStore.add(chunks);
```

`withMinChunkSizeChars` is **not** an overlap setting and not a hard minimum for every chunk.

**Demo caution:** delete old vectors or write to a **separate namespace** before comparing chunking strategies. Otherwise old and new chunks get mixed in results.

---

## Better retrieval: Top-K, threshold, metadata filters

[33:00](https://www.youtube.com/watch?v=_FygCwbpreU&t=1980s)

Once good chunks are stored, retrieval settings decide which ones reach the LLM.

### Top-K — how many?

```java
SearchRequest.builder().query(question).topK(4).build();
```

| Rank | Document | Similarity (illustrative) |
|---|---|---|
| 1 | Electronics Return Policy | 0.94 |
| 2 | General Return Policy | 0.89 |
| 3 | Damaged Product Policy | 0.85 |
| 4 | Exchange Policy | 0.79 |
| 5 | Warranty Policy | 0.68 |
| 6 | Shipping Policy | 0.42 |
| 7 | Payment Policy | 0.31 |

- **K too small** → may drop the exception you need.
- **K too large** → irrelevant or contradictory content, more tokens, cost and latency, and a confused LLM.

For a small support bot, try K = 3, 5 and 8 and look at the actual chunks. Test on known questions to tune it.

### Similarity threshold — are the closest results relevant enough?

"Who won the FIFA World Cup?" The knowledge base has no football, but nearest-neighbour search **always returns something**, the "closest" policy docs. **Closest ≠ relevant.**

```java
SearchRequest.builder()
        .query(question)
        .topK(4)
        .similarityThreshold(0.75)    // drop anything below
        .build();
```

```java
if (relevantChunks.isEmpty()) {
    return "I couldn't find relevant information in our company documents.";
}
```

This also saves an LLM call for off-topic questions. System prompts aren't 100% reliable, so this is a cheap extra guard.

Caveats:

- **Similarity is not a correctness probability.** 0.90 does not mean a 90% chance the answer is right. Scores depend on the embedding model, vector store, metric and collection. Tune 0.75 against real test queries.
- A chunk above the threshold still might not contain the full answer, so the prompt must still discourage unsupported claims. ("I ordered food 2 hours ago and want to return it" might pull return policies on an e-commerce site that doesn't sell food.)

### Metadata filtering — search the right subset

Each chunk is stored as **embedding + metadata**. The metadata can carry structured attributes:

```java
new Document("Electronic products can be returned within 7 days of delivery.",
        Map.of("category", "electronics", "policyType", "returns", "source", "return-policy"));
```

| Text | category | policyType |
|---|---|---|
| Electronic products can be returned within 7 days. | electronics | returns |
| Clothing products can be returned within 30 days. | clothing | returns |
| Refunds are initiated after inspection. | general | refunds |

**Attach metadata while loading PDFs** (simple filename-based rules for the demo):

```java
for (Resource resource : policyFiles) {
    String fileName = resource.getFilename();
    List<Document> pages = new PagePdfDocumentReader(resource).read();

    String category   = fileName.contains("electronics") ? "electronics" : "general";
    String policyType = fileName.contains("return")      ? "returns"     : "general";

    List<Document> updatedPages = new ArrayList<>();
    for (Document page : pages) {
        updatedPages.add(page.mutate()
                .metadata("source", fileName)
                .metadata("category", category)
                .metadata("policyType", policyType)
                .build());
    }
    allChunks.addAll(splitter.apply(updatedPages));   // chunks inherit page metadata
}
vectorStore.add(allChunks);
```

The text is embedded for semantic search. The metadata is for filtering.

**Filter a search:**

```java
vectorStore.similaritySearch(SearchRequest.builder()
        .query(question)
        .topK(5)
        .filterExpression("category == 'electronics'")
        .build());
```

Spring AI's filter expressions get translated into the vector store's native filter (Pinecone here).

**Don't over-filter.** An electronics question may need both the electronics-specific *and* the general policy:

```java
.filterExpression("category in ['electronics', 'general']")
```

**Where does the category come from?** Many support chatbots ask the user to pick a category (Orders, Electronics, Payments…) before typing. If the UI already gives you the category, use it directly in the filter. You don't need an extra LLM call to infer it. If the user picks the wrong category, they may get "not found", so design the UX for that.

### All three together

```java
vectorStore.similaritySearch(SearchRequest.builder()
        .query(question)
        .topK(5)
        .similarityThreshold(0.70)
        .filterExpression("category in ['electronics', 'general']")
        .build());
```

| Control | Question it answers |
|---|---|
| Top-K | How many matches can come back? |
| Similarity threshold | How relevant must a match be? |
| Metadata filter | Which subset may be searched? |

---

## Hybrid search: semantic + keyword

[53:00](https://www.youtube.com/watch?v=_FygCwbpreU&t=3180s)

**"Does the AX-204 wireless headphone support fast charging?"**

| Doc | Excerpt |
|---|---|
| A — AX-204 manual | Model AX-204 supports standard USB-C charging. **Fast charging is not supported.** |
| B — BX-900 manual | These wireless headphones **support fast charging**, 5 hours playback after 10 minutes. |
| C — General charging guide | Wireless headphones supporting fast charging typically need a compatible adapter. |

Semantic search may rank **B** at the top because it's all about "wireless headphones + fast charging". But the user asked about **AX-204** specifically. Embeddings capture meaning, but they **don't reliably preserve arbitrary identifiers** like product codes, SKUs or error codes. The LLM then gets the wrong product's info and either says "irrelevant info" or hallucinates "yes, it supports fast charging".

| Semantic search | Keyword / lexical search |
|---|---|
| Matches concepts via embeddings | Matches the actual terms |
| Connects "money back" ↔ "refund" | Finds exact IDs like `AX-204` |
| Can rank a related-but-wrong product high | Misses different wording of the same idea |

**BM25** is the classic keyword ranking algorithm. **Hybrid search** runs both and merges them:

```
                 Customer question
          ┌──────────────┴──────────────┐
   Semantic search                 Keyword search (BM25)
   (vector similarity)                    │
     Ranked list A                  Ranked list B
          └──────────────┬──────────────┘
                 Combine ranked lists
                         ↓
                  Final candidates
```

### Reciprocal Rank Fusion (RRF)

You **can't just add the scores**. Cosine similarity and BM25 are on completely different scales. RRF combines **rank positions** instead:

```
RRF(d) = Σᵢ  1 / (k + rankᵢ(d))          (sum over the m result lists)
```

- `m` = number of result lists (2 here: semantic + keyword)
- `rankᵢ(d)` = position of document d in list i (1 = top). If a list didn't return d, it contributes 0.
- `k` = a smoothing constant, commonly **60**. **Not** the Top-K setting.

Example: keyword search ranks doc A **1st**, semantic search ranks A **2nd**:

```
RRF(A) = 1/(60+1) + 1/(60+2) ≈ 0.01639 + 0.01613 = 0.03252
```

Do the same for every doc, sort by RRF, take the top few. A document near the top of **both** lists wins. No incompatible scores are compared.

> Semantic search finds related meaning. Keyword search preserves exact terms. RRF merges their rankings.

---

## Reranking

[1:01:00](https://www.youtube.com/watch?v=_FygCwbpreU&t=3660s)

### The problem

"Can I return my damaged headphones after 10 days?" Initial retrieval:

| Rank | Document |
|---|---|
| 1 | General Return Policy |
| 2 | Electronics Return Policy |
| 3 | Refund Processing Policy |
| **4** | **Damaged Product Reporting Policy** |
| 5 | Exchange Policy |

The crucial damaged-product doc **was** retrieved, but at #4. With top-3 you'd drop it. You could use top-5, but you can't know in advance which position the important one will land at. Retrieval found it. **Ranking** let it down.

### Why another ranking stage?

In embedding search, **query and documents are embedded independently**. Document vectors are precomputed before any question exists. That's what makes it fast over millions of chunks, but a cosine score doesn't deeply evaluate *this* question against *this* policy.

A **reranker** reads the **actual question and each candidate together** and scores their relevance. A common architecture is the **cross-encoder**.

| Initial embedding retrieval (bi-encoder) | Cross-encoder reranking |
|---|---|
| Query and docs represented independently | Question + candidate processed **jointly** |
| Doc embeddings precomputed | Relevance computed for the current question |
| Efficient over huge collections | More work per candidate |
| Produces an initial candidate list | **Reorders** that list |

### Two-stage retrieval

```
Question
  → Stage 1: retrieve 10–20 candidate chunks (fast vector search)
  → Stage 2: rerank them against the question (slower, smarter)
  → keep the top 3–5
  → send to the LLM
```

| Document | Initial rank | Reranker score (illustrative) |
|---|---|---|
| General Return Policy | 1 | 0.68 |
| Electronics Return Policy | 2 | 0.91 |
| Refund Processing Policy | 3 | 0.39 |
| **Damaged Product Reporting** | **4** | **0.97** |
| Exchange Policy | 5 | 0.28 |

Damaged-product jumps from 4th to 1st. Take the top 3 (0.97, 0.91, 0.68) and the LLM has everything it needs.

> Reranking doesn't create missing information. It only reorders candidates already retrieved.

### Practical: Pinecone's hosted reranker (Java)

```java
// 1. Retrieve MORE candidates than you'll finally use
List<Document> candidates = vectorStore.similaritySearch(
        SearchRequest.builder().query(question).topK(10).build());

// 2. Pinecone inference client
Pinecone pc = new Pinecone.Builder(System.getenv("PINECONE_API_KEY")).build();
Inference inference = pc.getInferenceClient();

// 3. Candidates as {id, text}
List<Map<String, Object>> documents = candidates.stream()
        .map(doc -> Map.<String, Object>of("id", doc.getId(), "text", doc.getText()))
        .toList();

// 4. Rerank, keep top 4
RerankResult result = inference.rerank(
        "bge-reranker-v2-m3",   // hosted reranking model
        question,               // the original query
        documents,              // the 10 candidates
        List.of("text"),        // field(s) to rank on
        4,                      // return top 4
        true,                   // include document content in result
        Map.of());
System.out.println(result.getData());
```

Compare the initial vector order with the reranked order when you run it.

**Alternative:** you can ask an LLM to rerank ("score each document's relevance to this question from 0 to 1"). Dedicated cross-encoder models are usually cheaper and faster for this.

**Cost:** reranking is another model call, so more latency and cost, and it compares everything twice. A tiny knowledge base may not benefit. Use it when your domain is complicated enough that ordering matters.

---

## Context engineering (intro)

[1:11:00](https://www.youtube.com/watch?v=_FygCwbpreU&t=4260s)

Retrieval can be perfect and the answer still wrong.

"Can I return my headphones after 10 days?" Retrieved:

| Doc | Info |
|---|---|
| General Return Policy | Products can be returned within 30 days. |
| Electronics Return Policy | Electronic products can be returned within 7 days. |
| Refund Policy | Approved refunds processed in 5–7 business days. |
| Electronics Exchange Policy | Eligible electronics can be exchanged within 7 days. |

If the LLM says *"Yes, the general return window is 30 days"*, it ignored the more specific rule even though both were in context. That's a **generation failure**.

**Context engineering** is about *what* information you give the model and *how you organise it*, so it applies the relevant facts and constraints correctly. It matters most when context is redundant, unrelated or contradictory. (The instructor's script ends at this introduction.)

---

## Quick reference

| Technique | Main question | Typical failure it fixes |
|---|---|---|
| **Better chunking** | Where should documents be split? | A rule and its exception end up in separate, incomplete chunks |
| **Top-K** | How many chunks to retrieve? | Too few miss rules. Too many add noise. |
| **Similarity threshold** | Are even the closest results relevant? | Off-topic question still retrieves policy docs |
| **Metadata filtering** | Which subset is eligible? | Search ignores a known product category/policy type |
| **Hybrid search** | How to combine meaning and exact terms? | A similar product shows up instead of the exact SKU |
| **RRF** | How to merge differently-scored ranked lists? | Vector and BM25 scores can't be compared directly |
| **Reranking** | Which candidates deserve the top positions? | Useful policy retrieved but ranked too low |
| **Context engineering** | How should retrieved material be presented? | Model prefers a general rule over a specific one |

> **Practical principle:** evaluate retrieval and generation separately. First inspect what the DB returned, then how the model used it.

## Key takeaways

1. Basic RAG assumes retrieval brings back everything needed. Often it doesn't.
2. Separate **retrieval failures** (wrong or missing chunks) from **generation failures** (right chunks, wrong use). They need different fixes.
3. Always print and inspect retrieved chunks to set a baseline.
4. Chunking: fixed-size (+ overlap) is simple. Structural keeps headings and boundaries. Semantic finds topic shifts via embedding similarity drops.
5. Tune Top-K empirically. Add a similarity threshold so off-topic questions retrieve nothing. Similarity isn't a probability of correctness.
6. Store metadata with chunks and filter on it (`filterExpression`). Don't over-filter. Use UI-provided categories when you have them.
7. Hybrid search (semantic + BM25) catches exact IDs that embeddings miss. Merge with **RRF** `Σ 1/(k + rank)`.
8. Reranking: retrieve wide (10–20), rerank with a cross-encoder, keep the top 3–5. It reorders, never invents.
9. Context engineering handles cases where the right info was retrieved but misused.

## Interview questions

**Q: Your RAG bot gives a wrong answer. How do you debug it?**
First check retrieval: print the chunks. If the needed info isn't there, it's a retrieval problem (chunking, K, threshold, filters, hybrid search, reranking). If it is there, it's a generation problem (prompt, context ordering, instructions).

**Q: Fixed-size vs structural vs semantic chunking?**
Fixed-size cuts every N tokens (simple, may split rules, use overlap). Structural splits on headings/sections and keeps headings (great for structured docs). Semantic embeds sentences and cuts where neighbour similarity drops (for unstructured text, costs more).

**Q: Why add a similarity threshold if you already have Top-K?**
Top-K always returns K results even for off-topic queries. A threshold drops irrelevant matches, so you can return "not found" without calling the LLM.

**Q: Why hybrid search?**
Embeddings capture meaning but can miss exact tokens like product codes. BM25 catches exact terms. Combine both rankings (usually with RRF).

**Q: What is RRF and why not just add the scores?**
Reciprocal Rank Fusion scores each doc as `Σ 1/(k + rank)` across lists. Cosine and BM25 scores are on different scales, so ranks are a fair common currency.

**Q: Bi-encoder vs cross-encoder?**
Bi-encoder embeds query and docs separately (fast, precomputable, used for first-stage retrieval). Cross-encoder reads query + doc together (slower, more accurate, used to rerank a short candidate list).
