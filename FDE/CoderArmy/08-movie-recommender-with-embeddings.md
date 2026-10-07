# 08 — Project: A Netflix-Style Movie Recommender With Embeddings

**Video:** [FDE Full Course #8 — Build a Netflix Recommendation System with Vector Embeddings](https://www.youtube.com/watch?v=EQihXl-HVUA) (52m)
**Instructor notes + code:** [Lecture 08](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2008) (Spring, Python, JS + a frontend)

Lecture 7 was theory. This one puts it to work: semantic movie search ("a sad romantic movie") and "show similar movies", with **no generative LLM deciding anything**. Just an embedding model, vectors, cosine similarity and a sort.

---

## Contents

1. [The demo](#the-demo)
2. [Embedding model vs generative LLM](#embedding-model-vs-generative-llm)
3. [Why not just ask an LLM to recommend?](#why-not-just-ask-an-llm-to-recommend)
4. [Thinking from first principles: precompute](#thinking-from-first-principles-precompute)
5. [It's a ranking problem, not a generation problem](#its-a-ranking-problem-not-a-generation-problem)
6. [Two features, one mechanism](#two-features-one-mechanism)
7. [What kind of recommender this is](#what-kind-of-recommender-this-is)
8. [Represent → Store → Compare → Rank](#represent--store--compare--rank)
9. [The code](#the-code)

---

## The demo

[0:00](https://www.youtube.com/watch?v=EQihXl-HVUA&t=0s)

A small Netflix-like page shows a catalog of movies (title + description). Two things work:

- **Search by description.** Type "romantic movie sad" → Titanic, DDLJ, Forrest Gump.
- **Click a movie** (e.g. Interstellar) → "similar movies" like Gravity, The Martian, The Matrix, each with its cosine-similarity score pinned on the card.

---

## Embedding model vs generative LLM

[5:00](https://www.youtube.com/watch?v=EQihXl-HVUA&t=300s)

```
A: "An astronaut is stranded on Mars."
B: "A man must survive alone on another planet."
C: "A detective investigates a murder in London."
```

Humans see A ≈ B and C far away, even though A and B share almost no words (astronaut/stranded/Mars vs man/survive/alone/planet). We need `language → numbers → maths`, and that's what an embedding gives you:

```
"A stranded astronaut tries to survive on Mars." → [0.014, -0.031, 0.087, ...]
```

Reminder: individual dimensions aren't "dimension 17 = sadness". Meaning is spread across the whole vector. What matters is that **similar meanings land close together**.

### Same front half, different job

Both kinds of model start the same way:

```
"The Martian is amazing" → Tokenizer → Token IDs → Neural network (Transformer)
```

Then they diverge:

| | Generative LLM | Embedding model |
|---|---|---|
| Question it answers | What token comes next? | What's a useful numeric representation of this whole input? |
| Given "The astronaut opened the…" | door 0.31, hatch 0.27, airlock 0.19 → pick → repeat | — |
| Given "The astronaut opened the airlock." | — | `[-0.024, 0.182, ...]` (fixed size) |
| Output | **Text** | **A vector** |
| "What is the capital of India?" | "New Delhi" | `[0.023, -0.193, 0.082, ...]` |

> An embedding model is a language model specialised for **representation**. A generative LLM is specialised for **generation**. The embedding model represents information. The LLM generates it.

Providers sell them separately. The project uses OpenAI's `text-embedding-3-small`, which outputs **1536-dimension** vectors (the video prints `embedding.length` and gets 1536).

---

## Why not just ask an LLM to recommend?

[9:00](https://www.youtube.com/watch?v=EQihXl-HVUA&t=540s)

An LLM can absolutely recommend movies. "I loved Interstellar, suggest three" → Arrival, The Martian, Gravity. The real distinction is:

```
KNOWLEDGE / GENERATION    vs    SEARCHING MY DATA
```

**Scenario 1 — generic recommendation.** "Movies like Interstellar." The LLM already knows thousands of films from training. Using it is fine.

**Scenario 2 — recommend from *my* catalog.** You run *CoderArmyFlix* with 7,328 movies: some regional, indie, internal, or released after the model's training cutoff. "Recommend the 3 most relevant, **only from our catalog**." That's a different problem.

### The naive approach: send the whole catalog

```
Here are my 7,328 movies… Movie 1… Movie 2… … Movie 7328…
The user likes Interstellar. Recommend three.
```

It technically works for 10 or 200 movies. It breaks because of:

1. **Context size.** 10,000 movies, 100,000 products, 5 million documents won't fit or make sense in every request.
2. **Repeated work.** A million users means a million times the model re-reads a catalog that barely changes.
3. **You don't need generation.** You want a ranked list, not prose.
4. **Cost and latency.** Generation is token by token and you pay for every token, every request.

---

## Thinking from first principles: precompute

[11:00](https://www.youtube.com/watch?v=EQihXl-HVUA&t=660s)

Ask: **what changes often?**

- Movie descriptions → rarely.
- User queries → constantly.

So split the work:

```
STABLE DATA (once):     movie description → embedding model → movie vector → save it
CHANGING DATA (per request): user query → embedding model → query vector
THEN:                   query vector ⟷ mathematical comparison ⟷ movie vectors
```

This is **precomputation**:

> If something expensive doesn't change often, calculate it once and reuse it.

Interstellar's description is the same tomorrow, so there's no reason to embed it again. Save `Interstellar → vector` and reuse it. You've turned repeated AI processing into reusable numbers.

---

## It's a ranking problem, not a generation problem

The actual question is: *which movie is closest in meaning to this query?* You don't need *"The Martian would be an excellent choice because…"*. You need:

```
Movie A → 0.87
Movie B → 0.81
Movie C → 0.42
```

Then sort. Once meaning is vectors, ranking is just maths.

> **Think like an engineer.** Don't ask "which AI model is smartest?" Ask "**what operation does my problem actually require?**"
> - Summarise an article → generation
> - Write an email → generation
> - Which item in my dataset is closest to this query → embeddings + similarity

And it's cheap:

```
User query → ONE embedding request → local vector comparison → results
```

No generative LLM call for the recommendation itself.

### Combining both (the seed of RAG)

"I want an emotional space movie where the protagonist is isolated."

```
User query → embedding search → retrieve: The Martian, Gravity, Interstellar
           → (optional) give those to an LLM → "Based on your preference, The Martian would be a strong match…"
```

Each part does what it's best at: **embedding model → SEARCH, LLM → GENERATION**. That split is the foundation of **Retrieval-Augmented Generation** (lecture 11).

---

## Two features, one mechanism

[13:00](https://www.youtube.com/watch?v=EQihXl-HVUA&t=780s)

Each movie has a **title** and a **description**:

> **Interstellar** — "A team of astronauts travels through a wormhole in search of a new home for humanity."
> **The Martian** — "An astronaut becomes stranded on Mars and must use science and ingenuity to survive."

We embed the **description**, not the title. A title alone isn't descriptive enough, and bad input gives bad vectors.

### Feature 1 — Semantic search

The user doesn't know the title: *"Something about a person trying to stay alive far away from Earth."* Keyword search struggles: the description says "stranded astronaut, Mars, survive", the query says "person, stay alive, far away from Earth". Different words, same meaning.

```
USER QUERY → embedding model → query vector
  → compare with every movie vector → similarity scores
  → sort → top 3: The Martian, Gravity, Interstellar
```

Searching by **meaning**, not exact words.

### Feature 2 — Similar movies

The user clicks Interstellar → "Show similar". There's no query to embed. Interstellar **already has** a vector:

```
Interstellar vector → compare with all other movie vectors → sort → The Martian, Gravity, Arrival
```

This flow needs **zero** model calls at request time.

### Same problem underneath

```
SEARCH:          query vector  vs movie vectors
RECOMMENDATION:  movie vector  vs movie vectors
```

Only the starting vector changes. One similarity system powers both features.

---

## What kind of recommender this is

**Content-based recommendation.** Recommendations come from the *content* of the movie: description → embedding → compare meaning → similar movies.

**Not** collaborative filtering ("users who watched Interstellar also watched X"). That uses **behaviour**:

```
User A watched X and Y. User B watched X. → Y may be relevant for B.
```

Real systems (Netflix, Amazon's "you may also buy") combine many signals: content, watch history, ratings, clicks, other users' behaviour, freshness, popularity and business rules. This demo deliberately isolates one: **embedding similarity**.

---

## Represent → Store → Compare → Rank

The whole thing, before any Spring code:

1. **REPRESENT** — meaning → numbers. `"The Martian is about survival on Mars." → [0.12, -0.43, 0.08, ...]`
2. **STORE** — keep them. `Interstellar → vec, The Martian → vec, …` Here a `List<Movie>` in memory. In production, a real store (lectures 9–10).
3. **COMPARE** — take a starting vector (query or movie) and compute its similarity to every other vector.
4. **RANK** — sort by score (0.89, 0.84, 0.78, 0.21…) and return the top K.

```
Phase 1 — prepare catalog (at startup):
  for each movie: description → embedding → store vector

Phase 2 — at request time:
  SEARCH:          query → embed → compare → rank
  RECOMMENDATION:  existing movie vector → compare → rank
```

The pattern applies to products, documents, news, courses, jobs, images, support tickets, knowledge bases and code. Whenever the problem is *"find items in my dataset whose meaning is closest to this input"*, the answer is **represent → store → compare → rank**.

---

## The code

[24:00](https://www.youtube.com/watch?v=EQihXl-HVUA&t=1440s)

### Config

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.embedding.model=text-embedding-3-small
```

Spring AI gives you an `EmbeddingModel` interface, so you can plug in any provider's embedding model.

### Models

```java
public class MovieData  { String title; String description; }                     // from movies.json
public class Movie      { String title; String description; float[] embedding; }   // + vector
public class MovieMatch { String title; String description; double match; }       // result + score
```

Storing embeddings on the object in memory is for the demo. In production they go in a database.

### Phase 1 — embed the catalog at startup

```java
@Service
public class MovieService {

    private final EmbeddingModel embeddingModel;
    private final List<Movie> moviesEmbedding = new ArrayList<>();

    @PostConstruct                                  // runs once when the app starts
    public void initializeMovies() throws IOException {
        List<MovieData> movieDataList = jsonMapper.readValue(
                new ClassPathResource("movies.json").getInputStream(),
                new TypeReference<>() {});

        for (MovieData m : movieDataList) {
            float[] embedding = embeddingModel.embed(m.getDescription());
            moviesEmbedding.add(new Movie(m.getTitle(), m.getDescription(), embedding));
        }
    }
```

### Phase 2a — semantic search

```java
    public List<MovieMatch> search(String query) {
        float[] queryEmbedding = embeddingModel.embed(query);       // the only model call

        List<MovieMatch> matches = new ArrayList<>();
        for (Movie movie : moviesEmbedding) {
            double sim = cosineSimilarity(queryEmbedding, movie.getEmbedding());
            matches.add(new MovieMatch(movie.getTitle(), movie.getDescription(), sim));
        }
        sortBySimilarity(matches);                                  // highest first
        return topKMatches(matches, 3);
    }
```

### Phase 2b — similar movies (no model call)

```java
    public List<MovieMatch> similarMovies(String title) {
        Movie selected = findMovie(title);
        List<MovieMatch> matches = new ArrayList<>();
        for (Movie movie : moviesEmbedding) {
            if (movie.getTitle().equalsIgnoreCase(title)) continue;   // skip itself
            double sim = cosineSimilarity(selected.getEmbedding(), movie.getEmbedding());
            matches.add(new MovieMatch(movie.getTitle(), movie.getDescription(), sim));
        }
        sortBySimilarity(matches);
        return topKMatches(matches, 3);
    }
```

### Cosine similarity by hand

```
cos(A, B) = (A · B) / (|A| × |B|)
A · B = Σ aᵢbᵢ          |A| = √(Σ aᵢ²)
```

```java
    private double cosineSimilarity(float[] a, float[] b) {
        double dot = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.length; i++) {
            dot   += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        if (normA == 0 || normB == 0) return 0.0;
        return dot / (Math.sqrt(normA) * Math.sqrt(normB));      // between -1 and 1
    }
```

`sortBySimilarity` sorts descending by `match`. `topKMatches` takes the first K.

### Controller

```java
@RestController
@RequestMapping("/movies")
public class MovieController {

    @GetMapping("/search")                 // GET /movies/search?query=sad romantic movie
    public List<MovieMatch> search(@RequestParam String query) {
        return movieService.search(query);
    }

    @GetMapping("/{title}/similar")        // GET /movies/Interstellar/similar
    public List<MovieMatch> similarMovies(@PathVariable String title) {
        return movieService.similarMovies(title);
    }
}
```

The frontend calls `/movies/{title}/similar` when you click a card.

### Notice the scaling problem

`search()` compares the query against **every** movie, one by one. Fine for 50 movies. For 10 million vectors of 1,536 dimensions that's ~15 billion multiplications per query, plus keeping them all in RAM. That's exactly what **vector databases** solve. Next lecture.

---

## Key takeaways

1. An embedding model turns input into a fixed-size vector. A generative LLM produces text. Same Transformer front half, different objective.
2. LLMs are fine for *generic* recommendations. For "only from **my** data", use embeddings + similarity.
3. Sending your whole catalog to an LLM every request hits context limits, repeats work, and costs money and time.
4. **Precompute** what's stable (catalog embeddings). Compute only what changes (the query) at request time.
5. Recommendation and search are **ranking** problems. Ask what operation the problem needs, not which model is smartest.
6. Search (query vs items) and recommendation (item vs items) use the same mechanism.
7. This is **content-based** recommendation. Collaborative filtering uses user behaviour instead. Real systems combine both.
8. **Represent → Store → Compare → Rank** works for any "find the closest item in my data" problem.
9. Embedding model for search + LLM for wording = the seed of RAG.

## Interview questions

**Q: Embedding model vs LLM?**
Both tokenize and run a neural network, but an embedding model outputs a fixed-size vector for the whole input, while a generative LLM outputs text token by token.

**Q: Why not paste the catalog into the LLM prompt?**
Context limits, re-processing the same data for every user, paying for generation you don't need, and latency. Precompute embeddings once and do cheap vector maths per query.

**Q: Content-based vs collaborative filtering?**
Content-based uses item attributes (here, description embeddings). Collaborative uses user–item interactions ("people who watched X also watched Y").

**Q: How many model calls does "similar to Interstellar" need at request time?**
Zero. Interstellar's vector is precomputed. It's pure vector comparison.

**Q: What breaks when the catalog grows to millions?**
Brute-force comparison against every vector gets slow and memory-heavy. You need a vector database with approximate nearest-neighbour indexes.
