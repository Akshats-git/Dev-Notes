# 07 — Vectors and Embeddings

**Video:** [FDE Full Course #7 — Vectors & Embeddings Explained](https://www.youtube.com/watch?v=b55entyKoms) (1h 11m)
**Instructor notes:** [Lecture 07 PDF](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2007)

Embeddings are probably the single most important idea for the rest of this course: recommendations, semantic search, vector databases and RAG all rest on them. This lecture builds them from scratch, starting with a toy movie recommender and ending at learned embeddings.

---

## Contents

1. [Storing a word ≠ understanding it](#storing-a-word--understanding-it)
2. [Attempt 1: one number per movie](#attempt-1-one-number-per-movie)
3. [Attempt 2: many numbers — vectors](#attempt-2-many-numbers--vectors)
4. [Dimensions and vector space](#dimensions-and-vector-space)
5. [Measuring similarity](#measuring-similarity)
6. [The problem: who picks the dimensions?](#the-problem-who-picks-the-dimensions)
7. [Let the machine learn the space: embeddings](#let-the-machine-learn-the-space-embeddings)
8. [How training learns embeddings](#how-training-learns-embeddings)
9. [What the numbers mean (and don't)](#what-the-numbers-mean-and-dont)
10. [Vector vs embedding](#vector-vs-embedding)
11. [Embedding models and what you can do with them](#embedding-models-and-what-you-can-do-with-them)

---

## Storing a word ≠ understanding it

[0:00](https://www.youtube.com/watch?v=b55entyKoms&t=0s)

Say "Batman" to a human and a cloud of associations lights up: superhero, Gotham, Bruce Wayne, DC, dark, serious, vigilante, rich, action. A computer stores `B a t m a n` (as character codes, then binary) and knows none of that.

> Representing symbols is not the same as representing meaning.

Traditional software is great at **identity**:

```java
"Batman".equals("Batman")    // true
"Batman".equals("Superman")  // false
```

String comparison checks codes character by character. It tells you whether two strings are *identical*, not whether they're *related*.

```
"I forgot my password."
"How can I recover my login credentials?"
```

As strings: completely different. In meaning: almost the same. That's the central problem:

> How do we turn **meaning** into something maths can work with?

Models do maths: addition, multiplication, distance, angles, probabilities, matrices. You can't do much maths on "Batman" or "Pizza", so we need numbers. Assigning random IDs (Batman = 23, Superman = 43, Banana = 13) doesn't help. The numbers have to encode real relationships. This problem is older than LLMs, and it applies to words, sentences, documents, movies, users, songs, products, images, proteins and web pages.

---

## Attempt 1: one number per movie

[10:00](https://www.youtube.com/watch?v=b55entyKoms&t=600s)

Forget language. You're building early Netflix's recommendation engine. Give each movie an **action score**:

| Movie | Action |
|---|---|
| Avengers | 10 |
| Batman | 9 |
| Interstellar | 6 |
| Titanic | 2 |
| Hera Pheri | 1 |

For the first time, a property of a movie is **maths**. Similarity is now a calculation:

```
|Batman − Avengers| = |9 − 10| = 1   ← close
|Batman − Titanic|  = |9 − 2|  = 7   ← far
```

Someone who's never seen any of these films could still tell you Batman is closer to Avengers. If you liked Batman, recommend whatever has the smallest distance. Sort and show the top 3.

It's a **1-dimensional space**, a number line:

```
0 ─────────────────────────────────────────── 10
Hera Pheri  Titanic        Interstellar   Batman Avengers
    1          2                 6           9      10
```

Once things have *positions*, distance becomes meaningful.

**Limitation:** if Titanic = 2 and Hera Pheri = 2, the system thinks they're very similar. One's a romantic drama, the other's a comedy. **One number can capture only one property.**

---

## Attempt 2: many numbers — vectors

[19:00](https://www.youtube.com/watch?v=b55entyKoms&t=1140s)

Add a second property, comedy:

| Movie | Action | Comedy |
|---|---|---|
| Batman | 9 | 1 |
| Avengers | 10 | 7 |
| Titanic | 2 | 1 |
| Hera Pheri | 2 | 10 |
| Interstellar | 6 | 1 |

Batman is now `[9, 1]`. Titanic `[2, 1]`. Avengers `[10, 7]`. Plot action on x and comedy on y, and each movie is a point on a plane. Titanic and Hera Pheri are now far apart.

> **A vector is an ordered collection of numbers representing something.**

Add more properties:

```
Batman = [9,  // action
          1,  // comedy
          2,  // romance
          9,  // darkness
          8,  // violence
          7]  // mystery
```

**Order matters.** If dimension 1 = action and dimension 2 = comedy, then `[9, 2]` (high action, low comedy) and `[2, 9]` (low action, high comedy) are different movies. Every position has a fixed role.

---

## Dimensions and vector space

[31:00](https://www.youtube.com/watch?v=b55entyKoms&t=1860s)

Each value is one **dimension**. `[9]` is 1-D, `[9, 2]` is 2-D, `[9, 2, 3]` is 3-D. Real embeddings have 768, 1024, 1536 or more.

You can draw up to 3 dimensions. Beyond that you can't picture it, but the maths works exactly the same. The distance formula from school extends to any number of dimensions.

> High-dimensional spaces are hard for humans to visualise, not hard for computers to calculate with.

A set of vectors lives in a **vector space**. Once things have locations, the question changes:

| Traditional systems ask | Vector systems ask |
|---|---|
| Same or different? (`true` / `false`) | **How** similar? (`0.92`, `0.71`, `0.22`) |

Continuous similarity is the foundation of vector search.

---

## Measuring similarity

[32:00](https://www.youtube.com/watch?v=b55entyKoms&t=1920s)

Three common ways. Understand the intuition. You don't need to memorise formulas.

### 1. Euclidean distance — straight-line distance

```
d = √((x₂ − x₁)² + (y₂ − y₁)²)          (and so on, for every dimension)
```

Smaller distance = closer = more similar. If the dimensions are meaningful (action, comedy), geometric closeness means semantic similarity.

**Distance isn't magic. The representation is what matters.** Represent movies as `[letters_in_title, release_month]` and Batman may land right next to Titanic. The maths is correct and the result is useless.

> Bad representation → bad similarity.

### Why direction can matter more than distance

```
dimensions = [action preference, romance preference]
User A = [9, 1]
User B = [90, 10]
```

Euclidean distance says they're far apart. But the **ratio** is identical (9:1). Both strongly prefer action over romance, just with different intensity (maybe B rates everything on a bigger scale or watches far more). The vectors point in the **same direction**. Sometimes the *pattern* matters more than the *size*.

### 2. Dot product — direction + magnitude

```
A = [2, 3], B = [4, 5]
A · B = (2×4) + (3×5) = 23
```

Captures both how aligned the vectors are and how large they are. Vectors pointing the same way give strong positive values. Very different directions give small or negative ones.

### 3. Cosine similarity — direction only

> How similar is the **direction** of these two vectors?

```
cos(θ) = (A · B) / (|A| × |B|)
```

It looks at the angle and ignores length:

| Value | Meaning |
|---|---|
| 1 | Same direction |
| 0 | Unrelated (perpendicular) |
| −1 | Opposite directions |

Users A and B above get cosine similarity 1.

**Recommendation intuition:** a user's preference vector points ↗. Movie A points ↗, Movie B ↑, Movie C ←. Movie A is most aligned → recommend it. That's a recommendation engine without any LLM.

### Which one?

| Metric | Main idea | Question it answers |
|---|---|---|
| Euclidean | Straight-line distance | How physically close are they? |
| Cosine | Angle / direction | Do they represent a similar pattern, regardless of size? |
| Dot product | Direction × magnitude | Are they aligned, and how strongly? |

It depends on how the vectors were made and what you're measuring. Embedding models usually document which metric they were trained for.

---

## The problem: who picks the dimensions?

[42:00](https://www.youtube.com/watch?v=b55entyKoms&t=2520s)

So far *we* chose action, comedy, romance, darkness, and *we* assigned the scores (subjectively). The computer just ran a distance formula. Designing features by hand is **feature engineering**, and it has big problems.

**1. How many features?** Interstellar could be space, science, family, fatherhood, time, love, survival, philosophy, physics, sadness, adventure, visual spectacle, hope, sacrifice, isolation, VFX, runtime… Different people would pick different ones. There could be thousands.

**2. Features depend on the task.** Same movie, different needs:

| Task | Useful features |
|---|---|
| Recommendation | genre, tone, pace |
| Box-office prediction | budget, actor popularity, release date, marketing spend |
| Parental controls | violence, language, sexual content |

There's no universal feature list.

**3. Language makes it impossible.** "King" → human, male, royalty, authority, wealth, leader, power. "Queen" overlaps heavily. "Apple" needs fruit, food, company, technology, sweet, red, round. "Java" is a programming language, an island *and* coffee. Every word needs its own set of dimensions, and most are irrelevant to other words. How much "food" is a queen? You'd have to fill in zeros everywhere.

**4. Context changes meaning.** "bank" (money) vs "river bank". Plus ambiguity, grammar, tone, intent, metaphor. Hand-designing features for all of language is basically impossible.

---

## Let the machine learn the space: embeddings

[50:00](https://www.youtube.com/watch?v=b55entyKoms&t=3000s)

What we want:

```
Batman ≈ Superman     dog ≈ puppy     king ≈ queen
Batman ≠ banana       dog ≠ database  king ≠ refrigerator
```

…without humans designing thousands of dimensions. The scalable answer:

> **Let the machine learn the vector space itself.**

The key shift: **we don't care what each individual number means. We care that the whole space behaves meaningfully.**

```
similarity(Batman, Superman) > similarity(Batman, Titanic)
```

That's all we ask of the model. The video puts it as: "I just want you to tell me king–queen is high similarity and king–banana is low. I don't need to know what dimension 1 means."

---

## How training learns embeddings

[52:00](https://www.youtube.com/watch?v=b55entyKoms&t=3120s)

### Start random

Vocabulary of four words, 2-D vectors, random values (kept between −1 and +1):

```
king   = [ 0.2, -0.7]
queen  = [-0.8,  0.3]
banana = [ 0.9,  0.1]
apple  = [-0.1, -0.5]
```

Meaningless. King might be closer to banana than to queen. That's fine, since training hasn't happened yet.

> Embeddings don't start meaningful. Useful structure **emerges** during training.

### Give it an objective

A model doesn't learn because you say "understand language". It learns parameters that help it do some **task**. Meaningful representations show up as a side effect.

### Meaning through context

```
The king ruled the kingdom.     The queen ruled the kingdom.
The king lived in the palace.   The queen lived in the palace.
The king wore the crown.        The queen wore the crown.

I ate a banana.                 I ate an apple.
The banana is a fruit.          The apple is a fruit.
```

King and queen show up in the same surroundings. So do banana and apple. That's a famous idea from linguistics:

> **"You shall know a word by the company it keeps."** (J.R. Firth)

Words used in similar contexts tend to have related meanings. The instructor tells a story here: a school teacher asked him the meaning of a word he didn't know, and he worked it out from the sentence around it. Humans do this all the time. Even without knowing what "male" means, if you see "the gender of king is male" you'd guess queen behaves similarly in most sentences.

### A toy objective

> Given one word, predict nearby words.

"The king wears a crown" → input `king` → targets `the, wears, a, crown`. "The queen wears a crown" → input `queen` → almost the same targets. If king and queen need to produce similar predictions, it helps for their vectors to become similar.

```
Before:  king ●                ● queen
After:          king ● ● queen          banana ●  (somewhere else entirely)
```

Nobody typed "king is similar to queen". It came from patterns in the data.

### The training loop

1. Start with random vectors.
2. Use them to predict.
3. Compare with the right answer. (Predicted "banana" for king's neighbour? Wrong, high loss.)
4. Compute the error (**loss**).
5. Nudge the parameters slightly (gradient descent).
6. Repeat, billions of times.

```
RANDOM SPACE                        LEARNED SPACE
king      banana                    king ● ● queen
     dog                            dog  ● ● cat
queen         apple      ──►        banana ● ● apple
  cat
```

### The embedding table

After training you have a lookup table, which is itself model parameters:

| Word | d1 | d2 | d3 |
|---|---|---|---|
| king | 0.21 | −0.71 | 0.45 |
| queen | 0.19 | −0.68 | 0.49 |
| dog | −0.51 | 0.22 | 0.78 |
| cat | −0.48 | 0.19 | 0.73 |
| banana | 0.89 | 0.54 | −0.13 |
| apple | 0.84 | 0.57 | −0.09 |

See `dog` → fetch `[-0.51, 0.22, 0.78]`. That's an **embedding lookup**. Look at the numbers: king/queen, dog/cat and banana/apple are near-twins.

### Why this helps the network

"The king lives in the ___" → palace, castle, kingdom. "The queen lives in the ___" → mostly the same. If king and queen had unrelated vectors, the network would learn the same behaviour twice. If `king ≈ queen`, it reuses the same learned machinery.

> Similar representations are useful because similar inputs often need to produce similar behaviour.

The model isn't consciously thinking "king and queen are similar". It's statistics over huge data. Whether that counts as "understanding" is the same philosophical question as the Turing test in lecture 1.

---

## What the numbers mean (and don't)

[1:05:00](https://www.youtube.com/watch?v=b55entyKoms&t=3900s)

Does the machine "discover features"? Sort of, but carefully. A hand-made vector `[royalty, gender, authority]` has readable coordinates. A learned one:

```
king  = [0.31, -0.84, 0.17, 0.66, ...]
queen = [0.29, -0.78, 0.21, 0.61, ...]
```

It's usually **wrong** to assume dimension 1 = royalty, dimension 2 = gender. A concept like "royalty" may be spread across dimensions 4, 18, 91, 207…, and those same dimensions help represent many other concepts too. **Meaning lives in the overall configuration, not in one clean coordinate.**

### The map analogy

```
Delhi = [28.61, 77.20]
```

Neither number means "Delhi-ness". 77.20 doesn't mean "capital city". Together they put Delhi at a meaningful *location*, and what matters is its relationship to other locations. Embeddings work the same way, in hundreds or thousands of dimensions.

### Latent space

**Latent** ≈ hidden, not directly observed, not manually specified. `[action, romance, comedy]` are explicit dimensions. `[0.21, -0.81, 0.44, …]` are **learned internal factors**. Hence *latent space*, *latent representation*, *latent factors*.

---

## Vector vs embedding

> Every embedding is a vector. Not every vector is an embedding.

| | Vector | Embedding |
|---|---|---|
| What | The mathematical container: `[2.1, 4.7, 1.3]` | A **learned** representation placing an object in a continuous space |
| Example | `[height, weight, age]` — hand-picked → a *feature vector* | `[0.23, -0.81, 0.47, …]` from a trained model |

"Vector embedding" just means: a model created this vector for you.

---

## Embedding models and what you can do with them

[1:07:00](https://www.youtube.com/watch?v=b55entyKoms&t=4020s)

Anything can be embedded: words, sentences, documents, movies, users, songs, images, products. The question is always: *can we learn a space where relationships between these things become geometry?*

An **embedding model** is a function:

```
"Batman is a dark superhero movie"
        ↓  embedding model
[0.12, -0.37, 0.91, ..., 0.22]

f(text) → ℝ⁷⁶⁸     (or ℝ¹⁰²⁴, ℝ¹⁵³⁶ … dimensionality is part of the model's design)
```

Once things are vectors, you can do:

- Similarity search / nearest-neighbour search
- Clustering
- Classification
- Ranking
- **Recommendation** (lecture 8)
- **Retrieval** (lectures 9–12)

```
Human concepts → vector representation → mathematics → ML applications
```

LLMs use the same idea internally (lecture 2). To predict the next token, the model compares its context vector with vocabulary vectors and picks the most similar.

---

## The whole journey

```
REAL-WORLD OBJECT: Batman
  ↓ need numbers
MANUAL VECTOR: [action=9, romance=2, darkness=9]
  ↓ doesn't scale
Change the question from "which dimensions do we create?"
                      to "what task should the system learn?"
  ↓ training objective + data + gradient descent
LEARNED VECTOR: [0.23, -0.81, 0.47, ...]
  ↓
EMBEDDING
```

> **An embedding is not the meaning itself. It is a position inside a learned mathematical space where useful relationships in meaning or behaviour become relationships in geometry.**

## Key takeaways

1. Computers store symbols, not meaning. Exact string matching can't see "forgot password" ≈ "recover credentials".
2. Turning things into numbers enables maths. Random IDs don't, meaningful numbers do.
3. One number = one property. A vector (ordered list) captures many properties. Order matters.
4. Vectors live in a vector space. Similarity becomes continuous ("how similar?"), not binary.
5. Euclidean = straight-line closeness. Cosine = direction/pattern. Dot product = direction × magnitude.
6. Similarity is only as good as the representation.
7. Hand-crafted features don't scale: too many, task-dependent, impossible for language.
8. Embeddings are **learned**: start random, train on an objective, and structure emerges ("a word is known by the company it keeps").
9. Individual embedding dimensions usually have no human meaning. Meaning is distributed, like map coordinates.
10. Every embedding is a vector, not every vector is an embedding.
11. An embedding model is a function `input → d-dimensional vector`, enabling search, clustering, recommendation and retrieval.

## Interview questions

**Q: What's an embedding?**
A learned, dense vector that places an object (word, sentence, image, user…) in a space where semantic similarity becomes geometric closeness.

**Q: Cosine similarity vs Euclidean distance?**
Euclidean measures straight-line distance and is sensitive to magnitude. Cosine measures the angle between vectors and ignores magnitude, so `[9,1]` and `[90,10]` are identical under cosine but far apart under Euclidean.

**Q: Can you say what dimension 37 of an embedding means?**
Usually not. Meaning is distributed across many dimensions, and each dimension contributes to many concepts. These are latent factors.

**Q: How do embeddings learn that "king" ≈ "queen"?**
Through a training objective (e.g. predicting surrounding words). Words that appear in similar contexts need similar predictions, so gradient descent pushes their vectors together.

**Q: Why not hand-engineer features instead?**
You can't decide how many or which ones, they change per task, and language (ambiguity, context, polysemy) makes it unmanageable.
