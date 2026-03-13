# Search Engines

> **21 questions** — 8 theory, 11 practical, 2 experience

- Inverted indexes: how they differ from B-tree indexes and why search is fast
- Search pipeline: analysis, tokenization, querying, scoring, ranking
- PostgreSQL full-text search vs Elasticsearch: when each is sufficient
- Analyzers, tokenizers, and filters at index time and query time
- Relevance scoring: TF-IDF vs BM25, custom boosting by recency/popularity/business signals
- Field types: keyword, text, nested, object — and the cost of choosing wrong
- Cluster architecture: node roles (master, data, coordinating), cluster health states, and how they affect availability
- Shards and replicas: sizing, distribution, why primary shard count is immutable, refresh interval tradeoffs and near-real-time search
- Index mapping design: denormalization for search, nested vs parent-child relationships, dynamic vs strict mapping mode
- Schema evolution with zero-downtime reindexing using aliases
- Autocomplete and search-as-you-type: edge ngrams, completion suggesters, custom analyzers for partial matching
- Query construction: bool queries (must/should/filter), multi-field search, fuzzy matching — and why filter context skips scoring
- Node.js/TypeScript integration: official client setup, bulk indexing with partial failure handling, query building patterns, connection management and retries
- Data synchronization: CDC, dual writes, scheduled reindexing

---

## Foundational

<details>
<summary>1. How do inverted indexes work and why do they make full-text search fast — what is the structural difference between an inverted index and a B-tree index (as used in PostgreSQL), how does an inverted index map terms to documents, and why does this make searching millions of documents nearly instantaneous?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What is the search pipeline from document ingestion to query results — how do analysis, tokenization, and filtering transform text at index time, how do querying and scoring work at search time, and why does the same analyzer need to be applied at both index and query time for search to work correctly?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. When is PostgreSQL full-text search sufficient vs when do you need Elasticsearch — what does PostgreSQL's tsvector/tsquery give you, what does Elasticsearch add (analyzers, relevance tuning, fuzzy matching, aggregations, horizontal scaling), and what are the operational costs of running Elasticsearch?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. How does relevance scoring work — what are TF-IDF and BM25, why did Elasticsearch switch from TF-IDF to BM25, how do you boost results by recency, popularity, or business signals (function_score), and when does the default scoring produce poor results that need custom tuning?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What Elasticsearch field types exist and why does choosing wrong hurt — what's the difference between keyword and text fields, what are nested vs object types (and why object type flattens arrays incorrectly), and what's the performance and storage cost of using the wrong field type?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How does Elasticsearch cluster architecture work — what are the roles of master-eligible, data, and coordinating nodes, why does Elasticsearch separate these roles instead of having identical nodes, and what do the cluster health states (green, yellow, red) mean for availability and data safety?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do shards and replicas work in Elasticsearch — how do you size shards (too many vs too few), why is the primary shard count immutable after index creation, how does the refresh interval affect near-real-time search latency, and what's the tradeoff between refresh frequency and indexing throughput?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do you synchronize data between your primary database and Elasticsearch — what are the approaches (CDC with Debezium, dual writes, scheduled reindexing), what are the consistency tradeoffs of each, and why is dual-write the most dangerous approach?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Index Design & Analysis

<details>
<summary>9. Design an index mapping for a product catalog (name, description, category, price, brand, attributes) — show the mapping with appropriate field types (text with analyzers for search, keyword for filtering, nested for attributes), explain the difference between nested and parent-child (join field) relationships and when each is appropriate, set the mapping to strict mode and explain why dynamic mapping is dangerous in production, and demonstrate the tradeoff between indexing flexibility and storage/performance</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Configure custom analyzers, tokenizers, and filters for a search use case — show an analyzer that handles English text (standard tokenizer, lowercase filter, stemmer, stop words), a language-aware analyzer, and demonstrate the difference between index-time and query-time analysis using the _analyze API</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Implement zero-downtime reindexing using aliases — show how to create a new index with an updated mapping, reindex data from the old index, atomically swap the alias to point to the new index, and clean up the old index. Explain when you need this (mapping changes, analyzer updates, shard count changes)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Implement autocomplete/search-as-you-type using edge ngrams — show the custom analyzer with edge ngram tokenizer at index time, the search query that uses a different analyzer at query time, compare with the completion suggester approach, and explain the tradeoffs in index size and relevance quality</summary>

<!-- Answer will be added later -->

</details>

## Practical — Query Construction & Features

<details>
<summary>13. Build a search query using bool queries (must, should, filter) — show a product search that combines full-text search on name/description (must), boosts by rating (should), filters by category and price range (filter), and explain why filter context skips scoring and improves performance</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Implement multi-field search with boosting and fuzzy matching — show a query that searches across title (boosted 3x) and description (boosted 1x), add fuzziness for typo tolerance, and demonstrate how the boosting affects result ordering. Explain when fuzzy matching hurts relevance more than it helps</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Implement custom relevance scoring using function_score — show a query that combines text relevance with recency boost (newer products rank higher) and popularity boost (more sales rank higher), explain how the score functions combine, and demonstrate tuning the weights to get the desired ranking</summary>

<!-- Answer will be added later -->

</details>

## Practical — Integration & Operations

<details>
<summary>16. Implement bulk indexing in Node.js/TypeScript using the official Elasticsearch client — show how to batch documents, handle partial failures in the bulk response (some documents succeed while others fail), and implement retry logic for transient errors. What breaks if you don't check the bulk response item-by-item?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Build a search endpoint in Node.js/TypeScript that translates HTTP query parameters into Elasticsearch queries — show the client setup with connection management and retries, demonstrate how you handle common failures (index not found, timeout, version conflict), and explain what error responses the caller should receive for each.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Set up data synchronization between PostgreSQL and Elasticsearch using CDC — show the Debezium connector configuration for capturing PostgreSQL changes, the Kafka consumer that transforms and indexes documents into Elasticsearch, and the error handling for when Elasticsearch rejects a document. Compare with the simpler scheduled reindexing approach (show the cron-based reindexing script) and explain when real-time CDC isn't worth the operational complexity.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>19. Search queries are taking 2 seconds instead of 200ms — walk through diagnosing the performance issue: use query profiling to identify expensive clauses (wildcard queries, deeply nested aggregations, script scoring), check shard distribution and cluster health, identify if the refresh interval is too aggressive, and apply the fix</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>20. Tell me about a time you implemented search for a product or feature — what technology did you choose (PostgreSQL FTS, Elasticsearch, Algolia), how did you design the index, and what challenges did you face with relevance or performance?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>21. Tell me about a time you dealt with data synchronization between your primary database and search index — what approach did you use (CDC, dual write, scheduled sync), what consistency issues did you encounter, and how did you resolve them?</summary>

<!-- Answer framework will be added later -->

</details>
