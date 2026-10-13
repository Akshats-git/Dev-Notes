# 13 — Full-Text Search with Elasticsearch

**Video:** [#15 Full text search using Elasticsearch for blazingly fast search](https://www.youtube.com/watch?v=7_sovzAhRSM) (32m)

Why `LIKE '%term%'` stops working at scale, the **inverted index** idea behind full-text search, relevance scoring (BM25), typo tolerance, when to pick Elasticsearch vs Postgres full-text search, and a live benchmark: Postgres vs Elasticsearch on 50,000 reviews.

The speaker's stance: knowing Elasticsearch is a nice tool to have, not something you must master. Mastering **databases** matters far more.

---

## Contents

1. [The problem](#the-problem)
2. [The librarian analogy](#the-librarian-analogy)
3. [The inverted index](#the-inverted-index)
4. [Relevance scoring](#relevance-scoring)
5. [Typo tolerance and type-ahead](#typo-tolerance-and-type-ahead)
6. [Elasticsearch vs Postgres full-text search](#elasticsearch-vs-postgres-full-text-search)
7. [The benchmark demo](#the-benchmark-demo)
8. [Summary](#summary)
9. [Interview questions](#interview-questions)

---

## The problem

[0:00](https://www.youtube.com/watch?v=7_sovzAhRSM&t=0s)

It's 2005. You have 5,000 products and write a search endpoint:

```sql
SELECT * FROM products
WHERE name ILIKE '%laptop%' OR description ILIKE '%laptop%';
```

Fast, simple, fine. Then the company grows to **millions of products**. The same query goes from ~50 ms to **~30 seconds**. Now everyone wants more:

1. **Faster** search.
2. **Relevant** results. Searching "laptop" should show a MacBook Pro before a laptop bag.
3. **Typo tolerance**. Customers in a hurry type "laptpo"; still return laptops.

That's why search engines like Elasticsearch exist.

---

## The librarian analogy

[3:35](https://www.youtube.com/watch?v=7_sovzAhRSM&t=215s)

Your relational database is a **librarian** who knows where every book is, but has one fatal flaw: to find "machine learning" they must **check every book on every shelf, one by one**.

1. **Slow.** The work grows with the size of the library.
2. **No concept of relevance.** A book titled *Introduction to Machine Learning* and a book that mentions the phrase once on its last page both "match". They can come back in any order.

`LIKE '%term%'` is exactly this: scan every row, examine every text field, match character by character. It's thorough but painfully slow, and it returns results in no meaningful order.

---

## The inverted index

[7:12](https://www.youtube.com/watch?v=7_sovzAhRSM&t=432s)

By 2005 Google was crawling billions of pages, Amazon cataloguing millions of products, LinkedIn indexing millions of profiles. Search needed milliseconds. The answer came from **decades of information-retrieval research** (since the 1960s):

> **Flip the problem.** Instead of scanning documents to find a term, **build an index from terms to documents** when the documents arrive.

That's the **inverted index**:

| Term | Where it appears |
|---|---|
| machine | *Introduction to Machine Learning* (pages 1, 15, 23) · *The Machine Age* (5, 89) · *Coffee Machine Manual* (1) |
| learning | *Introduction to Machine Learning* (1, 16, 24) · *Learning to Cook* (2 places) · *Deep Learning Fundamentals* (3 places) |

Now to find "machine learning", you look up the terms and get the matching documents directly. No scanning.

This is the technology (**Apache Lucene**) behind Elasticsearch and many other tools. Postgres also supports full-text search. You don't need to understand the internals unless you're building such a tool. Know when to reach for it.

---

## Relevance scoring

[13:22](https://www.youtube.com/watch?v=7_sovzAhRSM&t=802s)

Beyond speed, the index lets the engine **rank** results. In the book example:

- *Introduction to Machine Learning*: "machine" is in the **title** and appears often → ranks first.
- *The Machine Age*: in the title, less often → second.
- Others: weaker → lower.

Elasticsearch scores with the **BM25** algorithm. Factors:

| Factor | Meaning |
|---|---|
| **Term frequency (TF)** | How often the term appears in *this* document |
| **Document frequency (IDF side)** | How common the term is across *all* documents. Rare terms are more informative |
| **Document length** | Short vs long documents |
| **Field boosting** | A match in `title` counts more than in `description`, which counts more than in `content`. **You can customise the weights** in your query |

A single record is a **document** (a JSON object, like in MongoDB). Queries use a **JSON-based query DSL**.

---

## Typo tolerance and type-ahead

[19:01](https://www.youtube.com/watch?v=7_sovzAhRSM&t=1141s)

Full-text engines can handle **typos** (fuzzy matching) and power **type-ahead** (autocomplete) experiences like Google's or Amazon's search box: typing "what is **treading** today" still resolves to "what is **trending** today".

---

## Elasticsearch vs Postgres full-text search

[20:33](https://www.youtube.com/watch?v=7_sovzAhRSM&t=1233s)

You have two options for a search feature:

| Option | When |
|---|---|
| **Postgres full-text search** | You already use Postgres and your needs are moderate. One less system to run |
| **Elasticsearch** | Heavy search needs, or your company **already runs it**. |

Elasticsearch is also the "E" in the **ELK stack** (**E**lasticsearch + **L**ogstash + **K**ibana), the well-known stack for **log management**. Being fast at searching logs and deriving statistics is its other big use. If your company already has it for logs, using it for product search is natural.

---

## The benchmark demo

[22:05](https://www.youtube.com/watch?v=7_sovzAhRSM&t=1325s)

Setup (kept fair: both in the **same region**, US-West):

- **Postgres** on Neon (serverless Postgres): `reviews(id, review text, sentiment)`.
- **Elasticsearch** on Elastic Cloud: an index `reviews` with `review` mapped as **`text`** (analysed, full-text) and `sentiment` as **`keyword`** (exact match).
- A CSV of **50,000 reviews** loaded into both. Postgres inserts in batches of 1,000. Elasticsearch uses a bulk insert.
- A search endpoint runs both queries and **streams** each result as soon as it's ready, so one doesn't delay the other. Both use equivalent criteria (lower-cased term, anything before or after).

Postgres query used: `SELECT ... FROM reviews WHERE review ILIKE '%term%'`.

| Search | Elasticsearch | Postgres `ILIKE` |
|---|---|---|
| "laptop" | ~1 s | ~3 s |
| a common word, ~8,000 matching rows | **~500 ms** | **~7.5 s** |

Same result count, but a large difference in time, and that's with only 50,000 rows.

### Field mapping note

| Elasticsearch type | Behaviour |
|---|---|
| `text` | Analysed and tokenised for full-text search |
| `keyword` | Exact-value matching (filters, sorting, aggregations) |

---

## Summary

1. `LIKE '%term%'` scans every row. It's slow at scale and has no concept of relevance.
2. An **inverted index** maps **term → documents (and positions)**, so searching is a lookup, not a scan.
3. Elasticsearch (built on **Apache Lucene**) returns **ranked** results via **BM25** (term frequency, document frequency, length, **field boosting**).
4. It also supports typo tolerance, type-ahead and aggregations.
5. Use **Postgres full-text search** for simpler cases or **Elasticsearch** for heavy search needs or if you already run ELK.
6. In the demo: 50k rows, ~500 ms vs ~7.5 s.
7. A document is JSON. Fields are `text` (analysed) or `keyword` (exact).

## Interview questions

**Q: Why is `LIKE '%x%'` slow?**
A leading wildcard can't use a normal index, so the database scans every row and compares text character by character. It also returns no relevance ordering.

**Q: What is an inverted index?**
A data structure mapping each term to the documents (and positions) where it occurs. Search becomes a lookup instead of a scan.

**Q: What does Elasticsearch add beyond speed?**
Relevance ranking (BM25), field boosting, fuzzy matching / typo tolerance, type-ahead, aggregations.

**Q: What does BM25 consider?**
Term frequency in the document, how common the term is across all documents, and document length. Field boosts are applied on top.

**Q: `text` vs `keyword` fields in Elasticsearch?**
`text` is analysed for full-text search. `keyword` is stored as a single exact value for exact matches, filtering and sorting.

**Q: Postgres full-text search or Elasticsearch?**
Postgres FTS if your needs are modest and you want fewer moving parts. Elasticsearch for heavy or advanced search, or if you already run it (e.g. in an ELK stack).

**Q: What is the ELK stack?**
Elasticsearch, Logstash, Kibana: a popular stack for collecting, searching and visualising logs.

**Q: What's the underlying library?**
Apache Lucene.
