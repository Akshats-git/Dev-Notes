# Forward Deployed Engineer Full Course (GenAI) — Notes

**Source:** [Coder Army — "Forward Deployed Engineer Full Course || GENAI Full Course"](https://www.youtube.com/playlist?list=PLWSlr9idVBc0) by Aditya Tandon
**Length:** 15 videos, about 16 hours total (Sep–Oct 2026)
**Official notes and code:** [adityatandon15/Forward-Deployed-Engineer-Full-Course](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course). Each lecture has a PDF and Excalidraw diagram plus Spring Boot, Python and JavaScript versions of the code.
**Language of the videos:** Hindi/Hinglish. These notes are in English.

---

## How to use these notes

The course is language independent. The instructor teaches with **Spring AI (Java)**, and the repo also has Python (FastAPI/OpenAI SDK) and Node versions of every demo. The code in these notes is mostly Spring AI, with a Python snippet where it helps. The ideas carry over to any stack.

Read in order the first time. Every lecture builds on the one before it. After that, use [16-quick-reference.md](16-quick-reference.md) as a cheat sheet.

Lectures 7–12 (embeddings → vector DBs → RAG) are the core of the course. If you only have time for one block, read those.

---

## Contents

| # | File | Covers | Video |
|---|------|--------|-------|
| 01 | [What an FDE does + AI foundations](01-fde-and-ai-foundations.md) | Request vs requirement, the FDE role, rule-based AI → ML → deep learning → RNNs → attention → GPT | [#1](https://www.youtube.com/watch?v=kBM5UXRbo3U) 1h 22m |
| 02 | [How LLMs work](02-how-llms-work.md) | Tokens, tokenizers, embeddings, Q/K/V self-attention, Transformer layers, logits, softmax, greedy/sampling/temperature | [#2](https://www.youtube.com/watch?v=vZRE_jhA3Qc) 1h 35m |
| 03 | [Calling an LLM from your app](03-calling-an-llm-from-your-app.md) | ChatGPT ≠ LLM, why tools exist, API key/model/input, Postman, token usage, Spring AI `ChatClient` | [#3](https://www.youtube.com/watch?v=SFD22DT0wCI) 48m |
| 04 | [Chatbot: context, memory, hallucinations](04-chatbot-context-memory-hallucinations.md) | Stateless calls, user/assistant/system roles, context window, context management, prompt ≠ training, hallucination | [#4](https://www.youtube.com/watch?v=URucTTDWgI0) 59m |
| 05 | [Tools and AI agents](05-tools-and-ai-agents.md) | Tool calling, `@Tool`, the think-act-observe loop, agent vs workflow, website-builder agent, sandboxing, human-in-the-loop | [#5](https://www.youtube.com/watch?v=jS3cHV43Uxs) 1h 05m |
| 06 | [Streaming](06-streaming.md) | Generation vs delivery, TTFT, SSE vs WebSockets, `Flux<String>`, buffering history, `ReadableStream` + `TextDecoder` | [#6](https://www.youtube.com/watch?v=r4x3Zm4WIWA) 42m |
| 07 | [Vectors and embeddings](07-vectors-and-embeddings.md) | Meaning as numbers, vector space, Euclidean/dot/cosine, feature engineering limits, learned embeddings | [#7](https://www.youtube.com/watch?v=b55entyKoms) 1h 11m |
| 08 | [Project: movie recommender](08-movie-recommender-with-embeddings.md) | Embedding model vs LLM, precomputation, semantic search + "similar movies", represent→store→compare→rank | [#8](https://www.youtube.com/watch?v=EQihXl-HVUA) 52m |
| 09 | [Why vector databases](09-why-vector-databases.md) | Storage vs search, KNN, O(N×D) cost, memory maths, why B-trees fail, k-d trees, curse of dimensionality | [#9](https://www.youtube.com/watch?v=5mXvdCDv2eM) 1h 05m |
| 10 | [How vector DBs work](10-how-vector-databases-work.md) | ANN, Recall@K, IVF (centroids, probes), HNSW (layered graph), quantization, product quantization, pgvector | [#10](https://www.youtube.com/watch?v=YitjkFRN20Q) 1h 34m |
| 11 | [RAG: support bot project](11-rag-customer-support-bot.md) | Why not stuff the system prompt, Retrieval-Augmented-Generation, chunking + overlap, Pinecone + Spring AI | [#11](https://www.youtube.com/watch?v=9oeF1Uckuxo) 53m |
| 12 | [Advanced RAG](12-advanced-rag.md) | Retrieval vs generation failures, structural/semantic chunking, Top-K, thresholds, metadata filters, hybrid search, RRF, reranking | [#12](https://www.youtube.com/watch?v=_FygCwbpreU) 1h 12m |
| 13 | [Structured outputs](13-structured-outputs.md) | Humans want meaning, software wants structure, schemas, `.entity(Class)`, structured output vs tool calling | [#13](https://www.youtube.com/watch?v=yKsgSHJ3Jfo) 41m |
| 14 | [MCP servers](14-mcp-servers.md) | N×M integration problem, host/client/server, MCP vs tool calling, `tools/list` & `tools/call`, Task MCP server demo | [#14](https://www.youtube.com/watch?v=hSxdqkKZoqw) 1h 10m |
| 15 | [MCP resources and prompts](15-mcp-resources-and-prompts.md) | Resources (URIs, list/read, templates), prompts (arguments, list/get), model vs app vs user control | [#15](https://www.youtube.com/watch?v=MUttM7fIqO4) 53m |
| 16 | [Quick reference](16-quick-reference.md) | Every core idea, pipeline and rule of thumb on one page | — |

> The playlist shows a 16th entry, but it's currently unavailable/hidden on YouTube, so it isn't covered. Re-run the notes when it's published.

---

## The short summary

A **Forward Deployed Engineer** sits next to a customer's real problem, works out what the actual requirement is (usually not "build us a chatbot"), and ships a production system around it. In GenAI work that system is mostly *not* the model.

An **LLM** does one thing: given tokens so far, it predicts a probability distribution over the next token, picks one, and repeats. Text is split into **tokens**, tokens become **embeddings**, **self-attention** (Query/Key/Value) mixes in context across many **Transformer** layers, and the last vector becomes **logits → softmax → a sampled token**. Temperature only changes how we pick.

The model is **stateless** and **only knows what reaches it**. Memory is the app re-sending history. Company knowledge has to be retrieved and put into the prompt. Live data, maths and actions come from **tools** the app executes. All of it competes for a finite **context window** measured in tokens, and fluent output can still be **hallucinated**.

Give the model tools and a loop (goal → decide → act → observe → repeat) and you have an **agent**. The model decides *what* it wants to do. The application decides what it's *allowed* to do: sandboxing, narrow tools, human approval.

**Embeddings** place meaning in a vector space so similarity becomes geometry (cosine). That alone builds search and recommendations without any LLM. At scale you need a **vector database**, because brute-force KNN is O(N×D) and tree indexes die in high dimensions. **ANN** indexes (IVF, HNSW) trade a little recall for huge speed, and **quantization/PQ** shrinks storage.

**RAG** combines both: embed the question, retrieve the most relevant chunks of *your* documents, put them in the prompt, and let the LLM answer from them. **Advanced RAG** fixes each stage: better chunking, Top-K/threshold/metadata filters, hybrid (semantic + BM25 via RRF) search, and cross-encoder reranking.

**Structured output** turns LLM text into typed objects your code can act on. **MCP** standardises how any AI app discovers and uses external **tools** (model-controlled), **resources** (application-controlled) and **prompts** (user-controlled), so capabilities are built once and reused everywhere.
