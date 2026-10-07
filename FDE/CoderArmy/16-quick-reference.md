# 16 — Quick Reference

Everything from the course on one page. Each section links back to the full note.

---

## The FDE mindset — [01](01-fde-and-ai-foundations.md)

- A customer's request ("build a chatbot") is usually a **proposed solution**. Find the **requirement** (e.g. "cut resolution time without losing accuracy or trust").
- Flow: request → business problem → existing workflow → constraints → design → integrate → test/evaluate → production.
- The LLM is one component. The system around it (data, APIs, rules, security, validation, humans) is the job.
- **Fluent ≠ correct.** Generating "your order is cancelled" ≠ cancelling the order.

## AI history in one line — [01](01-fde-and-ai-foundations.md)

Rules (don't scale) → ML (learn from examples, needs features) → deep learning (learns representations) → n-grams (sparse, short context) → RNNs (forget long-range) → **attention** → **Transformers** → **GPT** (Generative Pre-trained Transformer).

## How an LLM generates — [02](02-how-llms-work.md)

```
text → tokenizer → tokens → token IDs → embeddings (+ position)
     → N × Transformer blocks (self-attention Q/K/V + more)
     → final vector → logits (one per vocab token) → softmax → probabilities
     → decoding (greedy / sampling / temperature) → token → append → repeat
```

| Term | Meaning |
|---|---|
| Token | Word / sub-word / symbol chunk. ≈ 4 chars of English (rough estimate only) |
| Token ID | Arbitrary lookup number. Carries no meaning |
| Embedding | Learned vector that carries meaning |
| Q / K / V | What I'm looking for / what I contain / what I contribute |
| Self-attention | New token vector = Σ (weight × Value of every token), weights from Q·K |
| Logits | Raw scores over the vocabulary |
| Temperature | `logit / T` before softmax. Low = sharper/deterministic, high = flatter/creative |
| Autoregressive | Output is fed back in as input, one token at a time |

## Calling an LLM — [03](03-calling-an-llm-from-your-app.md)

- Every request = **endpoint + API key + model + input**. Keys go in env vars, never Git.
- Cost/latency/context = **input tokens + output tokens**.
- **The LLM only knows what reaches the LLM.** Live data, maths and actions come from app-provided **tools**.

```java
chatClient.prompt().system(SYS).messages(history).user(msg)
          .tools(t1, t2)          // lecture 5
          .call().content();      // or .stream().content() → Flux<String> (6)
                                  // or .call().entity(MyRecord.class)   (13)
```

## Chat, context, memory — [04](04-chatbot-context-memory-hallucinations.md)

- API calls are **stateless**. The app stores history and re-sends it.
- Roles: **system** (standing rules, higher priority) · **user** · **assistant**.
- System prompt = role + task + behaviour + constraints. **Not retraining, not security.**
- **Context window** = everything sent + everything generated, in tokens.
- History (stored) ≠ context (sent). Strategies: last-N, first+recent, **summary + recent**, retrieve-relevant.
- Relevant context > maximum context.
- Prompt info ≠ training. App memory ≠ model learning.
- **Hallucination** = plausible continuation, not verified fact. Reduce it with RAG, tools, validation and guardrails. It never reaches zero.

## Tools & agents — [05](05-tools-and-ai-agents.md)

- **LLM = brain, tools = hands.** A tool is a normal function + name + description + param schema.
- Model emits `{"tool": "...", "arguments": {...}}`. The **app executes** it, then sends the result back to the model, looping.
- Information tools (observe) vs action tools (change the world).
- **Agent** = goal + decision maker (LLM) + tools + environment + observations + loop.
- Workflow: you define what + how. Agent: you define what, the LLM decides how.
- **Tool = capability + permission.** Narrow tools, sandbox dir, `safePath` against traversal, errors returned as strings, **human-in-the-loop** for risky actions.
- *LLM decides what it wants. The application decides what it's allowed.*

## Streaming — [06](06-streaming.md)

- Streaming changes **delivery**, not generation. It improves **time to first token**, not total time.
- Token ≠ provider chunk ≠ network chunk ≠ `reader.read()`. Never depend on boundaries.
- SSE = one-way server→client (what providers use). WebSockets = two-way (usually overkill).
- Must survive **end to end**: provider → `Flux<String>` → controller → HTTP → `response.body.getReader()` + `TextDecoder({stream:true})` → append to **one** bubble.
- `await response.text()` kills streaming.
- History: `.doOnNext(buffer::append).doOnComplete(() -> history.add(new AssistantMessage(buffer)))`.

## Embeddings — [07](07-vectors-and-embeddings.md)

- Vector = ordered list of numbers. Each position = a dimension. Order matters.
- Similarity: **Euclidean** (distance), **dot** (direction × size), **cosine** (direction only: 1 same, 0 unrelated, −1 opposite).
- Hand-crafted features don't scale. **Learned embeddings**: start random, train on an objective, similar contexts → similar vectors ("know a word by the company it keeps").
- Individual dimensions are latent and not human-readable. Meaning lives in the overall position, like map coordinates.
- Every embedding is a vector. Not every vector is an embedding.

## Embedding search / recommendations — [08](08-movie-recommender-with-embeddings.md)

- Embedding model → vector. Generative LLM → text. Same Transformer front, different objective.
- "Recommend only from **my** catalog" → don't paste the catalog into prompts. **Precompute** item vectors once, embed only the query.
- It's a **ranking** problem: **represent → store → compare → rank**.
- Search = query vec vs items. "Similar to X" = item vec vs items (zero model calls).
- Content-based (item attributes) vs collaborative filtering (user behaviour).
- `cos(A,B) = A·B / (|A||B|)`.

## Vector DB fundamentals — [09](09-why-vector-databases.md) · [10](10-how-vector-databases-work.md)

| Fact | Number |
|---|---|
| Brute-force KNN cost | O(N × D) per query |
| 10M × 1536-dim | ≈ 15.4 billion ops per query |
| 1 vector, 1536 × float32 | ≈ 6 KB |
| 1M / 100M vectors (raw) | ≈ 6 GB / ≈ 600 GB |

- SQL DBs *can* store vectors. Searching by similarity is the hard part.
- Store **ID + vector + metadata/original text** (embeddings aren't reversible).
- B-trees need order. Vectors have none. k-d trees prune in low-D but fail in high-D (**curse of dimensionality**).
- **ANN**: check only promising candidates. Distances are exact, the misses come from never checking some vectors. Measure with **Recall@K**. Speed ↔ recall.
- **IVF**: k-means clusters → centroid → inverted list. Search the nearest `probes` lists. Boundary misses → more probes.
- **HNSW**: layered sparse proximity graph. Top layers = highways, bottom = all nodes. Descend greedily with a candidate list. `M` = links per node. Fast but RAM-hungry.
- **Quantization**: float32 → float16 (2×) → int8 (4×). **PQ**: 1536 dims → 96 sub-vectors × 256-centroid codebooks → 96 bytes (**64×**). Lossy.
- Vector DB = data + vectors + distance fn + ANN index + **metadata filtering**. Pinecone/Qdrant/Weaviate/Milvus, or **pgvector** (`VECTOR(1536)`, `ORDER BY embedding <=> :q LIMIT 10`, HNSW/IVFFlat indexes).

## RAG — [11](11-rag-customer-support-bot.md)

```
INGEST:  docs → chunk (with overlap) → embed → vector DB (+ text as metadata)
QUERY:   question → embed → top-K similar chunks → prompt = system + chunks + question → LLM
```

- **R**etrieve → **A**ugment the prompt → **G**enerate.
- Why: system prompts can't hold all company knowledge (tokens, context, quality).
- Chunk because one vector per huge document blurs meaning. **Overlap** so boundary facts survive.
- Index dimension must equal embedding output dimension.
- Prompt: "Answer ONLY from the provided information" + an explicit "I don't know" fallback.
- Spring AI: `PagePdfDocumentReader` → `TokenTextSplitter` → `vectorStore.add()` · `vectorStore.similaritySearch(SearchRequest.builder().query(q).topK(4).build())`.

## Advanced RAG — [12](12-advanced-rag.md)

| Technique | Fixes |
|---|---|
| Inspect retrieved chunks first | Tells **retrieval failure** from **generation failure** |
| Structural chunking (keep headings) | Rules split from their exceptions / context |
| Semantic chunking | Topic shifts in unstructured text |
| Tune Top-K (try 3/5/8) | Too few miss rules, too many add noise |
| `similarityThreshold` | Off-topic questions still return "closest" docs |
| Metadata `filterExpression` | Searching the wrong category (don't over-filter) |
| Hybrid search (semantic + BM25) | Exact IDs/SKUs missed by embeddings |
| **RRF** `Σ 1/(k + rankᵢ)`, k≈60 | Can't add cosine and BM25 scores |
| Rerank (retrieve 10–20 → cross-encoder → top 3–5) | Right doc retrieved but ranked low |
| Context engineering | Model prefers a general rule over a specific one |

## Structured output — [13](13-structured-outputs.md)

- Humans want meaning, software wants structure. Define a **schema**, the LLM fills the slots.
- "Please return JSON" is an instruction (breaks on preambles/renames). Structured output is a **contract**.
- Spring: `.call().entity(MeetingDetails.class)`. Python: `client.responses.parse(..., text_format=Model)`.
- Give the model today's date, fix formats, define defaults, and **don't invent** missing fields.
- Structured output = the final result. Tool calling = structured **arguments** for execution.

## MCP — [14](14-mcp-servers.md) · [15](15-mcp-resources-and-prompts.md)

- **Model Context Protocol**: a standard so N AI apps × M systems don't need N×M adapters (like USB/JDBC).
- **Host** (AI app, orchestrator) ⊃ **client** (speaks MCP, like a JDBC driver) ⇄ **server** (exposes capabilities, **no LLM required**).
- Tool calling = *which* tool. MCP = *where tools come from and how to reach them*. Complementary.
- Transport: Streamable HTTP (or stdio for local).

| Primitive | Purpose | Controlled by | Ops | Spring annotation |
|---|---|---|---|---|
| Tools | Do things | **Model** | `tools/list`, `tools/call` | `@McpTool`, `@McpToolParam` |
| Resources | Provide context (URI-addressed) | **Application** | `resources/list`, `resources/read` | `@McpResource` |
| Prompts | Reusable, parameterised workflows | **User** | `prompts/list`, `prompts/get` | `@McpPrompt`, `@McpArg` |

- Client side (Spring): `builder.defaultTools(ToolCallbackProvider mcpTools)` for tools. `McpSyncClient.readResource(...)` / `.getPrompt(...)` for the others.
- Dynamic discovery: add a tool on the server and every host gets it.
- Resources don't auto-enter context. The app picks them via user choice, app state, search or LLM routing.
- A prompt returns **messages**, not an answer.
- Security, approval and auth stay with the **host**. Discovery results can be cached.

---

## Rules of thumb (all lectures)

1. Find the requirement behind the request.
2. The LLM only knows what reaches it. Engineer what reaches it.
3. Fluent ≠ factual. Ground it, validate it, evaluate it.
4. Relevant context beats maximum context.
5. Prompts aren't training and aren't security.
6. The model proposes, the application disposes: execute tools yourself, sandbox, approve risky actions.
7. Ask "what operation does the problem need?", not "which model is smartest?" Ranking ≠ generation.
8. Precompute what's stable. Compute per request only what changes.
9. Similarity is only as good as the representation, and the embedding model creates it, not the vector DB.
10. Debug RAG by inspecting retrieval first, generation second.
11. Use structured output whenever code, not a human, consumes the response.
12. Build capabilities once behind MCP and reuse them from every AI app.
