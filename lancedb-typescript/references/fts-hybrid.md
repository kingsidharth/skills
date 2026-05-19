# Full-Text Search & Hybrid Search

## Full-text search (BM25)

Requires creating an FTS index first, then searching with a string query.

```typescript
// Create FTS index
await table.createIndex("text", { config: lancedb.Index.fts() });

// Search (auto-detects FTS when input is string)
const results = await table.search("puppy")
  .select(["text"])
  .limit(10)
  .toArray();
// Results include `_score` field (BM25 relevance score)
```

### Tokenizer options

Default tokenizer splits on punctuation/whitespace, lowercases, stems (English), removes stop words. Configurable via index creation options.

### Filtering with FTS

```typescript
// Pre-filter (default)
await table.search("puppy")
  .where("meta = 'foo'")
  .prefilter(true)
  .limit(10)
  .toArray();

// Post-filter
await table.search("apple")
  .where("meta = 'foo'")
  .prefilter(false)
  .limit(10)
  .toArray();
```

### Query types

**Terms query** (default): `"old man sea"` — matches any of the individual terms.

**Phrase query**: requires `with_position: true` on index creation. Matches exact word sequence.

**Fuzzy search**: tolerates typos via Levenshtein distance.

```typescript
import { MatchQuery } from "@lancedb/lancedb";

// Exact match
await table.search(new MatchQuery("crazily", "text")).limit(100).toArray();

// Fuzzy match (allows typos)
await table.search(new MatchQuery("craziou", "text", { fuzziness: 2 })).limit(100).toArray();

// Prefix match
await table.search(new MatchQuery("cra", "text", { prefixLength: 3 })).limit(100).toArray();
```

### Phrase search

```typescript
import { PhraseQuery } from "@lancedb/lancedb";

// Exact phrase
await table.search(new PhraseQuery("puppy runs", "text")).limit(100).toArray();

// Flexible phrase (slop allows words between terms)
await table.search(new PhraseQuery("puppy merrily", "text", { slop: 1 })).limit(100).toArray();
```

### Boolean queries

```typescript
import { BooleanQuery, MatchQuery, Occur } from "@lancedb/lancedb";

// AND — both must match
await table.search(new BooleanQuery([
  [Occur.Must, new MatchQuery("puppy", "text")],
  [Occur.Must, new MatchQuery("merrily", "text")],
])).limit(100).toArray();

// OR — at least one must match
await table.search(new BooleanQuery([
  [Occur.Should, new MatchQuery("puppy", "text")],
  [Occur.Should, new MatchQuery("merrily", "text")],
])).limit(100).toArray();
```

Must include at least one `Should` or `Must` clause (pure `MustNot` not allowed).

### Boosting & multi-field search

```typescript
import { BoostQuery, MultiMatchQuery, MatchQuery } from "@lancedb/lancedb";

// Boost: promote "runs" matches, demote "puppy" matches
await table.search(new BoostQuery(
  new MatchQuery("runs", "text"),
  new MatchQuery("puppy", "text"),
  { negativeBoost: 0.2 }
)).limit(100).toArray();

// Search across multiple columns
await table.search(new MultiMatchQuery("crazily", ["text", "text2"])).limit(100).toArray();

// With field boosting
await table.search(new MultiMatchQuery("crazily", ["text", "text2"], {
  boosts: [1.0, 2.0]
})).limit(100).toArray();
```

### Substring search (n-gram)

```typescript
// Create n-gram index for substring matching
await table.createIndex("text", {
  config: lancedb.Index.fts({ baseTokenizer: "ngram", ngramMinLength: 3, ngramMaxLength: 3 })
});
```

## Hybrid search

Combines vector similarity + FTS with a reranker to merge results.

```typescript
// Requires both vector index and FTS index
const results = await table.search("flower moon", {
  queryType: "hybrid",
  vectorColumnName: "vector",
  ftsColumns: "text",
})
  .rerank(new lancedb.RRFReranker())
  .limit(10)
  .toArray();
```

### Explicit vector + text query

```typescript
await table.search({ queryType: "hybrid" })
  .vector([0.1, 0.2, 0.3, 0.4, 0.5])
  .text("flower moon")
  .limit(5)
  .toArray();
```

## Reranking

Default reranker: `RRFReranker` (Reciprocal Rank Fusion). Others available via integrations: Cohere, CrossEncoder, ColBERT, Jina, VoyageAI.

```typescript
import { RRFReranker } from "@lancedb/lancedb";
const reranker = new RRFReranker();
// Pass to .rerank(reranker) in search chain
```

Custom rerankers: extend the base `Reranker` class and implement the `rerank` method.
