# 01 — What an FDE Does, and How We Got to LLMs

**Video:** [FDE Full Course #1 — Forward Deployed Engineering Full Course](https://www.youtube.com/watch?v=kBM5UXRbo3U) (1h 21m)
**Instructor notes:** [Lecture 01 PDF](https://github.com/adityatandon15/Forward-Deployed-Engineer-Full-Course/tree/main/Lecture%2001)

The lecture has two halves. The first uses one running example (an e-commerce support bot) to show what a Forward Deployed Engineer actually does. The second is a history of AI, from if/else rules to GPT, told as a chain of "this approach broke, so people tried the next thing".

---

## Contents

1. [The running example: "build us an AI chatbot"](#the-running-example-build-us-an-ai-chatbot)
2. [Three attempts at a solution](#three-attempts-at-a-solution)
3. [What a Forward Deployed Engineer is](#what-a-forward-deployed-engineer-is)
4. [Where AI started: Turing and Dartmouth](#where-ai-started-turing-and-dartmouth)
5. [Rule-based AI and ELIZA](#rule-based-ai-and-eliza)
6. [Machine learning](#machine-learning)
7. [Neural networks and deep learning](#neural-networks-and-deep-learning)
8. [Why language is hard](#why-language-is-hard)
9. [Language models: n-grams to RNNs](#language-models-n-grams-to-rnns)
10. [Attention and Transformers](#attention-and-transformers)
11. [GPT](#gpt)
12. [Key takeaways](#key-takeaways)

---

## The running example: "build us an AI chatbot"

[4:43](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=283s)

A large e-commerce company gets around 50,000 support requests a day:

- "Where is my order?"
- "I received the wrong product."
- "Can I return an item after 15 days?"
- "My payment was deducted but the order was not created."
- "The delivery agent marked it delivered but I never got it."

Today a large team of human support executives answers these. That means salaries, infrastructure, waiting queues, and executives spending their day on the same repetitive questions while important cases get buried.

The company comes to you and says: **"We want an AI chatbot that can answer customer questions."**

That sounds like a requirement but it tells an engineer almost nothing. It is a *proposed solution*. The company doesn't actually care about having a chatbot. It cares about:

- Customers waiting too long
- Executives answering the same questions repeatedly
- Different executives giving inconsistent answers
- Support cost growing with the company
- Important cases getting buried under routine ones
- No support outside working hours

So the real requirement is closer to:

> Reduce the time required to resolve customer problems without reducing accuracy or customer trust.

A chatbot is one possible way to get there. Finding the real requirement behind the request is the first job of an FDE.

---

## Three attempts at a solution

### Attempt 1 — Rule-based bot

```
IF message contains "order status"  THEN show order tracking page
IF message contains "refund"        THEN show refund policy
IF message contains "cancel"        THEN show cancellation instructions
```

This works when input is predictable. People don't talk in fixed commands though. All of these mean the same thing:

- "Where is my order?"
- "It has been seven days and my package has not arrived."
- "Bhai mera parcel abhi tak nahi aaya."
- "The expected delivery date has passed. Could you check?"

Adding more rules doesn't save you. You run into spelling mistakes, Hinglish, multiple languages, context from earlier messages, two problems in one message, and rules that conflict. **Human language has too many variations to describe with handwritten rules.** A bot that keeps replying with the wrong canned answer also breaks customer trust fast.

### Attempt 2 — ML classification

Instead of writing rules, train a model on labelled examples:

| Customer message | Category |
|---|---|
| Where is my package? | Order Status |
| Cancel my purchase | Cancellation |
| When will I get my money back? | Refund |
| The product arrived broken | Damaged Product |

```
Customer message → ML model → Predicted category
```

Better than keyword matching. It's the same idea as a spam filter, which learns what spam looks like from examples instead of a list of banned words.

But knowing `category = Refund` doesn't explain the refund policy, look up the order, check eligibility, ask for missing details, start the refund, handle weird cases, or escalate to a human. Classification solves one step of the workflow, not the business problem. It also doesn't *generate* anything, and we want the bot to reply like a person.

### Attempt 3 — A general-purpose LLM

Plug in ChatGPT, Claude, Gemini, Grok, DeepSeek, whichever. An LLM can understand the request, summarise it, detect intent, pull out an order ID, translate, write a reply, ask follow-up questions and decide which tool might help.

Then this happens:

```
Customer: Can I return a laptop after 20 days?
LLM:      Yes, laptops can be returned within 30 days of delivery.
```

It sounds professional and confident. The company's real policy is 7 days. The answer is wrong.

> Generating a convincing answer and generating a correct business answer are different problems.

LLMs always give you *some* answer, whether or not it's right. A general model knows what a "refund" is, but it does not know:

- Your current refund policy, or recent changes to it
- This customer's orders, payments, delivery status
- Inventory, internal procedures
- Who the customer is and what they're allowed to do

So the application has to hand the model that information:

```
Customer question
+ Relevant company policy
+ Current order details
+ Previous conversation
        ↓
       LLM
        ↓
 Grounded response
```

### Generating text is not the same as doing something

"Cancel my order and refund me." The LLM can happily write *"Your order has been cancelled."* Nothing was cancelled. A real system looks like:

```
Customer request
  → LLM works out the required action
  → Application validates the request
  → Cancellation API is called
  → Database is updated
  → Confirmed result returned to the user
```

And now a pile of engineering questions show up:

- Is this user allowed to cancel this order? Has it shipped already?
- Should the LLM call the cancellation service directly?
- Should big refunds need human approval?
- What if the API fails, or the model extracts the wrong order ID?
- How do we audit and test this? What does each conversation cost? What latency is OK?
- When do we hand off to a human?

"Build an AI chatbot" has turned into:

> Build a secure and reliable system that combines an AI model, company knowledge, customer data, existing APIs, validation rules and human oversight to resolve customer-support requests.

The gap between those two sentences is where Forward Deployed Engineering lives.

---

## What a Forward Deployed Engineer is

[~21:00](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=1260s)

**"Deployed"** here does not mean pushing code to a server. It means *the engineer* is placed close to where the real problem is: with customers, clients, business departments, operations teams and their existing engineering teams.

**"Forward"** is about position. A normal product team builds one general platform for many customers:

```
Product engineering team → General platform → Many customers
```

An FDE sits right next to one customer's problem:

```
Customer's real problem
        ↕
Forward Deployed Engineer
        ↕
Product + Data + APIs + Engineering teams
```

> **Definition:** A Forward Deployed Engineer is a software engineer who works closely with customers or operational teams to understand important problems, build technical solutions around their real systems, and take those solutions into production.

In a typical company, a product person writes a ticket that spells out exactly what to build, and the developer implements the ticket. An FDE often has no ticket. The customer may not know what technical solution they need. **Discovering the right solution is part of the job.**

The skills it combines: software engineering, product thinking, system design, integrations, rapid experimentation, production delivery, evaluation and troubleshooting.

### Request vs requirement

```
"We need an AI chatbot."   ← a request (really a proposed solution)
        ↓
Understand the business problem
        ↓
Understand the existing workflow
        ↓
Identify constraints
        ↓
Design the technical solution
        ↓
Integrate with existing systems
        ↓
Test and evaluate
        ↓
Production
```

This is the mindset for the whole series. The goal is to take a vague ask like "build an AI assistant for our support team" and turn it into a real system built on GenAI. To do that you first need to understand the technology, which is the rest of this lecture.

---

## Where AI started: Turing and Dartmouth

[29:26](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=1766s)

Computers were built to compute. `20 + 30`, instruction "add", output `50`. Everything we built on top (variables, conditions, loops, functions, recursion, data structures, algorithms) is a way for humans to write precise instructions. Traditional programs work great when:

- The input is clear
- The operation can be precisely defined
- The expected result is known
- Every step can be written as an instruction

Human intelligence doesn't fit that. We understand language, recognise things, learn from experience, decide with incomplete information, and create stories, music and code. The deeper question is: **can intelligence itself be described as a computation?**

### The Turing Test (1950)

Alan Turing's paper *Computing Machinery and Intelligence* opened with "Can machines think?". The trouble is that nobody can define "thinking" precisely. Even with other humans, we only observe behaviour and infer intelligence.

So Turing swapped the question for something observable: **can a machine communicate so convincingly that a human evaluator can't reliably tell it apart from a human?**

```
            Human evaluator
                  │
        ┌─────────┴─────────┐
   Participant A       Participant B
   (one is a human, one is a machine — talking only through text)
```

If the evaluator can't tell which is which, the machine counts as intelligent for practical purposes, even if it's only mimicking intelligence. The test doesn't tell you how to *build* AI. It moves the debate from philosophy to observable behaviour.

The catch, which matters a lot for LLMs: **sounding intelligent does not mean being correct.**

### Dartmouth (1956)

John McCarthy and team proposed a workshop at Dartmouth College (proposal 1955, held 1956) and coined the term **Artificial Intelligence**. The idea: if aspects of intelligence can be described precisely enough, machines can be made to simulate them. This kicked off AI as its own research field.

The dream is old. What changed over the decades was better algorithms, far more data, and far more compute.

### AI ≠ LLMs

AI is a broad field: chess engines, medical diagnosis, face recognition, recommendation engines, self-driving, speech recognition, language models, image generators. An LLM is one kind of AI system. Generative AI is one area inside AI.

---

## Rule-based AI and ELIZA

[33:39](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=2019s)

The first big idea: if intelligent behaviour follows rules, humans can just write the rules.

```
IF age >= 18 THEN eligible ELSE not eligible

IF device does not turn on AND battery = 0
THEN recommend charging the device
```

```
Human expert writes rules → Computer applies rules → Conclusion / action
```

This is **symbolic AI**. A rule-based system has three parts:

1. **Facts** — the current situation. *Customer is premium. Order delivered 3 days ago. Product is returnable. Product is damaged.*
2. **Rules** — what should happen. *IF damaged AND delivered within 7 days AND returnable THEN eligible for replacement.*
3. **Inference** — combining facts with rules to reach a result.

The computer isn't inventing the policy. It's applying human logic.

### The real problem: getting facts out of language

The business rule is easy:

```
IF daysSinceDelivery <= 7 THEN returnEligible = true
```

But no customer sends `{"daysSinceDelivery": 4}`. They write *"I got this sometime last week, can I send it back?"* Now the system needs to know what "this" is, what date "last week" means, which order, refund or replacement, is it returnable. **The rule is easy. Turning human language into the facts the rule needs is the hard part.**

### Two early directions

- **Chatbots** — can a machine *talk* like a person?
- **Expert systems** — can a machine *decide* like a specialist?

### ELIZA

An early chatbot that felt surprisingly human:

```
User:  I am feeling unhappy.
ELIZA: Why are you feeling unhappy?
User:  My friends do not understand me.
ELIZA: Why do you think your friends do not understand you?
User:  They never listen to me.
ELIZA: Can you tell me more about them?
```

Underneath it was templates:

```
IF user says "I am <something>"   RESPOND "Why are you <something>?"
IF user mentions "mother"         RESPOND "Tell me more about your family."
```

**The lesson:** a system can produce language that *feels* meaningful without understanding anything. People naturally read intention into fluent language. Modern LLMs are far more than templates, but the lesson still holds:

> Language fluency should not be confused with factual reliability.

A fluent system makes users assume it knows what's true, verified its answer, remembers things it was never told, or performed an action it only described. In real applications those assumptions are dangerous.

### Why rules don't scale

Take just one of ten support intents, Order Tracking. Customers say "Where is my order?", "Track my package", "Delivery kab hogi?", "Order abhi tak nahi mila", "Your app says dispatched but nothing arrived". Multiply by typos, abbreviations, sarcasm, languages, missing info, multi-issue messages, conversation history, regional and policy differences. The combinations explode.

Which led to the next question: instead of writing every rule, can the machine learn the patterns from examples?

---

## Machine learning

[42:42](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=2562s)

Try writing rules for "dog": four legs, fur, two ears, a tail. That also describes cats and wolves. Add more rules and you still can't cover both a Chihuahua and a Great Dane. A child learns "dog" from examples, not a definition. ML works the same way: **give examples and let the system learn the mapping from input to output.**

```
Traditional programming:   Input + Human-written rules → Output

ML training:               Training inputs + Correct outputs → Learning algorithm → Model

Using the model:           New input + Trained model → Predicted output
```

In ML, humans provide data, examples, an objective, a model architecture and a training process. The model adjusts its own internal parameters.

- **Supervised learning** — the data is labelled (email → spam / not spam).
- **Unsupervised learning** — no labels. Given a million uncategorised support messages, the system might find clusters by itself: delivery issues, payment issues, quality complaints.

**ML does not learn truth.** It learns patterns from the data it gets. If the data is wrong, incomplete, biased, outdated or unrepresentative, the model learns that too.

### Model, training, inference

A **model** is a computational system that maps an input to an output.

```
Email          → Spam model     → Spam probability
Photo          → Image model    → Objects detected
Previous text  → Language model → Probabilities for the next token
```

**Training:** the model sees many examples, produces outputs, gets scored against an objective, adjusts its parameters, and repeats. Large models have billions of parameters and training costs enormous compute.

**Inference:** using the trained model on new input. When your app calls an LLM from Python, JS or Spring AI, that's inference.

> **Does the model learn from my prompt?** No. A prompt is temporary input that shapes the current response. A normal inference call does not change the model's trained parameters. Putting information in a prompt and training the model are two different things.

---

## Neural networks and deep learning

[50:22](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=3022s)

### The feature engineering bottleneck

A computer doesn't see "eyes, nose, mouth". It sees pixel numbers. Classic ML needed humans to design **features**: edges, shapes, colour differences, distances between facial parts. For text: word frequency, keywords, sentence length, punctuation.

Problems with that:
1. It takes a lot of human effort per problem.
2. Humans miss patterns that matter.
3. Features don't transfer. Face features are useless for speech or language.

What if the useful pattern is too complicated for a human to describe? That's where neural networks come in.

### A neuron

Say we want to predict whether a support ticket is urgent:

- `x₁` = contains the word "urgent"
- `x₂` = payment failed
- `x₃` = customer contacted support multiple times
- `x₄` = customer is locked out

They shouldn't count equally. Being locked out probably matters more than the word "urgent". So each input gets a **weight** (`w₁…w₄`):

```
Inputs → Weighted combination → Activation → Output
```

That's one artificial neuron, loosely inspired by a biological one. One neuron can only learn simple relationships, so you connect many:

```
Input layer → Hidden layer → Hidden layer → Output layer
```

Training adjusts all the weights from data, so nobody has to decide by hand how much each input matters.

### Representation learning

The big win of deep learning: models learn useful internal representations themselves. "Where is my parcel?", "Track my package", "Order abhi tak nahi aaya" use different words but mean similar things, and a network can learn that. It doesn't store `ORDER_TRACKING = [these sentences]`. The knowledge is spread across many numeric parameters.

```
Artificial Intelligence
  └── Machine Learning
        └── Deep Learning
```

We don't need to become deep learning researchers. We need enough understanding to build reliable apps on top.

**CNNs** (Convolutional Neural Networks) made computer vision work: edges → shapes → textures → objects → faces. That gave us face recognition, photo tagging, object detection and self-driving perception. Images got largely solved. Language turned out to be a different beast.

---

## Why language is hard

[1:00:49](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=3649s)

Meaning isn't in individual words. It depends on context, word order, grammar, intent, shared knowledge, culture, previous conversation, tone and ambiguity.

| Problem | Example |
|---|---|
| Same word, different meaning | "bank" — money or river side? |
| Different words, same meaning | "Where is my order?" ≈ "Track my package." |
| Word order | "The dog chased the man" ≠ "The man chased the dog" |
| Negation | "received the refund" vs "did **not** receive the refund" |
| References | "Riya put the laptop on the table because **it** was unstable / heavy." Which "it"? |
| Sarcasm | "Amazing service. My order is only three weeks late." A keyword system sees "amazing" and calls it positive. |
| Productivity | People constantly say sentences no one has said before, so you can't memorise them all. |

A useful language system has to learn patterns that generalise to new sentences.

---

## Language models: n-grams to RNNs

[1:04:00](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=3840s)

A language model estimates what text is likely to come next in a context.

- "The sun rises in the ___" → *east* is very likely
- "She deposited the money in the ___" → *bank*

> The likelihood of the next piece of text depends on the context before it.

### Counting: statistical models and n-grams

Training text:

```
I like machine learning.
I like Java programming.
Students like machine learning.
```

After "like": *machine* 2 times, *Java* 1 time. So `P(machine | like) = 2/3`, `P(Java | like) = 1/3`.

An **n-gram** is a sequence of n items. Unigram: `machine`. Bigram: `machine learning`. Trigram: `I like Java`. A bigram model uses `P(next | previous word)`. A trigram model uses `P(next | previous two words)`.

More context helps a lot. "some ___" tells you nothing. "went to the bank to deposit some ___" makes *money* obvious.

Problems with counting:
- It treats "The customer requested a refund" and "The buyer asked for reimbursement" as unrelated, though humans see customer ≈ buyer, refund ≈ reimbursement.
- **Sparsity**: most word combinations are rare or never appear in the training data at all.

### RNNs (Recurrent Neural Networks)

[1:08:11](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=4091s)

An RNN reads one word at a time and keeps updating an internal state, like running notes:

```
Initial state → "The" → state 1 → "payment" → state 2 → "has" → … → "refunded" → final state
```

That makes sense because language arrives in order, and processing in order preserves word order.

**The bottleneck:** everything seen so far gets squeezed into one fixed-size state. Imagine reading a 20-page policy word by word while keeping a tiny notepad of everything important. Early details fade and long-distance links get lost.

> "The replacement policy for laptops purchased during the festival sale, which was updated after several suppliers changed their warranty terms, **does not apply** to refurbished devices."

To understand "does not apply" you need to connect it to "The replacement policy" way back at the start. This is a **long-range dependency**, and RNNs (and LSTMs) struggled with it.

---

## Attention and Transformers

[1:15:44](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=4544s)

> "Riya gave Priya the laptop because **she** needed **it** for work."

To understand "she" the model has to decide between Riya and Priya. For "it" it has to connect to "laptop". Different words need information from different places in the sentence. **Attention** lets each position look at the other positions that are relevant to it right now.

> "The animal didn't cross the street because **it** was tired."

When processing "it": *animal* → high relevance, *tired* → high, *street* → low (streets don't get tired). Nobody wrote `IF word = "it" THEN look for animal`. These patterns are **learned** during training.

Attention is also context-dependent. "bank" pays attention to *deposited, money* in one sentence and to *sat, river* in another.

**Transformers** put attention at the centre of processing a sequence. That fixed the main limits of sequential models and became the base of every modern LLM: ChatGPT, Gemini, Grok, Claude and the rest.

**Attention is not intelligence by itself.** It's a mathematical way of weighting and combining relevant information. It doesn't guarantee the model knows what's true, has current or private data, can take real actions, or makes the right call.

---

## GPT

[1:16:47](https://www.youtube.com/watch?v=kBM5UXRbo3U&t=4607s)

Modern LLMs = huge neural networks + huge training data + learned numerical representations + Transformer architecture + attention + huge compute.

**GPT = Generative Pre-trained Transformer.**

- **Generative** — it creates output by continuing a sequence. Answers, summaries, emails, code, structured data, tool-call requests. Under the hood it's **repeated next-token prediction**: predict a likely next token, append it, repeat. (Lecture 2 goes deep on this.)
- **Pre-trained** — trained once on massive data, then reused for many apps (support, coding, docs, education, extraction…). Developers don't train it. They steer it with instructions and context.
- **Transformer** — specifically a decoder-style Transformer doing autoregressive generation.

### The whole journey

| Stage | Idea | What went wrong / what it fixed |
|---|---|---|
| Rule-based / symbolic AI | Humans write the knowledge | Impossible to maintain for messy input like language |
| Machine learning | Learn patterns from examples | No more writing every rule, but needed hand-made features |
| Neural nets / deep learning | Learn representations from data | Less feature engineering |
| N-gram language models | Count which words appear together | Limited context, sparsity |
| RNNs | Process sequences while carrying state | Long sequences and distant relationships still hard |
| Attention + Transformers | Every position can pull from relevant positions | Foundation of modern LLMs |

Each generation fixed a limitation of the previous one. GenAI didn't appear out of nowhere.

### Back to the FDE

The point of all this history: **don't treat the model as magic.** The LLM is one powerful component. The real system around it looks like:

```
Customer → Application → LLM
                          ↕ Company knowledge
                          ↕ Customer data
                          ↕ Business APIs
                          ↕ Validation rules
                          ↕ Security
                          ↕ Human oversight
```

The application is what makes it useful, grounded, secure, reliable, testable, observable, and able to take real actions. Building that is what this series is about.

---

## Key takeaways

1. A customer's proposed solution is usually not the real requirement.
2. FDEs work close to real customer problems and turn business needs into production systems.
3. AI is far broader than GenAI or LLMs.
4. Rule-based systems depend on humans writing all the knowledge and logic.
5. Human language is too varied and contextual for handwritten rules.
6. ML learns patterns from examples. It doesn't learn "truth", it learns the data.
7. A model maps inputs to outputs using learned parameters.
8. Training changes parameters. Inference (what your app does) does not.
9. Deep learning learns its own internal representations instead of relying on handcrafted features.
10. Language is hard because meaning depends on context.
11. Statistical language models introduced predicting likely continuations.
12. RNNs read sequentially but struggle with long-range relationships.
13. Attention lets each part of a sequence pull information from the relevant parts.
14. Transformers made attention central and became the base of modern LLMs.
15. GPT = Generative Pre-trained Transformer, generating text by repeated next-token prediction.
16. Fluent language does not mean factually correct.
17. An LLM alone is not a production application. Real systems need data, APIs, business rules, security, validation, evaluation and often humans in the loop.

## Interview questions

**Q: What's the difference between a customer request and a requirement?**
A request is often a proposed solution ("build a chatbot"). The requirement is the underlying business outcome ("cut resolution time without hurting accuracy or trust"). An FDE's job starts by digging out the requirement.

**Q: How is an FDE different from a regular product engineer?**
A product engineer usually builds general features for many customers from a defined ticket. An FDE embeds with a specific customer, discovers what needs to be built, integrates with their real systems, and takes it to production.

**Q: Does an LLM learn from the prompts I send it?**
No. Prompts affect only the current response. The model's parameters change only during training or fine-tuning.

**Q: Why did RNNs struggle with long text?**
They compress everything seen so far into a fixed-size state, so information from far back fades. Attention lets any position directly look at any other relevant position.

**Q: What does GPT stand for?**
Generative Pre-trained Transformer.
