# 11 — RAG: Building an AI Customer Support Bot

**Video:** [FDE Full Course #11 — RAG Explained with a Real Project](https://www.youtube.com/watch?v=9oeF1Uckuxo) (53m)
**Instructor diagram + code:** [Lecture 11](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2011) (Spring, Python, JS. This lecture has no PDF notes, so these notes are built from the video, the diagram and the code.)

Everything so far comes together here: LLMs (lectures 2–4), embeddings (7–8) and vector databases (9–10). The project is an e-commerce support bot that answers from the company's own policy PDFs, using OpenAI embeddings, Pinecone and Spring AI.

---

## Contents

1. [Where we left off: memory through history](#where-we-left-off-memory-through-history)
2. [The problem: company knowledge doesn't fit in a system prompt](#the-problem-company-knowledge-doesnt-fit-in-a-system-prompt)
3. [The idea: store knowledge as vectors, fetch only what's relevant](#the-idea-store-knowledge-as-vectors-fetch-only-whats-relevant)
4. [R-A-G: Retrieval, Augmented, Generation](#r-a-g-retrieval-augmented-generation)
5. [Chunking and overlap](#chunking-and-overlap)
6. [The project: architecture](#the-project-architecture)
7. [Code: ingesting the knowledge base](#code-ingesting-the-knowledge-base)
8. [Code: answering a question](#code-answering-a-question)
9. [Config: OpenAI + Pinecone](#config-openai--pinecone)
10. [Trying it out, and project ideas](#trying-it-out-and-project-ideas)

---

## Where we left off: memory through history

[1:00](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=60s)

A raw LLM predicts tokens and remembers nothing between calls ("What's my name?" → "I don't know"). In lecture 4 we gave it "memory" by storing the chat history and sending it every time:

```
User: My name is Aditya     → stored
LLM:  Hello Aditya          → stored
User: What is my name?      → [whole history] → LLM → "Your name is Aditya"
```

Every turn the context grows: A, then A+B, then A+B+C… and the context window fills up. This is **short-term memory** that only lives inside the conversation.

---

## The problem: company knowledge doesn't fit in a system prompt

[7:30](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=450s)

The Tomato bot (our Zomato clone) used a system prompt to restrict it to food-delivery topics, so it wouldn't write Python scripts or explain Docker. But a support bot also needs **company-specific facts**:

- Refund policy (e.g. "if food is delayed more than 30 minutes, we refund")
- Return policy
- Payment details and delays
- Order tracking rules

Every company's policies are different, and the LLM was never trained on yours. The first instinct is to paste it all into the system prompt:

```
System prompt:
You are an AI assistant integrated with a food delivery company called Tomato…
Refund policy: …………
Return policy: ………
Payment: ……
```

**Why that breaks:**

- A good system prompt is roughly 500–700 words. Real policy docs can run to 10,000+ words, or hundreds of pages for a big e-commerce site.
- **Tokens:** every single request re-sends all of it, so you pay for it every time.
- **Context window:** a huge input crowds out history and output.
- **Quality:** overloaded prompts make models follow instructions worse and hallucinate more. The relevant line gets buried.
- Someone asking about payments doesn't need the return policy at all.

> We want the model to see **only the information relevant to this question**.

---

## The idea: store knowledge as vectors, fetch only what's relevant

[12:30](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=750s)

We've done this before. In the movie recommender (lecture 8) we embedded every movie description, stored the vectors, embedded the user's query, and returned the most similar movies. That was already **external memory**.

Same trick, except the "items" are now **company documents**:

```
Refund policy.pdf  ┐
Return policy.pdf  ├─→ embedding model → vectors → vector DB
payment.pdf        │
orders.pdf         ┘
```

User asks *"Can I get a refund on shoes I ordered?"* → embed the question → similarity search → the refund-policy and payment sections come back as the closest matches.

One difference from the movie project: we don't hand the raw matches to the user like movie cards. We hand them **to the LLM** as context, so it can write a proper answer:

```
System prompt:   (role, behaviour, rules)
Context:         refund-policy section, payment section     ← retrieved
User query:      Can I get a refund?
        ↓
       LLM → grounded, natural-language answer
```

The next question from a different user retrieves different sections. Nothing needs to stay in the prompt permanently. That's why RAG works like **long-term memory**: the knowledge sits in the vector DB and is pulled in only when needed.

---

## R-A-G: Retrieval, Augmented, Generation

[20:30](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=1230s)

| Letter | Step | What happens |
|---|---|---|
| **R** | **Retrieval** | Embed the user's question, similarity-search the vector DB, fetch the top-K relevant chunks |
| **A** | **Augmented** | Add those chunks to the prompt (the "context") alongside the system prompt and question |
| **G** | **Generation** | The LLM generates an answer grounded in that context |

```
User query
  → embedding model → query vector
  → vector DB similarity search → top-K chunks (as text)
  → prompt = system prompt + retrieved chunks + question
  → LLM → answer
```

Why it's a superpower:

- **Far fewer tokens.** Only relevant information goes in.
- **Grounded answers.** "Answer only from this information" sharply cuts hallucination (lecture 4).
- **Your data, not the model's training data.** New or private docs work with no retraining.
- **Scales.** Hundreds of documents, millions of chunks. The vector DB finds the right ones (lecture 10).

Lecture 8 used the embedding model for SEARCH and the LLM for nothing. RAG uses **both**: embedding model → search, LLM → generation.

---

## Chunking and overlap

[22:30](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=1350s)

### Why not one vector per document?

A 100–200-page policy PDF squeezed into **one** vector loses its meaning. Too many topics get averaged into one point, no matter how many dimensions you have. A question about clothing refunds would be compared against a blur of everything.

So split each document into **chunks** (say ~1,000 words) and embed each chunk:

```
d1: words    1–1000 → v1
d1: words 1001–2000 → v2
…
d2: …                → v50 …
```

Every chunk becomes its own vector in the DB. A query now matches the specific passage that answers it.

### The boundary problem → overlap

Suppose chunk 1 ends mid-sentence:

> …word 998, 999, 1000: **"Refund policy of clothing item is"** | **"3 days…"** ← chunk 2 starts here

The question lives in v1 and the answer lives in v2. Neither chunk alone contains the full fact, and you can't be 100% sure both will be retrieved.

**Fix: overlapping chunks.**

```
chunk 1: words   1–1000 → v1
chunk 2: words 801–1800 → v2     ← 200 words shared with chunk 1
chunk 3: words 1601–2600 → v3
```

With ~200 shared words, a definition or rule near a boundary is very likely to land whole inside at least one chunk. You store a few more vectors, but vector storage is cheap compared with wrong answers.

(Lecture 12 goes much deeper on chunking strategies.)

---

## The project: architecture

[29:00](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=1740s)

An **e-commerce support bot** with five policy PDFs in `src/main/resources/knowledge/`:

```
exchange-policy.pdf
orders-account-help.pdf
payment-refund-policy.pdf
return-policy.pdf
shipping-delivery.pdf
```

Three external pieces:

| Component | Role | Key |
|---|---|---|
| **OpenAI embedding model** (`text-embedding-3-small`) | Turn chunks and questions into vectors | `OPENAI_API_KEY` |
| **Pinecone** vector DB | Store chunk vectors + text, similarity search | `PINECONE_API_KEY` |
| **OpenAI chat model** (`gpt-5-mini`) | Generate the final answer | `OPENAI_API_KEY` |

Two phases:

```
INGEST (once, at startup)
  read every PDF → split into chunks → embed each chunk → store in Pinecone

QUERY (every request)
  question → embed → Pinecone top-K → build prompt with context → LLM → answer
```

Dependencies (`pom.xml`): `spring-ai-starter-model-openai`, `spring-ai-starter-vector-store-pinecone`, `spring-ai-pdf-document-reader`.

---

## Code: ingesting the knowledge base

[32:00](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=1920s)

Plan: (1) read all PDFs, (2) split each into chunks, (3) store the chunks in Pinecone.

```java
@Service
public class ChatbotService {

    private final VectorStore vectorStore;   // Spring AI abstraction — here backed by Pinecone
    private final ChatClient chatClient;

    @Value("classpath*:knowledge/*.pdf")
    private Resource[] policyFiles;

    public ChatbotService(VectorStore vectorStore, ChatClient.Builder builder) {
        this.vectorStore = vectorStore;
        this.chatClient = builder.build();
    }

    @PostConstruct                                        // runs once on startup
    public void loadKnowledgeBase() {
        List<Document> allChunks = new ArrayList<>();

        TokenTextSplitter splitter = TokenTextSplitter.builder()
                .withChunkSize(300)                       // ~300 tokens per chunk
                .build();

        for (Resource resource : policyFiles) {
            PagePdfDocumentReader reader = new PagePdfDocumentReader(resource);
            List<Document> pages  = reader.read();        // one Document per page
            List<Document> chunks = splitter.apply(pages);
            allChunks.addAll(chunks);
        }

        vectorStore.add(allChunks);   // embeds each chunk with OpenAI, then upserts into Pinecone
    }
```

What `vectorStore.add(...)` does behind the scenes: for every chunk, call the embedding model to get a vector, then write `{id, vector, metadata incl. original text}` to Pinecone. Pinecone can also embed text itself with its own hosted models, but since we already use OpenAI, OpenAI does the embedding.

Note: this demo re-ingests on every restart. A real app ingests once (or when docs change) and tracks what's already indexed.

---

## Code: answering a question

[36:00](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=2160s)

Plan: (1) question → vector, (2) similarity search, (3) fetch top 4, (4) hand system prompt + chunks + question to the LLM.

```java
    public String answerUserQuery(String question) {

        // R — Retrieval: Spring AI embeds the question and searches Pinecone
        List<Document> relevantChunks = vectorStore.similaritySearch(
                SearchRequest.builder()
                        .query(question)
                        .topK(4)
                        .build());

        // We need the TEXT of the chunks, not their numbers
        StringBuilder context = new StringBuilder();
        for (Document doc : relevantChunks) {
            context.append(doc.getText()).append("\n\n");
        }

        // A — Augmented: put the retrieved text into the prompt
        String systemPrompt = """
                You are an AI customer support assistant for our e-commerce company.

                Answer the customer using ONLY the company information provided below.

                If the answer is not available in the provided information, say:
                "I don't have that information in the company documents."

                COMPANY INFORMATION
                %s
                """.formatted(context);

        // G — Generation
        return chatClient.prompt()
                .system(systemPrompt)
                .user(question)
                .call()
                .content();
    }
```

Two prompt lines do the heavy lifting against hallucination:

- **"using ONLY the company information provided"** — don't use general knowledge.
- **An explicit fallback answer** — gives the model a safe way to say "I don't know".

Controller:

```java
@RestController
@RequestMapping("/api/v1")
public class ChatbotController {

    @GetMapping("/query")              // GET /api/v1/query?question=Can I get a refund?
    public Map<String, String> query(@RequestParam String question) {
        return Map.of("answer", chatbotService.answerUserQuery(question));
    }
}
```

---

## Config: OpenAI + Pinecone

[43:30](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=2610s)

```properties
# OpenAI — embeddings AND final answer
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-5-mini
spring.ai.openai.embedding.model=text-embedding-3-small
spring.ai.openai.embedding.dimensions=1024

# Pinecone
spring.ai.vectorstore.pinecone.api-key=${PINECONE_API_KEY}
spring.ai.vectorstore.pinecone.index-name=shop-support
spring.ai.vectorstore.pinecone.namespace=policies
```

Keep keys in environment variables, not in code.

### Pinecone concepts

- **Index** — where the vectors live. You create it up front with a **fixed dimension** and a distance metric. Its config can also attach a hosted embedding model (Microsoft, Pinecone's sparse English, NVIDIA…).
- **Dimension must match.** The index was created with **1024** dims, so the embedding model must output 1024. That's why `embedding.dimensions=1024` is set even though `text-embedding-3-small` defaults to 1536. You can't store a 1,536-dim vector in a 1,024-dim index.
- **Namespace** — a logical partition inside an index, a bit like a folder (not literally one). Here everything goes into `policies`, so you could keep, say, `products` separate in the same index.

After startup, the Pinecone console shows the records: an ID, the vector values, and **metadata** containing the original chunk text (e.g. *"Refund method: Prepaid orders are refunded to the…"*). That's the "store the original data alongside the vector" point from lecture 9 in practice.

---

## Trying it out, and project ideas

[49:00](https://www.youtube.com/watch?v=9oeF1Uckuxo&t=2940s)

A small static `index.html` frontend calls `/api/v1/query`. Asking about refunds returns an answer like *"refunds are processed within 2 business days after approval"*. That detail comes straight from the company PDF, not the model's general knowledge. Questions the documents don't cover get the fallback line.

### The full picture

```
Client → Server
          │
          ├─ (startup) PDFs → chunks → OpenAI embeddings → Pinecone
          │
          └─ (query)  question → embed → Pinecone top-K → chunks
                      → system prompt + chunks + question → OpenAI LLM → answer → Client
```

### Build your own: "chat with a book"

Let a user **upload** a PDF (cap the size at 100–200 MB), then on the backend: chunk → embed → store in a vector DB. Write a system prompt like *"answer only from this book"*, and now they can ask questions about any book without the model hallucinating. The same pattern works for any document-grounded Q&A product.

---

## Key takeaways

1. Chat history is short-term memory. It grows every turn and only helps within one conversation.
2. Company knowledge is too big and too specific for a system prompt: wasted tokens, crowded context, worse instruction-following.
3. **RAG** = **R**etrieve relevant chunks via vector search → **A**ugment the prompt with them → **G**enerate a grounded answer.
4. Embedding model → search. LLM → generation. Each does what it's good at.
5. Split documents into **chunks**. One vector per huge doc loses meaning.
6. **Overlap** chunks so facts that cross a boundary stay intact in at least one chunk.
7. Store the original text as metadata with each vector. You can't reverse an embedding.
8. Vector DB index dimension must match the embedding model's output dimension.
9. Prompt with "use ONLY this information" plus an explicit "I don't know" fallback to reduce hallucination.
10. Spring AI: `PagePdfDocumentReader` → `TokenTextSplitter` → `vectorStore.add()` to ingest, `vectorStore.similaritySearch()` + `chatClient` to answer.

## Interview questions

**Q: What is RAG and why use it instead of fine-tuning?**
Retrieval-Augmented Generation: at query time, retrieve relevant text from your own knowledge base and include it in the prompt so the LLM answers from it. It's cheaper and faster to update than fine-tuning, works with private or new data, and reduces hallucination.

**Q: Walk through a RAG request.**
Embed the question with the same model used for the docs, similarity-search the vector DB for top-K chunks, put their text into the prompt with instructions and the question, call the LLM, return the answer.

**Q: Why chunk documents?**
A single vector for a long document blurs many topics and matches poorly. Chunks give focused vectors that match specific questions, and you only send the relevant passages to the LLM.

**Q: What's chunk overlap for?**
So information near a chunk boundary (e.g. a sentence split in half) appears whole in at least one chunk.

**Q: Pinecone rejects my vectors. Likely cause?**
Dimension mismatch between the index and the embedding model output (e.g. a 1024-dim index with 1536-dim embeddings).

**Q: How do you stop a RAG bot from answering outside its documents?**
System prompt: answer ONLY from the provided context, with an explicit fallback phrase when the answer isn't there. Add evaluation and guardrails for production.
