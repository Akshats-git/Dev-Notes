# 02 — How LLMs Actually Work: Tokens, Attention, Transformers

**Video:** [FDE Full Course #2 — How LLMs Actually Work?](https://www.youtube.com/watch?v=vZRE_jhA3Qc) (1h 35m)
**Instructor notes:** [Lecture 02 PDF](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2002)

This lecture opens the black box. It follows one sentence all the way through an LLM: text → tokens → IDs → embeddings → attention → Transformer layers → logits → probabilities → one chosen token → repeat.

---

## Contents

1. [The core idea: predict what comes next](#the-core-idea-predict-what-comes-next)
2. [Text has to become numbers](#text-has-to-become-numbers)
3. [Tokens and tokenizers](#tokens-and-tokenizers)
4. [Position and context](#position-and-context)
5. [Embeddings](#embeddings)
6. [Attention](#attention)
7. [Query, Key, Value](#query-key-value)
8. [Transformer layers](#transformer-layers)
9. [From the last vector to the next token: logits and softmax](#from-the-last-vector-to-the-next-token-logits-and-softmax)
10. [Autoregressive generation and streaming](#autoregressive-generation-and-streaming)
11. [Decoding: greedy, sampling, temperature](#decoding-greedy-sampling-temperature)
12. [The full pipeline](#the-full-pipeline)

---

## The core idea: predict what comes next

[0:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=0s)

The lecture starts with a guessing game:

| Prompt | Likely next word |
|---|---|
| I drink coffee every ___ | morning, day, evening |
| I drink coffee every morning before going to ___ | work, office, college, gym |
| I work remotely as a software engineer. Every morning I drink coffee before going to my ___ | **desk** |

With little context, everyone guesses differently. With more context, almost everyone converges on the same answer. That game is what an LLM does:

> Given everything written so far, predict what should come next.

The next piece of text depends on the **entire** context, not just the last word.

For `The capital of India is`, the model produces something like:

| Next token | Probability |
|---|---|
| Delhi | 94% |
| Punjab | 2% |
| Bangalore | 1% |
| Hyderabad | 1% |
| others | 2% |

One token is picked and appended. Now the context is `The capital of India is Delhi`, and the model predicts again. Predict → append → predict → repeat.

One correction: an LLM doesn't predict the next *word*. It predicts the next **token**.

---

## Text has to become numbers

[10:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=600s)

Neural networks only do maths: additions, multiplications, matrix operations. "Hello" has to become numbers before the model can touch it. That conversion happens **outside** the model, in a separate piece called the tokenizer. ChatGPT is a whole application around the model. It tokenizes your text before the model ever sees it.

The simplest idea is to give every word an ID:

```
cat → 1    dog → 2    apple → 3    car → 4
```

These IDs carry no meaning. `car = 4` and `cat = 1` doesn't mean a car is "bigger" than a cat. They're like roll numbers. Aditya being 237 and Rohit being 104 doesn't make Aditya "more student".

> **Token ID ≠ meaning of the token.** It just says which token we're talking about.

---

## Tokens and tokenizers

[14:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=840s)

A token can be a whole word, part of a word, punctuation, a symbol, or a chunk that includes a leading space. **One token is not always one word.**

The video demos this with OpenAI's online tokenizer, switching between GPT-3, GPT-4 and GPT-5 tokenizers:

- `The capital of India is Delhi` → 5 tokens. The space is glued to the word: `" capital"`, `" of"`, `" India"`…
- The IDs are arbitrary (e.g. `capital` → 39315, `of` → 286). No logic, just lookup.
- A 5-word sentence with "Coder" in it became 6 tokens because `" C"` and `"oder"` were split.
- `automatically` → 2 tokens in GPT-3 and GPT-4, 1 token in GPT-5.
- A single emoji → 3 tokens in GPT-3/GPT-4, 1 token in GPT-5.

Newer tokenizers tend to use fewer tokens for the same text because they have bigger, smarter vocabularies.

### Why not one token per word?

`Automatically`, `Automatic`, `Automate`, `Automation` would each need their own entry. Add names, slang, new tech words, typos, and the vocabulary explodes.

### Why not one token per character?

The vocabulary would be tiny (26 letters plus symbols), but `programming` becomes 11 tokens. Every input gets very long, which is expensive and hard for the model.

### The middle ground: subwords

```
Individual characters  ←  Tokens  →  Whole words

unbelievable → un + believ + able
```

Tokenizers look for reusable chunks of characters. Common text compresses into few tokens. Rare text breaks into more pieces. The exact split depends on the tokenizer:

```
Same sentence + different tokenizer = potentially different tokens
```

**"1 token ≈ 4 characters"** is a rough estimate for English, useful for guessing cost, but it is not the definition of a token. Providers bill by tokens, not words, for exactly this reason.

### Three different things

```
"I love programming"          ← 1. Original text
["I", "love", "programming"]  ← 2. Tokens
[51, 872, 4381]               ← 3. Token IDs
```

On the way out, the predicted ID is mapped back: `4381 → "programming"`.

---

## Position and context

[30:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=1800s)

"Dog bites man" and "Man bites dog" have the same tokens and opposite meanings. So does "Rohit teaches Aditya" vs "Aditya teaches Rohit". The model can't treat a sentence as an unordered bag `{Rohit, teaches, Aditya}`. It needs to know **what the token is and where it appears**:

```
Token representation + Position information → Position-aware representation
```

Position isn't enough either. Take "bank":

- "I deposited money at the bank" → financial institution. Relevant words: *deposited*, *money*.
- "We sat on the bank of the river" → riverside. Relevant words: *sat*, *river*.

Same token, same token ID, different meaning. A fixed representation of "bank" can't capture both. **A token's representation has to change with its context.** That's what embeddings plus attention handle.

---

## Embeddings

[34:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=2040s)

A token ID says *which* token. The model needs something it can do maths on that carries meaning. Imagine scoring each word on a few made-up features:

| Word | Royalty | Food | Living | Technology |
|---|---|---|---|---|
| King | 0.95 | 0.01 | 0.90 | 0.02 |
| Queen | 0.96 | 0.01 | 0.90 | 0.02 |
| Banana | 0.00 | 0.99 | 0.20 | 0.00 |
| Laptop | 0.00 | 0.00 | 0.00 | 0.98 |

King and Queen come out almost identical. Banana and Laptop sit far away. Each word is now a list of numbers, a **vector**, describing it in 4 dimensions.

Real models don't have neat human-readable dimensions like "Royalty". They learn hundreds or thousands of dimensions automatically during training, by seeing words used over and over in huge amounts of text. So instead of carrying around `king → 781`, the token becomes:

```
king → [0.41, -0.82, 1.34, 0.15, ...]
```

That vector is an **embedding** (also called a vector embedding). Lecture 7 goes deep on these.

```
Text → Tokens → Token IDs → Embeddings → Neural network processing
```

### Why embeddings alone aren't enough

The starting embedding for "bank" is the same in both sentences. After the model looks at the context, the internal representation should split:

```
Initial representation of "bank"
+ information from the other tokens
→ context-specific representation
```

That mixing is attention.

---

## Attention

[43:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=2580s)

> "Rohit gave Aditya his laptop because his computer was broken."

You know which "his" is which because you read the whole sentence and connect the right words. Attention gives the network a way to do the same. Each token asks:

> Which other tokens matter to me, and how much should they influence my representation?

For the "his" being processed, the model computes an **attention score** against every other token (Rohit, gave, Aditya, laptop…). It doesn't just pick the top one. It blends information from all of them, weighted by score.

Attention is **dynamic**:

- "The animal didn't cross the street because **it** was too tired." → *it* ≈ animal
- "The animal didn't cross the street because **it** was too wide." → *it* ≈ street

The model never stores `"it" → animal`. The link depends on the current sentence.

---

## Query, Key, Value

[49:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=2940s)

The three famous terms. The analogy is Google search:

- You type `best Java course` → that's your **Query**.
- Google compares it against its index entries describing pages → those are **Keys**.
- From the matching pages it returns the actual content → those are **Values**.

| Vector | Question it answers |
|---|---|
| **Query** | What information am I looking for? |
| **Key** | What kind of information do I contain? |
| **Value** | What information can I contribute? |

In a Transformer these aren't English questions. They're vectors.

### Every token makes its own Q, K and V

For `The cat sat on the mat`, every token gets three learned vectors:

```
"The" → Query, Key, Value
"cat" → Query, Key, Value
"sat" → Query, Key, Value
...
```

**Why three views of the same token?** Think of describing one person. For "who knows Java?" you look at their skills. For "who lives closest?" you look at location. For "who should teach this class?" you look at subject knowledge, communication and availability. Same person, different aspect depending on the question. Q, K and V give separate learned views for *asking*, *matching* and *contributing*.

### Self-attention, step by step

Processing the token `on` in "The cat sat on the mat":

1. Take `Query(on)`.
2. Compare it with the `Key` of every token. Comparing vectors gives a relevance score: low with *the*, medium with *cat* and *sat*, high with *mat*.
3. Turn the scores into weights `w1, w2, w3, …` that add up to 1.
4. Blend the Values using those weights:

```
Z(on) = w1·Value(the) + w2·Value(cat) + w3·Value(sat) + … + wn·Value(mat)
```

Or with the "it" example:

```
New "it" ≈ 0.60 × Value(animal)
         + 0.20 × Value(street)
         + 0.05 × Value(cross)
         + …
```

(numbers illustrative)

The result `Z` is a new vector for the token that now contains information from its context. It's called **self**-attention because tokens attend to other tokens in the *same* sequence. Every token gets its own `Z`.

---

## Transformer layers

[1:10:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=4200s)

Common misconception: **Transformer = attention.** Wrong. Attention is one important part *inside* a Transformer block. A block also has other neural-network operations (feed-forward layers where every neuron connects to the next layer's neurons, weights and biases, etc.). A Transformer is a stack of these blocks:

```
Input representations
  → Transformer block 1
  → Transformer block 2
  → …
  → Transformer block N
  → Final representations
```

GPT-style models have many such layers.

**Why many layers?** Relationships build on each other.

> "The developer couldn't deploy the application because the server ran out of memory."

A direct link is `server ↔ memory`. The abstract understanding is "deployment failed *because* the server lacked resources". That doesn't happen in one step:

```
Layer 1 → basic relationships (his ↔ Aditya, laptop ↔ broken)
Layer 2 → richer combinations
Layer 3 → more contextual relationships
…       → final contextual representation
```

The analogy is photo editing: raw → exposure → contrast → colour → masking → details → final. Each layer refines the token vectors further. At every layer the tokens make new Q, K, V vectors, attend again, and pass updated vectors on.

By the last layer, each token's vector has information about itself, its position, its surroundings, and its relationships with other tokens. Even the people who built the model can't fully say *what* each layer learned. It finds its own patterns by adjusting weights during training.

---

## From the last vector to the next token: logits and softmax

[1:15:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=4500s)

After the last layer, look at the final position (`is` in "The capital of India is"). Its vector now carries the context of the entire sentence:

```
H_final = [0.4, -1.2, 0.8, ...]
```

The question becomes: out of every token in the vocabulary (say 100,000), which comes next? The model computes one raw score per vocabulary token:

```
Delhi   → 12.8
Mumbai  →  8.1
Kolkata →  5.2
Punjab  →  3.4
banana  → -4.1
...
```

These raw scores are **logits**. They aren't probabilities yet.

**Softmax** turns logits into a probability distribution that sums to 100%:

| Token | Probability |
|---|---|
| Delhi | 92% |
| Kolkata | 2% |
| Punjab | 2% |
| Mumbai | 1% |
| Chennai | 1% |
| others | 2% |

The model isn't "thinking *write an essay*". At generation level it only ever does:

```
Given context → predict next-token distribution
```

Complex answers come from repeating that many times, plus what the model learned in training.

---

## Autoregressive generation and streaming

`The capital of India is` → pick `Delhi`. Run again on `The capital of India is Delhi` → `"."` 75%, `","` 8%, `"and"` 5%… → pick `"."`. Run again. And so on.

```
Prompt → predict token 1 → append → predict token 2 → append → …
```

This is **autoregressive generation**:
- **Auto** — the model feeds its own previous output back in as input.
- **Regressive** — each prediction depends on what came before.

This is why ChatGPT can **stream**: `Java is` → `Java is a` → `Java is a programming` → … Keep two ideas separate:

1. Generation *happens* one token at a time (a property of the model).
2. The interface *can stream* those tokens as they arrive (a choice the app makes). Lecture 6 builds this.

---

## Decoding: greedy, sampling, temperature

[1:23:00](https://www.youtube.com/watch?v=vZRE_jhA3Qc&t=4980s)

The model gives a distribution. Something else decides which token to actually take. That's the **decoding strategy**.

```
MODEL → next-token probability distribution → DECODING STRATEGY → selected token
```

Keep these separate. Greedy, sampling and temperature don't change the model. They change how we pick from its output.

### Greedy decoding

Always pick the highest-probability token.

| Token | Prob |
|---|---|
| **Java** | **40%** ← chosen |
| Python | 35% |
| C++ | 10% |

Simple, but natural language has many valid continuations. Always taking the top token makes output predictable, repetitive and bland. Even if Python is at 39% and Java at 40%, Python never gets picked.

### Sampling

Treat the probabilities as a weighted lottery. For Java 40 / Python 35 / C++ 15 / Rust 10, hand out 100 tickets in those amounts and draw one. Java is most likely, but Python, C++ or Rust can still come up.

> Higher probability = more likely to be chosen, not guaranteed.

This is why the same prompt can give different answers. The video's example: "Hi, how are you?" → "Hi, I am doing well" one time and "Hi, I am good" another.

### Temperature

Temperature reshapes the distribution *before* sampling:

```
adjusted logit = original logit / temperature
```

then softmax on the adjusted values. Using logits Python 5, Java 4, JavaScript 3, C++ 2, Rust 1:

| Temperature | Adjusted logits | Effect |
|---|---|---|
| **1** | 5, 4, 3, 2, 1 | No change. Baseline. |
| **0.5** (low) | 10, 8, 6, 4, 2 | Gaps get bigger. Top token dominates even more. **Sharper.** |
| **2** (high) | 2.5, 2, 1.5, 1, 0.5 | Gaps shrink. Lower-ranked tokens get more chance. **Flatter.** |

- **Low temperature** → predictable, consistent, less varied. Good for factual answers, extraction, code.
- **High temperature** → more variety and unexpected wording, but also more mistakes, irrelevant or nonsensical output.

Most chat products keep temperature a bit above the minimum so answers feel less robotic.

---

## The full pipeline

```
User text
  → Tokenizer → Tokens → Token IDs
  → Embeddings + Position information
  → Transformer layers (self-attention + more, repeated N times)
  → Contextual representations → Final representation
  → Vocabulary scores (logits)
  → Softmax → Next-token probabilities
  → Decoding / sampling → Selected token
  → Append to context → Repeat
```

> **In one line:** An LLM takes the existing sequence of tokens, builds context-aware mathematical representations through Transformer layers, predicts a probability distribution over possible next tokens, selects one, appends it to the sequence, and repeats.

## Summary

1. Text becomes **tokens**. Models process tokens, not words or sentences.
2. **Token IDs** identify tokens. **Embeddings** give them learned meaning.
3. **Position** matters ("dog bites man" ≠ "man bites dog").
4. **Context** changes meaning ("bank").
5. **Attention** lets each token gather information from relevant tokens via Query/Key/Value.
6. **Transformer layers** refine representations again and again.
7. The final vector is turned into **logits**, one score per vocabulary token.
8. **Softmax** turns logits into probabilities.
9. A **decoding strategy** (greedy / sampling / temperature) picks one token.
10. The token is appended and the loop repeats. That's **autoregressive generation**.

## Interview questions

**Q: Is a token the same as a word?**
No. A token can be a word, part of a word, punctuation, or a chunk with a leading space. "1 token ≈ 4 characters" is a rough English estimate, not a definition. Different tokenizers split the same text differently.

**Q: Why use subword tokens instead of words or characters?**
Whole words make the vocabulary huge and can't handle new words. Characters make sequences very long. Subwords balance vocabulary size and sequence length.

**Q: What are Q, K and V?**
Three learned vectors each token produces. Query = what I'm looking for, Key = what I contain, Value = what I contribute. A token's new representation is a weighted sum of all Values, with weights from comparing its Query with every Key.

**Q: Is a Transformer just attention?**
No. Attention is one component inside each Transformer block. The model stacks many blocks to build progressively richer representations.

**Q: What are logits?**
Raw, unnormalised scores for every vocabulary token at the output. Softmax turns them into probabilities.

**Q: What does temperature do?**
It divides logits before softmax. Below 1 sharpens the distribution (more deterministic). Above 1 flattens it (more random). It's part of decoding and doesn't change the model.

**Q: Why does the same prompt give different answers?**
Sampling. The model's distribution is the same, but the token is drawn by weighted chance instead of always taking the top one.
