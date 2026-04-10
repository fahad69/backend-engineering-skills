---
name: full-text-search
description: Use this skill when implementing search functionality, choosing between database search and Elasticsearch, building autocomplete, fuzzy search, relevance ranking, or when the user asks about full-text search, inverted indexes, Elasticsearch, search indexing, search sync, or why LIKE queries are slow.
---

# Full-Text Search — Backend Engineering

## Why Not SQL LIKE Queries

```sql
-- This is slow at scale:
SELECT * FROM products WHERE name LIKE '%laptop%'

-- Problems:
-- 1. Cannot use index (leading wildcard)
-- 2. Full table scan on every search
-- 3. No relevance ranking
-- 4. No typo tolerance
-- 5. No stemming ("running" matches "run")
-- 6. Performance degrades linearly with table size
```

Use `LIKE '%term%'` only on small tables (< 10,000 rows) where search is infrequent. For anything else: use a proper full-text search engine.

## The Inverted Index — How Search Engines Work

A normal database is document-centric:
```
Document 1 → { title: "Laptop Pro", description: "Fast laptop for work" }
Document 2 → { title: "Gaming Laptop", description: "High performance laptop" }
```

An inverted index is term-centric:
```
"laptop" → [Document 1, Document 2]
"pro"    → [Document 1]
"gaming" → [Document 2]
"fast"   → [Document 1]
"high"   → [Document 2]
```

When searching "laptop", immediately know which documents contain it — no scanning needed.

## When to Use Each

| Scenario | Use |
|----------|-----|
| < 10K rows, simple contains | PostgreSQL `LIKE` or `ILIKE` |
| 10K–100K rows, structured data | PostgreSQL full-text search (`tsvector`) |
| > 100K rows | Elasticsearch / OpenSearch / Typesense |
| Typo tolerance required | Elasticsearch |
| Autocomplete / type-ahead | Elasticsearch completion suggester |
| Faceted search (filter by category + search) | Elasticsearch |
| Log analytics | Elasticsearch (ELK stack) |
| Product search with relevance | Elasticsearch |

## Elasticsearch Core Concepts

### Index = Database Table
An Elasticsearch index stores documents. Each document is a JSON object.

```json
// Product index document
{
  "id": "prod_123",
  "title": "Apple MacBook Pro 14-inch",
  "description": "Powerful laptop for professionals",
  "category": "laptops",
  "brand": "Apple",
  "price": 1999.00,
  "rating": 4.8,
  "in_stock": true
}
```

### Mapping = Schema
```json
{
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "english" },
      "description": { "type": "text", "analyzer": "english" },
      "category": { "type": "keyword" },    // exact match only
      "brand": { "type": "keyword" },       // exact match only
      "price": { "type": "float" },
      "rating": { "type": "float" },
      "in_stock": { "type": "boolean" }
    }
  }
}
```

`text` fields are analyzed (tokenized, stemmed). Use for full-text search.
`keyword` fields are not analyzed. Use for exact match, filtering, aggregation.

## Search Query Types

### Match Query (Full-Text)
```json
{
  "query": {
    "match": {
      "title": "gaming laptop"
    }
  }
}
// Returns documents where title contains "gaming" OR "laptop"
```

### Multi-Match (Search Multiple Fields)
```json
{
  "query": {
    "multi_match": {
      "query": "gaming laptop",
      "fields": ["title^3", "description^1", "brand^2"],
      // ^3 = title is 3x more important than description
      "type": "best_fields"
    }
  }
}
```

### Bool Query (Combine Conditions)
```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "laptop" } }
      ],
      "filter": [
        { "term": { "in_stock": true } },
        { "term": { "category": "laptops" } },
        { "range": { "price": { "gte": 500, "lte": 2000 } } }
      ],
      "should": [
        { "term": { "brand": "Apple" } }
      ],
      "minimum_should_match": 0
    }
  }
}
```

- `must`: required, affects score
- `filter`: required, does NOT affect score (faster)
- `should`: optional, boosts score if present
- `must_not`: excluded

### Fuzzy Search (Typo Tolerance)
```json
{
  "query": {
    "match": {
      "title": {
        "query": "laotop",
        "fuzziness": "AUTO"
        // "AUTO" = 0 edits for 1-2 chars, 1 edit for 3-5 chars, 2 edits for 6+ chars
      }
    }
  }
}
```

### Autocomplete / Type-Ahead
```json
// Mapping: use completion type
{
  "mappings": {
    "properties": {
      "suggest": { "type": "completion" }
    }
  }
}

// Query
{
  "suggest": {
    "product_suggest": {
      "prefix": "lap",
      "completion": {
        "field": "suggest",
        "size": 10
      }
    }
  }
}
```

## Relevance Scoring

Elasticsearch scores documents using BM25 algorithm. Higher score = more relevant.

Factors:
- **Term frequency**: how many times the search term appears in the document
- **Inverse document frequency**: how rare the term is across all documents (rare = more meaningful)
- **Field length**: shorter fields give higher score (title match > body match)

You can boost fields:
```json
"fields": ["title^3", "description^1"]
// title matches are 3x more valuable than description matches
```

## Keeping Search Index in Sync

The search index is a derived view of your database. It must stay in sync.

### Option 1: Synchronous Write (Simple)
```python
def update_product(product_id, data):
    product = product_repo.update(product_id, data)   # update DB
    es_client.index("products", id=product_id, body=serialize(product))  # update index
    return product
```

Risk: if ES call fails, DB and index are out of sync.
Use for: small scale, where temporary inconsistency is acceptable.

### Option 2: Async via Background Job (Recommended at Scale)
```python
def update_product(product_id, data):
    product = product_repo.update(product_id, data)   # update DB
    enqueue("sync_product_to_search", {"product_id": product_id})  # async
    return product

def sync_product_to_search(product_id):
    product = product_repo.find(product_id)  # fetch fresh from DB
    es_client.index("products", id=product_id, body=serialize(product))
```

### Option 3: Change Data Capture (CDC) at Scale
Listen to database WAL (Write-Ahead Log) changes and sync to Elasticsearch automatically. No application code changes needed. Tools: Debezium + Kafka.

## Performance Rules

- **Never use leading wildcards**: `*laptop*` does not use the inverted index → full scan
- **Use filters for structured data**: `filter` clauses are cached, much faster than `must` for exact matches
- **Paginate**: always use `from` + `size` or `search_after` for deep pagination
- **Avoid `from` > 10,000**: use `search_after` cursor-based pagination for deep pages
- **Use source filtering**: only return fields you need (`_source: ["title", "price"]`)

## PostgreSQL Full-Text Search (For Smaller Scale)

```sql
-- Add tsvector column
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- Populate it
UPDATE products SET search_vector = to_tsvector('english', title || ' ' || description);

-- Index it
CREATE INDEX idx_products_search ON products USING gin(search_vector);

-- Query it
SELECT *, ts_rank(search_vector, query) AS rank
FROM products, to_tsquery('english', 'gaming & laptop') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

PostgreSQL full-text search is good up to ~100K rows. After that: Elasticsearch.
