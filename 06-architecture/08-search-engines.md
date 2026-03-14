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

A **B-tree index** maps row values to row locations. It's optimized for exact lookups and range scans on structured data (e.g., `WHERE id = 5` or `WHERE price > 100`). The tree structure gives O(log n) lookups, but it indexes entire column values -- it can't efficiently find "which rows contain the word 'shipping' somewhere in a text field?"

An **inverted index** flips the relationship: instead of mapping documents to their words, it maps every unique term to a list of documents (called a **postings list**) that contain that term.

**How it works:**

1. At index time, each document's text is analyzed (lowercased, stemmed, stop words removed) into individual terms.
2. Each term gets an entry in a sorted dictionary.
3. Each dictionary entry points to a postings list: an ordered list of document IDs where that term appears, often with position and frequency data.

```
Term          → Postings List (doc IDs)
"elasticsearch" → [1, 4, 7, 15, 203]
"index"         → [1, 2, 4, 7, 8, 15]
"search"        → [1, 3, 4, 7, 12, 15, 203]
```

**Why it's fast:** Searching for "elasticsearch search" means:
1. Look up "elasticsearch" in the dictionary -- O(log n) where n is unique terms (not documents).
2. Look up "search" in the dictionary -- same cost.
3. Intersect the two postings lists -- linear merge of sorted lists.

The key insight: the cost scales with the number of unique terms and matching documents, not total documents. Searching 100 million documents is nearly as fast as searching 1 million if the term dictionary and postings lists fit the same profile.

**Structural comparison:**

| Aspect | B-tree | Inverted index |
|--------|--------|----------------|
| Maps | Values to row locations | Terms to document lists |
| Optimized for | Exact match, range scans | Full-text search, term lookup |
| Granularity | Whole column values | Individual terms within text |
| Search cost | O(log n) per row count | O(log t) + postings merge (t = unique terms) |
| Multi-word queries | Requires scanning | Postings list intersection/union |

Elasticsearch stores inverted indexes in immutable **segments** (Lucene segments). New documents go into new segments, and segments are periodically merged. This immutability enables efficient caching and concurrent reads.

</details>

<details>
<summary>2. What is the search pipeline from document ingestion to query results — how do analysis, tokenization, and filtering transform text at index time, how do querying and scoring work at search time, and why does the same analyzer need to be applied at both index and query time for search to work correctly?</summary>

The search pipeline has two phases: **index time** (document ingestion) and **query time** (search execution).

**Index-time pipeline:**

1. **Character filters** -- Transform raw text before tokenization. Examples: strip HTML tags, replace character patterns (`&` to `and`).
2. **Tokenizer** -- Splits text into individual tokens. The standard tokenizer splits on whitespace and punctuation: `"High-performance Node.js"` becomes `["High", "performance", "Node.js"]`.
3. **Token filters** -- Transform tokens sequentially:
   - **Lowercase**: `"High"` to `"high"`
   - **Stop words**: Remove common words like "the", "is", "and"
   - **Stemming**: Reduce words to root form: `"running"` to `"run"`, `"performances"` to `"perform"`
   - **Synonyms**: Expand `"laptop"` to also index `"notebook"`

The combination of character filters + tokenizer + token filters is called an **analyzer**. The output tokens are stored in the inverted index.

**Query-time pipeline:**

1. **Query parsing** -- The search query string is parsed into a structured query (bool, match, term, etc.).
2. **Analysis** -- The query text goes through an analyzer (usually the same one used at index time) to produce query terms.
3. **Term lookup** -- Each query term is looked up in the inverted index to get matching document lists.
4. **Scoring** -- Each matching document gets a relevance score based on BM25 (term frequency, inverse document frequency, field length).
5. **Sorting and pagination** -- Results are sorted by score (or custom sort), then the requested page is returned.

**Why the same analyzer matters:**

If you index `"Running shoes"` with stemming (stored as `"run"`, `"shoe"`), but search for `"running"` without stemming, the query term `"running"` won't match the indexed term `"run"`. The analyzer must produce the same token forms at both index and query time for matches to occur.

There are intentional exceptions: edge ngrams for autocomplete use a different query-time analyzer (covered in question 12), and synonym expansion is sometimes applied only at index time or only at query time depending on the use case.

</details>

## Conceptual Depth

<details>
<summary>3. When is PostgreSQL full-text search sufficient vs when do you need Elasticsearch — what does PostgreSQL's tsvector/tsquery give you, what does Elasticsearch add (analyzers, relevance tuning, fuzzy matching, aggregations, horizontal scaling), and what are the operational costs of running Elasticsearch?</summary>

**PostgreSQL full-text search gives you:**

- **tsvector**: A processed document representation (tokenized, stemmed, stop words removed) stored as a column or generated column.
- **tsquery**: A query language with boolean operators (`&`, `|`, `!`) and prefix matching.
- **GIN indexes**: Fast lookup on tsvector columns.
- **Ranking**: `ts_rank` and `ts_rank_cd` for basic relevance scoring.
- **Language support**: Built-in dictionaries for stemming in multiple languages.

```sql
-- Add a tsvector column with GIN index
ALTER TABLE products ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', name || ' ' || description)) STORED;
CREATE INDEX idx_search ON products USING GIN(search_vector);

-- Query with ranking
SELECT name, ts_rank(search_vector, query) AS rank
FROM products, to_tsquery('english', 'wireless & headphones') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

**PostgreSQL FTS is sufficient when:**
- Search is a secondary feature, not the primary UX
- Dataset is under ~1-5 million rows
- Simple keyword matching with basic ranking is enough
- You want zero additional infrastructure
- You don't need fuzzy matching, autocomplete, or faceted search

**Elasticsearch adds:**

| Capability | PostgreSQL FTS | Elasticsearch |
|------------|---------------|---------------|
| Custom analyzers | Limited (dictionaries) | Fully configurable pipeline |
| Fuzzy matching | No | Yes (Levenshtein distance) |
| Autocomplete | Basic prefix only | Edge ngrams, completion suggesters |
| Aggregations/facets | Use SQL GROUP BY (limited) | Native, nested, pipeline aggregations |
| Relevance tuning | ts_rank weights | BM25 + function_score + boosting |
| Horizontal scaling | Read replicas only | Sharding across nodes |
| Synonyms | Custom dictionaries | Synonym filters, easy updates |
| Highlighting | ts_headline (basic) | Multiple highlighting strategies |
| Nested objects | Joins (expensive) | Nested type, parent-child |

**Operational costs of Elasticsearch:**

- **Infrastructure**: Minimum 3 nodes for production (master election). Memory-hungry -- each node needs significant heap.
- **Data synchronization**: Your source of truth is still the database. You need CDC, dual writes, or scheduled sync (see question 8).
- **Operational complexity**: Cluster management, shard rebalancing, mapping migrations, monitoring cluster health.
- **Expertise**: Mapping design, analyzer configuration, query tuning, and capacity planning require specialized knowledge.
- **Cost**: Typically 3-10x more infrastructure cost than PostgreSQL alone, depending on data volume and query load.

**Decision framework**: Start with PostgreSQL FTS. Move to Elasticsearch when you need fuzzy matching, autocomplete, faceted navigation, complex relevance tuning, or your dataset outgrows what PostgreSQL can search within acceptable latency (typically 50-200ms).

</details>

<details>
<summary>4. How does relevance scoring work — what are TF-IDF and BM25, why did Elasticsearch switch from TF-IDF to BM25, how do you boost results by recency, popularity, or business signals (function_score), and when does the default scoring produce poor results that need custom tuning?</summary>

**TF-IDF (Term Frequency - Inverse Document Frequency):**

- **TF**: How often a term appears in a document. More occurrences = higher relevance.
- **IDF**: How rare a term is across all documents. Rare terms matter more than common ones.
- **Score**: TF x IDF. A document with many occurrences of a rare term scores highest.

**Problem with TF-IDF**: Term frequency grows linearly without bound. A document mentioning "elasticsearch" 100 times scores disproportionately higher than one mentioning it 10 times, even though the relevance difference is marginal.

**BM25 (Best Matching 25):**

BM25 is a refinement of TF-IDF with two key improvements:

1. **Term frequency saturation**: TF contribution has a diminishing returns curve. After a certain point, more occurrences barely increase the score. Controlled by parameter `k1` (default 1.2).
2. **Document length normalization**: Longer documents naturally contain more term occurrences. BM25 normalizes by field length so short, focused documents aren't disadvantaged. Controlled by parameter `b` (default 0.75).

Elasticsearch switched to BM25 as the default in version 5.0 because it produces more intuitive rankings out of the box, especially for varying document lengths.

**Custom scoring with function_score:**

When default BM25 isn't enough, `function_score` lets you combine text relevance with business signals:

```json
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "wireless headphones" } },
      "functions": [
        {
          "gauss": {
            "created_at": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          },
          "weight": 2
        },
        {
          "field_value_factor": {
            "field": "sales_count",
            "modifier": "log1p",
            "factor": 0.5
          },
          "weight": 1
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

This combines text relevance with a recency decay (products older than 30 days score lower) and popularity boost (more sales = higher score, with logarithmic dampening).

**When default scoring fails and needs tuning:**

- **Short vs long fields**: A product name matching "wireless headphones" should score higher than a description that mentions it once in 500 words, but BM25 might not differentiate enough. Solution: boost the name field.
- **Popularity bias needed**: Users expect popular/highly-rated items first, not just text-relevant ones. Solution: `function_score` with `field_value_factor`.
- **Recency matters**: News, job listings, or product catalogs where freshness is a relevance signal. Solution: decay functions.
- **Exact match vs partial match**: Searching "Nike Air Max" should rank the exact product above "Nike running shoes for air comfort and maximum support". Solution: phrase boosting with `match_phrase` in a `should` clause.
- **Multi-language content**: Default BM25 parameters may not suit all languages. Solution: per-field analyzer configuration and adjusted `k1`/`b`.

</details>

<details>
<summary>5. What Elasticsearch field types exist and why does choosing wrong hurt — what's the difference between keyword and text fields, what are nested vs object types (and why object type flattens arrays incorrectly), and what's the performance and storage cost of using the wrong field type?</summary>

**keyword vs text:**

| Aspect | `keyword` | `text` |
|--------|-----------|--------|
| Analysis | None -- stored as-is | Analyzed (tokenized, lowercased, stemmed) |
| Use case | Filtering, sorting, aggregations | Full-text search |
| Example | Status codes, categories, email addresses | Product descriptions, article content |
| Queries | `term`, `terms`, `range` | `match`, `match_phrase`, `multi_match` |

**Wrong choice costs:**
- Using `text` for a category field: Can't aggregate or filter exactly (analyzed tokens don't match exact values). Sorting produces unexpected results.
- Using `keyword` for a description: No full-text search capability. Must match the entire string exactly.
- Common pattern: Use `multi-field` mapping for fields that need both:

```json
{
  "brand": {
    "type": "text",
    "analyzer": "standard",
    "fields": {
      "raw": { "type": "keyword" }
    }
  }
}
```

Search on `brand`, filter/aggregate on `brand.raw`.

**object vs nested:**

The `object` type is the default for JSON objects. It **flattens** nested structures, which silently breaks array relationships.

```json
// Indexed document
{
  "product": "T-shirt",
  "variants": [
    { "color": "red", "size": "L" },
    { "color": "blue", "size": "S" }
  ]
}
```

With `object` type, Elasticsearch internally flattens to:
```json
{
  "variants.color": ["red", "blue"],
  "variants.size": ["L", "S"]
}
```

A query for `color: red AND size: S` **incorrectly matches** because the association between color and size within each array element is lost. Red exists, S exists, but there's no red-S variant.

With `nested` type, each array element is stored as a separate hidden document, preserving the relationship. The query correctly returns no match for red-S.

**Cost of nested:**
- Each nested object is a separate Lucene document. An array of 100 variants means 101 documents (1 parent + 100 nested).
- Queries are slower -- nested queries join across documents internally.
- Updates require reindexing the entire parent document and all nested docs.

**When to use which:**
- **object**: When you don't query on combinations of sub-fields, or when the array has a single element.
- **nested**: When you need to query on correlated fields within array elements.
- **parent-child (join field)**: When nested objects change frequently independent of the parent (avoids reindexing the parent). Much slower at query time. Covered in question 9.

**Other field types to know:**
- `integer`, `float`, `long`, `double` -- Numeric types for ranges and aggregations
- `date` -- Parsed dates with format specification, enables range queries and date math
- `boolean` -- For flag-style filtering
- `geo_point` -- Latitude/longitude for distance queries

**Storage/performance pitfalls:**
- Mapping explosion: Dynamic mapping on arbitrary JSON creates thousands of fields, bloating the cluster state and slowing queries. Use `strict` mapping mode in production.
- Using `text` where `keyword` suffices wastes storage (inverted index with analyzed tokens vs single term).
- Using `nested` when `object` is fine multiplies document count and memory usage unnecessarily.

</details>

<details>
<summary>6. How does Elasticsearch cluster architecture work — what are the roles of master-eligible, data, and coordinating nodes, why does Elasticsearch separate these roles instead of having identical nodes, and what do the cluster health states (green, yellow, red) mean for availability and data safety?</summary>

**Node roles:**

| Role | Responsibility | Resource needs |
|------|---------------|----------------|
| **Master-eligible** | Cluster state management: creating/deleting indexes, tracking nodes, allocating shards. One active master elected from eligible nodes. | Low CPU/memory, low disk. Stability is critical. |
| **Data** | Stores shard data, executes queries and indexing. Does the heavy lifting. | High CPU, memory, and disk I/O. |
| **Coordinating** (every node by default) | Routes requests: scatters search to relevant shards, gathers and merges results. Dedicated coordinating nodes have no other role. | Moderate memory for merging/sorting large result sets. |
| **Ingest** | Runs ingest pipelines (transform documents before indexing). | CPU-intensive for heavy transformations. |

**Why separate roles:**

In small clusters (3 nodes), every node does everything. As you scale, dedicated roles prevent resource contention:

- A heavy indexing workload on data nodes could destabilize the master if they share resources. A master node that pauses for GC might be perceived as dead, triggering unnecessary re-elections and shard reallocation cascades.
- Dedicated coordinating nodes offload the scatter-gather overhead from data nodes, improving search latency under load.
- Separating allows independent scaling: add data nodes for storage, add coordinating nodes for query throughput.

**Minimum production topology**: 3 dedicated master-eligible nodes (for quorum -- avoids split-brain), N data nodes based on data volume, and optionally dedicated coordinating nodes for high query throughput.

**Cluster health states:**

| State | Meaning | Impact |
|-------|---------|--------|
| **Green** | All primary and replica shards are allocated. | Full redundancy. No data loss risk. |
| **Yellow** | All primary shards are allocated, but some replicas are not. | Search and indexing work fine. Data exists on primaries but isn't replicated -- a node failure could cause data loss. Common with single-node clusters. |
| **Red** | Some primary shards are unallocated. | Data for those shards is unavailable. Queries against affected indexes will return partial results or fail. **This is an emergency.** |

**Common causes:**
- Yellow: Not enough nodes to place replicas (replica can't live on the same node as its primary). Adding a node usually resolves it.
- Red: Node failure with no replica, disk full (Elasticsearch watermarks block allocation), or corrupt shard data. Requires immediate investigation.

</details>

<details>
<summary>7. How do shards and replicas work in Elasticsearch — how do you size shards (too many vs too few), why is the primary shard count immutable after index creation, how does the refresh interval affect near-real-time search latency, and what's the tradeoff between refresh frequency and indexing throughput?</summary>

**Shards and replicas basics:**

An index is split into **primary shards** (horizontal partitions of data). Each primary shard can have zero or more **replica shards** (copies on different nodes). Replicas serve two purposes: fault tolerance (data survives node loss) and read throughput (searches can hit any replica).

**Why primary shard count is immutable:**

Documents are routed to shards using: `shard = hash(document_id) % number_of_primary_shards`. Changing the shard count would change which shard every document belongs to, requiring a complete reindex. This is why you choose shard count at index creation and use the reindex API with a new index if you need to change it (see question 11).

**Shard sizing guidelines:**

| Problem | Too few shards | Too many shards |
|---------|---------------|-----------------|
| Impact | Individual shards grow too large (slow queries, long recovery) | Overhead per shard (file handles, memory, cluster state). "Oversharding." |
| Symptom | Queries slow as data grows, can't parallelize across nodes | High memory usage, slow cluster state updates, many small segments |
| Guideline | Keep shards between 10-50 GB each | Aim for < 20 shards per GB of heap |

**Practical sizing approach:**
1. Estimate total data size (e.g., 500 GB).
2. Target 20-40 GB per shard: `500 / 30 = ~17 primary shards`.
3. Account for replicas: 17 primaries x 1 replica = 34 total shards. Ensure you have enough nodes to distribute them.
4. For time-series data, use index lifecycle management (ILM) with rollover instead of one massive index.

**Refresh interval and near-real-time search:**

When a document is indexed, it's written to an in-memory buffer. It becomes searchable only after a **refresh** operation flushes the buffer to a new Lucene segment. The default refresh interval is **1 second**, which is why Elasticsearch is called "near-real-time" -- there's up to a 1-second delay between indexing and searchability.

**Refresh tradeoffs:**

| Setting | Effect | Use case |
|---------|--------|----------|
| `1s` (default) | Documents searchable within 1 second. Creates many small segments that need merging. | Interactive search where users expect recent data |
| `30s` or `60s` | Fewer segments, better indexing throughput. Higher search latency for new data. | High-throughput indexing (bulk imports, logging) |
| `-1` (disabled) | No automatic refresh. Must refresh manually or on search. | Bulk loading -- refresh once after the load completes |

```json
PUT /products/_settings
{
  "index": {
    "refresh_interval": "30s"
  }
}
```

**Practical pattern for bulk loading**: Disable refresh and replicas during bulk import, then re-enable after.

Before bulk load:

```json
PUT /products/_settings
{ "index": { "refresh_interval": "-1", "number_of_replicas": 0 } }
```

Perform bulk indexing, then re-enable refresh and replicas:

```json
PUT /products/_settings
{ "index": { "refresh_interval": "1s", "number_of_replicas": 1 } }

POST /products/_refresh
```

</details>

<details>
<summary>8. How do you synchronize data between your primary database and Elasticsearch — what are the approaches (CDC with Debezium, dual writes, scheduled reindexing), what are the consistency tradeoffs of each, and why is dual-write the most dangerous approach?</summary>

**Three main approaches:**

**1. CDC (Change Data Capture) with Debezium:**

Debezium reads the database's write-ahead log (WAL in PostgreSQL) and publishes change events to Kafka. A consumer reads these events and updates Elasticsearch.

- **Consistency**: Near-real-time (seconds of delay). Changes are captured from the WAL, so nothing is missed -- even if the application crashes mid-operation.
- **Ordering**: Kafka partitioning by entity key preserves per-entity ordering.
- **Pros**: Decoupled from application code, captures all changes (including direct DB modifications), reliable.
- **Cons**: Operational complexity (Kafka + Debezium + connectors), schema changes need connector updates, WAL-level events need transformation to document shape.

**2. Scheduled reindexing:**

A cron job periodically queries the database and bulk-indexes documents into Elasticsearch.

- **Consistency**: Eventual, with lag equal to the cron interval (minutes to hours).
- **Approach**: Query for records modified since last run (`WHERE updated_at > last_run_timestamp`), or full reindex for small datasets.
- **Pros**: Simplest to implement and operate. No additional infrastructure beyond the cron job.
- **Cons**: Stale data between runs. Deletes are tricky (need soft deletes or a separate deletion pass). Full reindex becomes slow as data grows.

**3. Dual writes:**

The application writes to both the database and Elasticsearch in the same code path.

```typescript
// DANGEROUS -- don't do this
async function createProduct(product: Product) {
  await db.products.insert(product);       // Step 1
  await esClient.index({                   // Step 2
    index: 'products',
    id: product.id,
    document: product,
  });
}
```

**Why dual writes are dangerous:**

- **Partial failure**: If step 1 succeeds but step 2 fails (Elasticsearch is down, network timeout), data is in the DB but not in search. The reverse is also possible.
- **No atomicity**: There's no distributed transaction between PostgreSQL and Elasticsearch. You can't roll back one if the other fails.
- **Race conditions**: Two concurrent updates to the same document can arrive at Elasticsearch in a different order than they were committed to the database, leaving stale data in the index permanently.
- **Tight coupling**: Every code path that modifies data must remember to update both systems. A missed path means silent data drift.
- **Retry complexity**: Retrying the ES write on failure can cause duplicates or ordering issues. Not retrying causes data loss.

**Comparison:**

| Aspect | CDC (Debezium) | Scheduled reindex | Dual write |
|--------|---------------|-------------------|------------|
| Latency | Seconds | Minutes to hours | Milliseconds |
| Consistency | Strong eventual | Periodic eventual | Inconsistent under failure |
| Operational cost | High (Kafka, Debezium) | Low (cron job) | Low infra, high code risk |
| Captures all changes | Yes (WAL-level) | Yes (if using updated_at) | No (only instrumented paths) |
| Handles deletes | Yes | Needs soft deletes | Must instrument |
| Recommended | Production systems with real-time needs | Small datasets, batch-tolerant use cases | Almost never |

**Recommendation**: Start with scheduled reindexing for simplicity. Move to CDC when you need real-time search updates and can invest in the infrastructure. Avoid dual writes -- the failure modes are subtle and hard to detect.

</details>

## Practical — Index Design & Analysis

<details>
<summary>9. Design an index mapping for a product catalog (name, description, category, price, brand, attributes) — show the mapping with appropriate field types (text with analyzers for search, keyword for filtering, nested for attributes), explain the difference between nested and parent-child (join field) relationships and when each is appropriate, set the mapping to strict mode and explain why dynamic mapping is dangerous in production, and demonstrate the tradeoff between indexing flexibility and storage/performance</summary>

```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "english_stemmer", "english_stop"]
        }
      },
      "filter": {
        "english_stemmer": { "type": "stemmer", "language": "english" },
        "english_stop": { "type": "stop", "stopwords": "_english_" }
      }
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "product_analyzer",
        "fields": {
          "raw": { "type": "keyword" },
          "suggest": {
            "type": "text",
            "analyzer": "simple"
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "product_analyzer"
      },
      "category": {
        "type": "keyword"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "brand": {
        "type": "keyword"
      },
      "attributes": {
        "type": "nested",
        "properties": {
          "name": { "type": "keyword" },
          "value": { "type": "keyword" }
        }
      },
      "created_at": {
        "type": "date"
      },
      "sales_count": {
        "type": "integer"
      }
    }
  }
}
```

**Key design decisions:**

- **`name`** is multi-field: `text` for full-text search, `keyword` sub-field for sorting/aggregations, `simple` sub-field for case-insensitive autocomplete.
- **`category`** and **`brand`** are `keyword` -- you filter and aggregate on exact values, not full-text search.
- **`price`** uses `scaled_float` (stores as integer internally) -- more efficient than `float` for currency.
- **`attributes`** is `nested` because products have arrays like `[{name: "color", value: "red"}, {name: "size", value: "L"}]`. Using `object` type would flatten these and break queries like "color is red AND size is L" (as explained in question 5).

**Why `dynamic: "strict"`:**

Dynamic mapping lets Elasticsearch auto-detect field types when it sees new fields. This is dangerous in production:

- A single malformed document with an unexpected field creates a new mapping entry. If the first value is `"42"`, ES maps it as `text`. When the next document sends `42` (number), indexing fails.
- Arbitrary fields bloat cluster state and memory. Each unique field path adds to the mapping. With user-generated data, this can cause mapping explosion (thousands of fields).
- Strict mode rejects documents with unmapped fields, forcing explicit mapping changes through a controlled process.

**Nested vs parent-child (join field):**

| Aspect | Nested | Parent-child (join) |
|--------|--------|-------------------|
| Storage | Nested docs stored with parent in same shard segment | Parent and child are separate documents on same shard |
| Query speed | Fast (same Lucene segment) | Slow (requires join across documents) |
| Update independence | Updating any nested doc reindexes the entire parent | Child docs can be updated independently |
| Use case | Attributes, variants -- small arrays that change with the product | Comments, reviews -- many child docs that change frequently |
| Limit | Careful with large arrays (each element = hidden doc) | No practical array size limit, but slow queries |

**Use nested** for product attributes (small, change with the product). **Use parent-child** for something like product reviews (many, change independently, don't need to reindex the product).

</details>

<details>
<summary>10. Configure custom analyzers, tokenizers, and filters for a search use case — show an analyzer that handles English text (standard tokenizer, lowercase filter, stemmer, stop words), a language-aware analyzer, and demonstrate the difference between index-time and query-time analysis using the _analyze API</summary>

**Custom English analyzer setup:**

```json
PUT /articles
{
  "settings": {
    "analysis": {
      "analyzer": {
        "english_custom": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "english_possessive_stemmer",
            "english_stop",
            "english_stemmer"
          ]
        },
        "english_search": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "english_possessive_stemmer",
            "english_stop",
            "english_stemmer",
            "synonym_filter"
          ]
        }
      },
      "filter": {
        "english_stop": {
          "type": "stop",
          "stopwords": "_english_"
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        },
        "english_possessive_stemmer": {
          "type": "stemmer",
          "language": "possessive_english"
        },
        "synonym_filter": {
          "type": "synonym",
          "synonyms": ["laptop,notebook", "phone,mobile"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "english_custom",
        "search_analyzer": "english_search"
      },
      "body": {
        "type": "text",
        "analyzer": "english_custom"
      }
    }
  }
}
```

**How the analyzer pipeline transforms text:**

Input: `"The runner's high-performance shoes are running smoothly"`

1. **Standard tokenizer**: `["The", "runner's", "high", "performance", "shoes", "are", "running", "smoothly"]`
2. **Lowercase**: `["the", "runner's", "high", "performance", "shoes", "are", "running", "smoothly"]`
3. **Possessive stemmer**: `["the", "runner", "high", "performance", "shoes", "are", "running", "smoothly"]`
4. **Stop words**: `["runner", "high", "performance", "shoes", "running", "smoothly"]` (removes "the", "are")
5. **Stemmer**: `["runner", "high", "perform", "shoe", "run", "smooth"]`

**Using the _analyze API to inspect analysis:**

```bash
# Test index-time analysis
POST /articles/_analyze
{
  "analyzer": "english_custom",
  "text": "The runners were running quickly"
}
# Returns tokens: ["runner", "run", "quickli"]

# Test with a specific field's analyzer
POST /articles/_analyze
{
  "field": "title",
  "text": "The runners were running quickly"
}

# Test a built-in analyzer for comparison
POST /_analyze
{
  "analyzer": "standard",
  "text": "The runners were running quickly"
}
# Returns tokens: ["the", "runners", "were", "running", "quickly"]
```

**Index-time vs query-time analysis:**

You can specify different analyzers using `analyzer` (index-time) and `search_analyzer` (query-time). They're usually the same, but there are valid reasons to differ:

- **Synonyms at index time only**: Index "laptop" as both "laptop" and "notebook". At query time, searching for "laptop" matches directly without expanding, keeping queries fast. Downside: adding new synonyms requires reindexing.
- **Synonyms at query time only**: The index stores only the original terms. At query time, "laptop" expands to `"laptop" OR "notebook"`. Adding new synonyms works immediately without reindexing. Downside: queries are slightly more expensive.
- **Edge ngrams for autocomplete**: Index with edge ngram analyzer (produces `["ru", "run", "runn", "runni", "runnin", "running"]`), but search with a standard analyzer (just `"running"`) so partial input matches indexed ngrams. Covered in detail in question 12.

</details>

<details>
<summary>11. Implement zero-downtime reindexing using aliases — show how to create a new index with an updated mapping, reindex data from the old index, atomically swap the alias to point to the new index, and clean up the old index. Explain when you need this (mapping changes, analyzer updates, shard count changes)</summary>

**When you need zero-downtime reindexing:**

- Changing field types (e.g., `text` to `keyword`)
- Updating analyzer configuration (new stemmer, synonym list changes)
- Changing primary shard count (immutable after creation)
- Adding new fields that require backfilling with transformed data
- Migrating to a new mapping structure

**The alias pattern:**

Your application always queries the alias `products`, never the actual index name. This lets you swap the underlying index without any application changes.

**Step 1: Initial setup (done once)**

```bash
# Create the first versioned index
PUT /products_v1
{
  "settings": { "number_of_shards": 3 },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "category": { "type": "keyword" }
    }
  }
}

# Point the alias to it
POST /_aliases
{
  "actions": [
    { "add": { "index": "products_v1", "alias": "products" } }
  ]
}
```

**Step 2: Create the new index with updated mapping**

```bash
PUT /products_v2
{
  "settings": {
    "number_of_shards": 5,
    "analysis": {
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "english_stemmer"]
        }
      },
      "filter": {
        "english_stemmer": { "type": "stemmer", "language": "english" }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "product_analyzer"
      },
      "category": { "type": "keyword" },
      "brand": { "type": "keyword" }
    }
  }
}
```

**Step 3: Reindex data**

```bash
POST /_reindex
{
  "source": { "index": "products_v1" },
  "dest": { "index": "products_v2" }
}
```

For large indexes, use `slices` for parallelism and `wait_for_completion: false` to run as a background task:

```bash
POST /_reindex?wait_for_completion=false
{
  "source": { "index": "products_v1" },
  "dest": { "index": "products_v2" },
  "conflicts": "proceed"
}
# Returns a task ID to monitor with GET /_tasks/<task_id>
```

**Step 4: Atomically swap the alias**

```bash
POST /_aliases
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add": { "index": "products_v2", "alias": "products" } }
  ]
}
```

This is atomic -- both actions happen in a single cluster state update. There's no moment where the alias points to neither or both indexes. Queries against `products` instantly start hitting `products_v2`.

**Step 5: Clean up**

```bash
# Verify the alias points to v2
GET /_alias/products

# Delete the old index when confident
DELETE /products_v1
```

**Handling writes during reindex:**

During the reindex, new data is still being written to `products_v1` (via the alias). Two strategies:

1. **Write alias separation**: Use separate read and write aliases. Point the write alias to `products_v2` before reindexing, so new data goes directly to the new index. Then reindex only existing data.
2. **Catch-up reindex**: After the main reindex, do a second pass for documents modified during the reindex window (using a timestamp filter). Then swap.

</details>

<details>
<summary>12. Implement autocomplete/search-as-you-type using edge ngrams — show the custom analyzer with edge ngram tokenizer at index time, the search query that uses a different analyzer at query time, compare with the completion suggester approach, and explain the tradeoffs in index size and relevance quality</summary>

**Edge ngram approach:**

Edge ngrams generate prefixes of each token at index time. When a user types "elast", it matches the indexed ngram "elast" from "elasticsearch".

```json
PUT /products_autocomplete
{
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete_index": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "edge_ngram_filter"]
        },
        "autocomplete_search": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      },
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 15
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete_index",
        "search_analyzer": "autocomplete_search"
      },
      "category": { "type": "keyword" }
    }
  }
}
```

**How it works:**

Indexing `"Wireless Headphones"` with the `autocomplete_index` analyzer produces:
```
"wi", "wir", "wire", "wirel", "wirele", "wireles", "wireless",
"he", "hea", "head", "headp", "headph", "headpho", "headphon", "headphone", "headphones"
```

At search time, the `autocomplete_search` analyzer only produces `["wire"]` for the input `"wire"` -- no ngrams. This matches the indexed ngram `"wire"` from `"wireless"`.

**Why different analyzers are critical:** If you used the edge ngram analyzer at search time too, searching "wire" would produce ngrams `["wi", "wir", "wire"]`, and `"wi"` would match `"window"`, `"winter"`, etc. Using a simple analyzer at query time ensures only the full typed input is matched.

**Search query:**

```json
POST /products_autocomplete/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "name": {
            "query": "wire head",
            "operator": "and"
          }
        }
      },
      "filter": {
        "term": { "category": "electronics" }
      }
    }
  }
}
```

**Completion suggester approach:**

```json
PUT /products_suggest
{
  "mappings": {
    "properties": {
      "name_suggest": {
        "type": "completion"
      },
      "name": {
        "type": "text"
      }
    }
  }
}

PUT /products_suggest/_doc/1
{
  "name": "Wireless Headphones",
  "name_suggest": {
    "input": ["Wireless Headphones", "Headphones Wireless"],
    "weight": 10
  }
}

POST /products_suggest/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "wire",
      "completion": {
        "field": "name_suggest",
        "fuzzy": { "fuzziness": 1 },
        "size": 5
      }
    }
  }
}
```

**Comparison:**

| Aspect | Edge ngrams | Completion suggester |
|--------|-------------|---------------------|
| Speed | Fast (regular inverted index) | Fastest (in-memory FST data structure) |
| Index size | Larger (many ngram tokens per term) | Moderate (FST is compact) |
| Relevance | Full BM25 scoring, can combine with filters and bool queries | No scoring -- only prefix match with optional weight |
| Flexibility | Can search across multiple fields, combine with other queries | Limited to prefix matching on a single field |
| Fuzzy support | Via `fuzziness` on match query | Built-in fuzzy on the suggester |
| Middle-of-word match | No (edge = from beginning only) | No (prefix only) |
| Use case | Primary search box where autocomplete integrates with full search | Dedicated autocomplete dropdown with pre-defined suggestions |

**Recommendation:** Use edge ngrams for the main search bar where you want autocomplete results with full search relevance (filtered, scored, aggregated). Use the completion suggester for a fast, lightweight suggestion dropdown (e.g., "did you mean..." or search box suggestions) where speed matters more than scoring flexibility.

</details>

## Practical — Query Construction & Features

<details>
<summary>13. Build a search query using bool queries (must, should, filter) — show a product search that combines full-text search on name/description (must), boosts by rating (should), filters by category and price range (filter), and explain why filter context skips scoring and improves performance</summary>

```json
POST /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "wireless noise cancelling headphones",
            "fields": ["name^3", "description"],
            "type": "best_fields"
          }
        }
      ],
      "should": [
        {
          "range": {
            "rating": {
              "gte": 4.5,
              "boost": 2
            }
          }
        },
        {
          "term": {
            "featured": {
              "value": true,
              "boost": 1.5
            }
          }
        }
      ],
      "filter": [
        {
          "term": { "category": "electronics" }
        },
        {
          "range": {
            "price": {
              "gte": 50,
              "lte": 300
            }
          }
        },
        {
          "term": { "in_stock": true }
        }
      ],
      "minimum_should_match": 0
    }
  },
  "sort": [
    { "_score": "desc" },
    { "sales_count": "desc" }
  ],
  "from": 0,
  "size": 20
}
```

**How each clause works:**

- **`must`**: Documents MUST match. Contributes to the relevance score. Here, the `multi_match` searches across `name` (boosted 3x) and `description`. Only documents matching the text query are returned.

- **`should`**: Documents SHOULD match but don't have to (when `minimum_should_match` is 0 and `must` is present). Matching `should` clauses **boost the score** -- products rated 4.5+ or marked as featured rank higher but aren't excluded if they don't match.

- **`filter`**: Documents MUST match, but **no scoring happens**. Category must be "electronics", price must be 50-300, and the product must be in stock. These are yes/no conditions that narrow results without affecting ranking.

**Why filter context skips scoring and improves performance:**

1. **No score calculation**: BM25 scoring involves TF, IDF, and field length normalization per document per term. Skipping this math for filter clauses is a significant CPU savings, especially across millions of documents.

2. **Cacheable**: Filter results are cached in a bitset (a compact representation of which documents match). The first execution of `category: electronics` builds the bitset; subsequent queries reuse it. Scored queries can't be cached this way because scores depend on the full query context.

3. **Bitset intersection**: Multiple filters are combined using fast bitwise AND operations. Intersecting two cached bitsets is orders of magnitude faster than scoring each document individually.

**Rule of thumb:** If a clause answers "does this match yes/no?" (category, price range, availability, date ranges), put it in `filter`. If a clause answers "how well does this match?" (text relevance, rating preference, recency), put it in `must` or `should`.

</details>

<details>
<summary>14. Implement multi-field search with boosting and fuzzy matching — show a query that searches across title (boosted 3x) and description (boosted 1x), add fuzziness for typo tolerance, and demonstrate how the boosting affects result ordering. Explain when fuzzy matching hurts relevance more than it helps</summary>

```json
POST /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "wireles headphnes",
            "fields": ["name^3", "description"],
            "type": "best_fields",
            "fuzziness": "AUTO",
            "prefix_length": 2
          }
        }
      ]
    }
  }
}
```

**How boosting affects ordering:**

With `name^3`, a match in the name field contributes 3x the score of the same match in description. Given two products:

- Product A: name = "Wireless Headphones Pro", description = "Premium audio device"
- Product B: name = "Audio Device Pro", description = "Best wireless headphones for travel"

Both match "wireless headphones", but Product A matches in the `name` field (3x boost) while Product B matches in `description` (1x). Product A ranks significantly higher.

**How fuzziness works:**

`"AUTO"` sets the edit distance based on term length:
- 0-2 characters: exact match required
- 3-5 characters: 1 edit allowed
- 6+ characters: 2 edits allowed

So `"wireles"` (7 chars, missing an 's') is within 2 edits of "wireless" -- match found. `"headphnes"` (9 chars, missing an 'o') is also within 2 edits of "headphones" -- match found.

**`prefix_length: 2`** requires the first 2 characters to match exactly. This prevents absurd fuzzy matches (e.g., "cat" matching "bat") and dramatically reduces the number of candidate terms Elasticsearch evaluates.

**`best_fields` vs `most_fields`:**

```json
// best_fields (default): Takes the highest-scoring field match
// Good when you expect the answer to be in one field
"type": "best_fields"

// most_fields: Sums scores across all matching fields
// Good when matching across multiple fields increases confidence
"type": "most_fields"

// cross_fields: Treats all fields as one big field
// Good for searching a person's name split across first_name/last_name
"type": "cross_fields"
```

**When fuzzy matching hurts more than it helps:**

1. **Short terms**: "go" with fuzziness could match "to", "do", "no" -- meaningless results. This is why `prefix_length` and `AUTO` are essential.

2. **High-cardinality keyword fields**: Fuzzy matching on product SKUs or IDs produces nonsensical matches. Only use fuzzy on natural language text fields.

3. **Precise domains**: Medical, legal, or scientific search where "affect" vs "effect" are meaningfully different. Fuzzy matching conflates distinct concepts.

4. **Performance at scale**: Fuzzy queries expand to all terms within the edit distance, then OR them together. On large indexes with diverse vocabulary, this can be expensive. Mitigate with `prefix_length: 2` and `max_expansions: 50` (limits candidate terms).

5. **Non-Latin scripts**: Edit distance doesn't map well to character-based languages (Chinese, Japanese) where a single character change alters meaning completely.

**Practical recommendation:** Enable `fuzziness: "AUTO"` with `prefix_length: 2` for user-facing search boxes where typos are common. Disable it for programmatic queries, exact-match scenarios, and API-driven search where input is controlled.

</details>

<details>
<summary>15. Implement custom relevance scoring using function_score — show a query that combines text relevance with recency boost (newer products rank higher) and popularity boost (more sales rank higher), explain how the score functions combine, and demonstrate tuning the weights to get the desired ranking</summary>

```json
POST /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query": "running shoes",
          "fields": ["name^3", "description"]
        }
      },
      "functions": [
        {
          "gauss": {
            "created_at": {
              "origin": "now",
              "scale": "30d",
              "offset": "7d",
              "decay": 0.5
            }
          },
          "weight": 10
        },
        {
          "field_value_factor": {
            "field": "sales_count",
            "modifier": "log1p",
            "factor": 0.5,
            "missing": 0
          },
          "weight": 5
        },
        {
          "field_value_factor": {
            "field": "rating",
            "modifier": "none",
            "factor": 1,
            "missing": 3.0
          },
          "weight": 3
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply",
      "max_boost": 50
    }
  }
}
```

Building on the `function_score` example from Q4 (which showed recency decay and popularity boosting), here we add a third signal (rating) and examine how `score_mode` and `boost_mode` combine them.

**How each function works:**

**Recency decay (gauss):** Same concept as Q4 -- a gaussian decay centered on "now" with an offset and scale that control the decay curve.
- `offset: "7d"` -- products within the last 7 days get full score (no decay).
- `scale: "30d"` -- at 30 days past the offset, the score decays to `decay: 0.5` (50%).
- A product created yesterday gets ~1.0, 30 days ago gets ~0.5, 90 days ago gets ~0.1.

**Popularity boost (field_value_factor):** Same `log1p` approach as Q4, dampening high-volume outliers.
- `missing: 0` -- products without a sales_count field are treated as 0.

**Rating boost:**
- Linear factor on the rating field. A 5-star product gets 5x the boost of a 1-star product.
- `missing: 3.0` -- unrated products get a neutral middle score.

**How scores combine:**

1. **`score_mode: "sum"`** -- The individual function scores are summed: `recency_score * 10 + popularity_score * 5 + rating_score * 3`. The `weight` on each function scales its contribution before summing.

2. **`boost_mode: "multiply"`** -- The combined function score is multiplied with the original query score: `final_score = query_score * function_scores_sum`. This means text relevance still matters -- a poor text match with great popularity scores lower than a great text match with mediocre popularity.

3. **`max_boost: 50`** -- Caps the function score multiplier at 50 to prevent extreme outliers.

**Other score_mode options:**
- `multiply`: Functions multiply together (use when each factor should compound).
- `avg`: Average of functions (use when factors should balance equally).
- `first`: Only the first matching function applies.
- `max`/`min`: Take the highest/lowest function score.

**Tuning the weights:**

The weights are relative to each other and require iteration. Practical approach:

1. Start with equal weights and examine results.
2. Use the `explain: true` parameter to see score breakdowns:

```json
POST /products/_search
{
  "explain": true,
  "query": { ... }
}
```

3. Adjust weights based on which signal should dominate. If fresh-but-unpopular products rank too high, reduce the recency weight or increase the popularity weight.
4. Test with representative queries and expected result orderings. There's no formula -- tuning is empirical.

**Common pitfall:** Using `boost_mode: "replace"` discards the text relevance score entirely, making text matching irrelevant. Almost always wrong unless you're building a non-text-based ranking system.

</details>

## Practical — Integration & Operations

<details>
<summary>16. Implement bulk indexing in Node.js/TypeScript using the official Elasticsearch client — show how to batch documents, handle partial failures in the bulk response (some documents succeed while others fail), and implement retry logic for transient errors. What breaks if you don't check the bulk response item-by-item?</summary>

**Using the bulk helper (recommended):**

```typescript
import { Client } from '@elastic/elasticsearch';

const client = new Client({
  node: 'http://localhost:9200',
  maxRetries: 3,
  requestTimeout: 60_000,
});

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  category: string;
}

async function bulkIndexProducts(products: Product[]): Promise<void> {
  const result = await client.helpers.bulk({
    datasource: products,
    onDocument(doc) {
      return {
        index: { _index: 'products', _id: doc.id },
      };
    },
    onDrop(doc) {
      // Called when a document fails after all retries
      console.error(
        `Failed to index document ${doc.document.id}: ${doc.error?.reason}`,
      );
      // In production: send to dead-letter queue or alert
    },
    flushBytes: 5_000_000, // Flush when 5MB accumulated
    concurrency: 3,        // 3 concurrent bulk requests
    retries: 3,            // Retry failed docs 3 times
    refreshOnCompletion: true,
  });

  console.log(
    `Indexed ${result.successful}/${result.total} documents ` +
    `(${result.failed} failed, ${result.retry} retries) in ${result.time}ms`,
  );
}
```

**Using the manual bulk API (when you need full control):**

```typescript
async function bulkIndexManual(products: Product[]): Promise<void> {
  const BATCH_SIZE = 500;

  for (let i = 0; i < products.length; i += BATCH_SIZE) {
    const batch = products.slice(i, i + BATCH_SIZE);
    const operations = batch.flatMap((doc) => [
      { index: { _index: 'products', _id: doc.id } },
      doc,
    ]);

    const response = await client.bulk({ operations, refresh: false });

    if (response.errors) {
      const failedDocs: Array<{ doc: Product; error: unknown }> = [];

      response.items.forEach((item, idx) => {
        const operation = item.index ?? item.update ?? item.delete;
        if (operation?.error) {
          const isRetryable =
            operation.status === 429 || // Too many requests
            operation.status === 503;   // Service unavailable

          if (isRetryable) {
            failedDocs.push({ doc: batch[idx], error: operation.error });
          } else {
            // Non-retryable: log and skip (bad mapping, invalid data)
            console.error(
              `Permanent failure for doc ${batch[idx].id}:`,
              operation.error,
            );
          }
        }
      });

      // Retry retryable failures with backoff
      if (failedDocs.length > 0) {
        await retryWithBackoff(failedDocs.map((f) => f.doc), 3);
      }
    }
  }
}

async function retryWithBackoff(
  docs: Product[],
  maxRetries: number,
): Promise<void> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    await new Promise((resolve) =>
      setTimeout(resolve, Math.pow(2, attempt) * 1000),
    );

    const operations = docs.flatMap((doc) => [
      { index: { _index: 'products', _id: doc.id } },
      doc,
    ]);

    const response = await client.bulk({ operations });
    if (!response.errors) return;

    // Filter to only still-failing docs for next retry
    docs = docs.filter((_, idx) => {
      const item = response.items[idx];
      const op = item.index ?? item.update ?? item.delete;
      return op?.error != null;
    });
  }

  console.error(`${docs.length} documents failed after ${maxRetries} retries`);
}
```

**What breaks if you don't check item-by-item:**

The bulk API returns HTTP 200 even when individual documents fail. The top-level `errors: true` flag tells you something went wrong, but the only way to know WHICH documents failed is to iterate `response.items`.

If you only check the HTTP status:
- **Silent data loss**: Failed documents are never indexed, but your application thinks everything succeeded.
- **No retry opportunity**: Transient failures (429, 503) could have been retried, but you don't know which documents to retry.
- **Mapping errors go undetected**: A field type mismatch silently drops documents. You discover the gap weeks later when a user can't find their product in search.
- **Partial index corruption**: Some documents from a batch are indexed, others aren't, leading to inconsistent search results with no audit trail.

</details>

<details>
<summary>17. Build a search endpoint in Node.js/TypeScript that translates HTTP query parameters into Elasticsearch queries — show the client setup with connection management and retries, demonstrate how you handle common failures (index not found, timeout, version conflict), and explain what error responses the caller should receive for each.</summary>

```typescript
import { Client, errors } from '@elastic/elasticsearch';
import express, { Request, Response, NextFunction } from 'express';

// Client setup with connection management
const esClient = new Client({
  nodes: [
    'http://es-node-1:9200',
    'http://es-node-2:9200',
    'http://es-node-3:9200',
  ],
  maxRetries: 3,
  requestTimeout: 10_000,   // 10s per request
  sniffOnStart: true,       // Discover cluster nodes on startup
  sniffInterval: 60_000,    // Re-discover every 60s
  compression: true,
});

interface SearchParams {
  q?: string;
  category?: string;
  minPrice?: string;
  maxPrice?: string;
  page?: string;
  size?: string;
  sort?: string;
}

function buildSearchQuery(params: SearchParams) {
  const must: object[] = [];
  const filter: object[] = [];

  if (params.q) {
    must.push({
      multi_match: {
        query: params.q,
        fields: ['name^3', 'description'],
        fuzziness: 'AUTO',
        prefix_length: 2,
      },
    });
  }

  if (params.category) {
    filter.push({ term: { category: params.category } });
  }

  if (params.minPrice || params.maxPrice) {
    const range: Record<string, number> = {};
    if (params.minPrice) range.gte = Number(params.minPrice);
    if (params.maxPrice) range.lte = Number(params.maxPrice);
    filter.push({ range: { price: range } });
  }

  const page = Math.max(1, Number(params.page) || 1);
  const size = Math.min(100, Math.max(1, Number(params.size) || 20));

  const sort: object[] = params.sort === 'price_asc'
    ? [{ price: 'asc' }]
    : params.sort === 'price_desc'
      ? [{ price: 'desc' }]
      : [{ _score: 'desc' }];

  return {
    index: 'products',
    query: {
      bool: {
        must: must.length > 0 ? must : [{ match_all: {} }],
        filter,
      },
    },
    from: (page - 1) * size,
    size,
    sort,
  };
}

const app = express();

app.get('/api/search', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const searchBody = buildSearchQuery(req.query as SearchParams);
    const result = await esClient.search(searchBody);

    const total = typeof result.hits.total === 'number'
      ? result.hits.total
      : result.hits.total?.value ?? 0;

    res.json({
      total,
      page: Number(req.query.page) || 1,
      results: result.hits.hits.map((hit) => ({
        id: hit._id,
        score: hit._score,
        ...hit._source as object,
      })),
    });
  } catch (error) {
    next(error);
  }
});

// Centralized error handler for ES errors
app.use((error: Error, _req: Request, res: Response, _next: NextFunction) => {
  if (error instanceof errors.ResponseError) {
    switch (error.statusCode) {
      case 404:
        // Index not found — caller gets 503 (infrastructure issue, not client error)
        res.status(503).json({
          error: 'Search service temporarily unavailable',
          message: 'Search index not found. This is being resolved.',
        });
        break;
      case 400:
        // Bad query syntax — caller error
        res.status(400).json({
          error: 'Invalid search query',
          message: error.meta.body?.error?.reason ?? 'Malformed query',
        });
        break;
      case 409:
        // Version conflict — safe to retry
        res.status(409).json({
          error: 'Conflict',
          message: 'Document was modified concurrently. Retry the request.',
        });
        break;
      case 429:
        // ES is overloaded
        res.status(503).json({
          error: 'Search service busy',
          message: 'Too many requests. Retry after a short delay.',
        });
        break;
      default:
        res.status(502).json({
          error: 'Search service error',
          message: 'Unexpected error from search backend',
        });
    }
  } else if (error instanceof errors.TimeoutError) {
    // Request timed out
    res.status(504).json({
      error: 'Search timeout',
      message: 'Search took too long. Try a more specific query.',
    });
  } else if (error instanceof errors.ConnectionError) {
    // Can't reach ES at all
    res.status(503).json({
      error: 'Search service unavailable',
      message: 'Cannot connect to search backend.',
    });
  } else {
    res.status(500).json({ error: 'Internal server error' });
  }

  // Always log the full error server-side
  console.error('Search error:', error);
});
```

**Error mapping rationale:**

| ES Error | HTTP to caller | Why |
|----------|---------------|-----|
| 404 (index not found) | 503 | Infrastructure issue, not the caller's fault. They shouldn't know about index names. |
| 400 (bad query) | 400 | Caller sent invalid parameters. Include a sanitized reason. |
| 409 (version conflict) | 409 | Relevant for write operations. Caller should retry. |
| 429 (too many requests) | 503 | Backpressure from ES. Caller should retry with backoff. |
| Timeout | 504 | Gateway timeout. Suggest narrowing the query. |
| Connection error | 503 | ES cluster is unreachable. Transient -- retry. |

**Key detail:** Never expose internal Elasticsearch error details (index names, mapping info, shard details) to external callers. Log them server-side, return sanitized messages to the client.

</details>

<details>
<summary>18. Set up data synchronization between PostgreSQL and Elasticsearch using CDC — show the Debezium connector configuration for capturing PostgreSQL changes, the Kafka consumer that transforms and indexes documents into Elasticsearch, and the error handling for when Elasticsearch rejects a document. Compare with the simpler scheduled reindexing approach (show the cron-based reindexing script) and explain when real-time CDC isn't worth the operational complexity.</summary>

**Step 1: PostgreSQL setup for CDC**

```sql
-- Enable logical replication (postgresql.conf or ALTER SYSTEM)
ALTER SYSTEM SET wal_level = 'logical';
-- Restart PostgreSQL after this change

-- Create a publication for the tables you want to capture
CREATE PUBLICATION product_publication FOR TABLE products;
```

**Step 2: Debezium connector configuration**

```json
{
  "name": "products-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-host",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "${DEBEZIUM_DB_PASSWORD}",
    "database.dbname": "myapp",
    "topic.prefix": "myapp",
    "table.include.list": "public.products",
    "plugin.name": "pgoutput",
    "publication.name": "product_publication",
    "slot.name": "debezium_products",
    "snapshot.mode": "initial",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter"
  }
}
```

Key settings:
- `snapshot.mode: "initial"` -- Takes a full snapshot of existing data on first start, then switches to streaming changes.
- `ExtractNewRecordState` -- Flattens the Debezium envelope to just the `after` state (simpler to consume).
- `delete.handling.mode: "rewrite"` -- Adds a `__deleted` field for delete events instead of producing null values.

**Step 3: Kafka consumer that indexes to Elasticsearch**

```typescript
import { Client } from '@elastic/elasticsearch';
import { Kafka, EachMessagePayload } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'es-indexer',
  brokers: ['kafka-1:9092', 'kafka-2:9092'],
});

const esClient = new Client({
  node: 'http://localhost:9200',
  maxRetries: 3,
});

const consumer = kafka.consumer({ groupId: 'es-indexer-group' });

interface ProductEvent {
  id: number;
  name: string;
  description: string;
  price: number;
  category: string;
  updated_at: string;
  __deleted?: string; // "true" for delete events
}

function transformToDocument(event: ProductEvent) {
  return {
    name: event.name,
    description: event.description,
    price: event.price / 100, // cents to dollars
    category: event.category,
    updated_at: event.updated_at,
  };
}

async function handleMessage({ message }: EachMessagePayload): Promise<void> {
  if (!message.value) return;

  const event: ProductEvent = JSON.parse(message.value.toString());
  const docId = String(event.id);

  try {
    if (event.__deleted === 'true') {
      await esClient.delete({
        index: 'products',
        id: docId,
      }).catch((err) => {
        // Ignore 404 -- document already deleted
        if (err.statusCode !== 404) throw err;
      });
    } else {
      await esClient.index({
        index: 'products',
        id: docId,
        document: transformToDocument(event),
      });
    }
  } catch (error: unknown) {
    const statusCode = (error as { statusCode?: number }).statusCode;

    if (statusCode === 400) {
      // Mapping error -- document shape doesn't match index mapping
      // Send to dead-letter topic for manual investigation
      console.error(`Rejected document ${docId}: mapping error`, error);
      await sendToDeadLetter(event, error);
    } else if (statusCode === 429 || statusCode === 503) {
      // Transient -- throw to trigger consumer retry/rebalance
      throw error;
    } else {
      console.error(`Unexpected error indexing ${docId}:`, error);
      await sendToDeadLetter(event, error);
    }
  }
}

// Shared producer — initialized once at startup to avoid per-call TCP/auth overhead
const dlqProducer = kafka.producer();

async function sendToDeadLetter(
  event: ProductEvent,
  error: unknown,
): Promise<void> {
  await dlqProducer.send({
    topic: 'es-indexer-dead-letter',
    messages: [{
      key: String(event.id),
      value: JSON.stringify({ event, error: String(error), timestamp: new Date().toISOString() }),
    }],
  });
}

async function start(): Promise<void> {
  await dlqProducer.connect();
  await consumer.connect();
  await consumer.subscribe({ topic: 'myapp.public.products', fromBeginning: true });
  await consumer.run({ eachMessage: handleMessage });
}

start().catch(console.error);
```

**Simpler alternative: Scheduled reindexing**

```typescript
import { Client } from '@elastic/elasticsearch';
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const esClient = new Client({ node: 'http://localhost:9200' });

async function reindexProducts(): Promise<void> {
  // Track last successful run
  const lastRun = await getLastRunTimestamp(); // from a state table or file

  const { rows } = await pool.query(
    `SELECT id, name, description, price, category, updated_at
     FROM products
     WHERE updated_at > $1
     ORDER BY id`,
    [lastRun],
  );

  if (rows.length === 0) {
    console.log('No changes since last run');
    return;
  }

  const result = await esClient.helpers.bulk({
    datasource: rows,
    onDocument(doc) {
      return { index: { _index: 'products', _id: String(doc.id) } };
    },
    onDrop(doc) {
      console.error(`Failed to index: ${doc.document.id}`);
    },
  });

  console.log(`Reindexed ${result.successful}/${result.total} products`);
  await saveLastRunTimestamp(new Date().toISOString());

  // Handle deletes: requires soft-delete column or a separate deleted_products table
  const { rows: deleted } = await pool.query(
    `SELECT id FROM products WHERE deleted_at > $1`,
    [lastRun],
  );

  for (const row of deleted) {
    await esClient.delete({ index: 'products', id: String(row.id) })
      .catch(() => { /* ignore 404 */ });
  }
}

// Run via cron: */5 * * * * (every 5 minutes)
reindexProducts().catch(console.error);
```

**When CDC isn't worth it:**

See Q8 for the full tradeoff comparison table between CDC, scheduled reindex, and dual writes. The key decision factors:

- **Use CDC** when search freshness matters (< 5 seconds), change volume is high, or you already run Kafka.
- **Use scheduled reindex** when minutes of staleness is acceptable, change volume is low, or adding Kafka just for search sync is overkill.

The operational cost of CDC is significant: Kafka cluster, Debezium connectors, monitoring replication lag, managing connector failures, WAL disk usage. For a product catalog updated a few hundred times per hour where 5-minute search staleness is fine, scheduled reindexing is dramatically simpler and perfectly adequate.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>19. Search queries are taking 2 seconds instead of 200ms — walk through diagnosing the performance issue: use query profiling to identify expensive clauses (wildcard queries, deeply nested aggregations, script scoring), check shard distribution and cluster health, identify if the refresh interval is too aggressive, and apply the fix</summary>

**Step 1: Profile the slow query**

```json
POST /products/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must": [
        { "wildcard": { "name": "*headphones*" } }
      ],
      "should": [
        {
          "script_score": {
            "query": { "match_all": {} },
            "script": { "source": "doc['sales_count'].value * 0.1" }
          }
        }
      ]
    }
  }
}
```

The `profile: true` response shows time spent in each query phase per shard. Look for:
- Which clause takes the most time (the `time_in_nanos` field)
- Whether the bottleneck is in the query phase (finding matches) or the collector phase (scoring/sorting)

**Step 2: Identify common expensive patterns**

| Pattern | Why it's slow | Fix |
|---------|-------------|-----|
| Leading wildcard (`*headphones*`) | Scans every term in the inverted index -- bypasses the fast dictionary lookup | Use `match` or `match_phrase` query. For partial matching, use edge ngrams (question 12) |
| `script_score` on every document | Executes a script for every matching doc | Pre-compute values into indexed fields at index time. Use `field_value_factor` instead |
| Deep nested aggregations | Each nesting level multiplies bucket count | Reduce nesting depth, use `composite` aggregation with pagination |
| `match_phrase` with high slop | Generates expensive position-aware matching | Reduce slop value or use `match` + `minimum_should_match` |
| Large `from` offset (deep pagination) | ES must score and sort all documents up to `from + size` | Use `search_after` for deep pagination |

**Step 3: Check cluster health and shard distribution**

```bash
# Cluster health
GET /_cluster/health
# Look for: status (yellow/red), relocating_shards, unassigned_shards

# Shard distribution -- are shards balanced?
GET /_cat/shards/products?v&s=store:desc
# Look for: uneven shard sizes, shards concentrated on one node

# Hot threads -- what's consuming CPU?
GET /_nodes/hot_threads

# Node stats -- memory pressure?
GET /_nodes/stats/jvm
# Look for: heap usage > 75%, frequent GC
```

**Common cluster-level causes:**
- Uneven shard distribution: One node handles most shards, becomes the bottleneck. Fix with shard allocation awareness or force rebalancing.
- Yellow/red health: Replicas unallocated means fewer shards to distribute search load.
- High GC pressure: Heap too small or too many field data structures in memory. Increase heap or reduce field cardinality.

**Step 4: Check refresh interval impact**

```bash
GET /products/_settings?filter_path=*.settings.index.refresh_interval
```

If refresh interval is `1s` and you're doing heavy indexing simultaneously, each refresh creates a new segment. Many tiny segments slow searches because each query must search every segment before merges consolidate them.

```bash
# Check segment count
GET /_cat/segments/products?v&s=size:desc

# If too many segments, force merge
POST /products/_forcemerge?max_num_segments=5
```

**Step 5: Apply fixes for the example**

Before (slow -- leading wildcard + script scoring):

```json
{
  "query": {
    "bool": {
      "must": [{ "wildcard": { "name": "*headphones*" } }],
      "should": [{
        "script_score": {
          "query": { "match_all": {} },
          "script": { "source": "doc['sales_count'].value * 0.1" }
        }
      }]
    }
  }
}
```

After (fast -- match query + field_value_factor):

```json
{
  "query": {
    "function_score": {
      "query": {
        "match": { "name": "headphones" }
      },
      "functions": [{
        "field_value_factor": {
          "field": "sales_count",
          "modifier": "log1p",
          "factor": 0.1
        }
      }],
      "boost_mode": "sum"
    }
  }
}
```

**Systematic checklist:**
1. Profile the query -- find the expensive clause
2. Replace wildcards with proper full-text queries or ngrams
3. Replace scripts with pre-computed fields or built-in score functions
4. Check shard count and distribution
5. Check segment count -- force merge if fragmented
6. Adjust refresh interval for write-heavy workloads
7. Add caching: use `filter` context for non-scoring clauses (cached as bitsets)
8. For deep pagination, switch from `from/size` to `search_after`

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>20. Tell me about a time you implemented search for a product or feature — what technology did you choose (PostgreSQL FTS, Elasticsearch, Algolia), how did you design the index, and what challenges did you face with relevance or performance?</summary>

**What the interviewer is looking for:**
- Ability to evaluate technology options and justify the choice
- Understanding of index design decisions (field types, analyzers, denormalization)
- Awareness of real-world challenges beyond the happy path
- Problem-solving approach when search results aren't good enough

**Key points to hit:**
- The business context that drove the search requirement
- Why you chose the specific technology (and what you considered but rejected)
- How you designed the mapping/index -- denormalization decisions, field type choices
- A specific challenge you faced (poor relevance, slow queries, data freshness) and how you resolved it
- What you'd do differently with hindsight

**Suggested structure (STAR+):**

1. **Situation**: Describe the product and the search requirement. "We had a product catalog with 500K items and needed faceted search with autocomplete for the storefront."
2. **Task**: Your specific role. "I was responsible for designing the search index and building the query layer."
3. **Action (technology choice)**: "I evaluated PostgreSQL FTS vs Elasticsearch. PostgreSQL FTS was insufficient because we needed faceted navigation, fuzzy matching, and autocomplete. Algolia was too expensive at our document volume. We went with Elasticsearch."
4. **Action (design)**: "I designed the mapping with multi-field types for name (text + keyword), nested attributes for faceted filtering, and edge ngram analyzers for autocomplete. We denormalized category names into the product document to avoid joins."
5. **Action (challenge)**: "Our initial relevance was poor -- exact matches on product names ranked below description matches. I added field boosting (name^3), phrase matching as a should clause, and a function_score for popularity. We used the explain API to debug scoring."
6. **Result**: Quantify if possible -- "Search latency dropped from 800ms to 120ms, and click-through rate on search results improved 40%."
7. **Reflection**: "If I did it again, I'd invest in relevance testing earlier -- we spent weeks tuning scoring by manual inspection instead of building automated relevance test suites."

</details>

<details>
<summary>21. Tell me about a time you dealt with data synchronization between your primary database and search index — what approach did you use (CDC, dual write, scheduled sync), what consistency issues did you encounter, and how did you resolve them?</summary>

**What the interviewer is looking for:**
- Understanding of the consistency challenges between two data stores
- Ability to evaluate sync approaches and articulate tradeoffs
- Experience debugging subtle data drift issues
- Awareness of operational complexity vs engineering simplicity

**Key points to hit:**
- Why synchronization was needed (what was the source of truth, what was the search index for)
- The approach you chose and why you rejected alternatives
- A specific consistency issue you encountered and how you detected it
- The resolution and any monitoring/safeguards you put in place

**Suggested structure (STAR+):**

1. **Situation**: "We had a PostgreSQL database as the source of truth for [products/orders/content] and Elasticsearch for the customer-facing search. We needed to keep them in sync as data changed."
2. **Task**: "I was responsible for designing and implementing the synchronization pipeline."
3. **Action (approach chosen)**: Describe which approach you used and why:
   - *CDC*: "We chose Debezium + Kafka because we needed near-real-time search freshness and already had Kafka in our stack. We rejected dual writes due to the partial failure risks (as covered in question 8)."
   - *Scheduled sync*: "We started with a cron job running every 5 minutes because our search freshness requirements were relaxed and we didn't want the operational overhead of Kafka + Debezium for a small dataset."
   - *Dual write (if you inherited it)*: "We inherited a dual-write approach and I was tasked with fixing the consistency issues it caused."
4. **Action (the challenge)**: Describe a specific consistency issue:
   - "We discovered documents in Elasticsearch that didn't match the database -- a race condition where two rapid updates to the same product arrived at Kafka out of order."
   - "Deletes weren't propagating -- our scheduled sync only queried `updated_at`, and hard-deleted rows disappeared from the query entirely."
   - "During a bulk migration, the CDC connector fell behind and replication lag grew to 30 minutes. Users were seeing stale search results."
5. **Action (resolution)**:
   - "We added optimistic concurrency using version numbers -- each document carries a version, and the consumer uses `version_type: external` so out-of-order updates are rejected."
   - "We switched from hard deletes to soft deletes with a `deleted_at` column, and added a reconciliation job that runs nightly comparing document counts and sampling random records."
   - "We built a monitoring dashboard tracking replication lag (Kafka consumer group lag), added alerts for lag > 60 seconds, and implemented backpressure by pausing the connector when ES was overloaded."
6. **Result**: "After the fix, we went from ~50 stale documents per day to zero detected inconsistencies over 3 months of monitoring."
7. **Reflection**: "The biggest lesson was that sync is never set-and-forget. You need continuous reconciliation and monitoring. The sync pipeline itself can fail silently -- detecting drift is as important as preventing it."

</details>
