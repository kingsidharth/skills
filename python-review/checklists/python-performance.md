# Python Performance Checklist

## Data size

- [ ] What is expected row/file/object count?
- [ ] What is the largest input size?
- [ ] Is memory bounded?
- [ ] Is processing chunked?

## Loops

- [ ] Is work repeated inside loops unnecessarily?
- [ ] Are there nested loops over large collections?
- [ ] Can work be batched?
- [ ] Can work be streamed?
- [ ] Can simple transforms use comprehensions/generators?
- [ ] Would vectorized operations be clearer/faster?

## Database

- [ ] Any N+1 queries?
- [ ] Any query inside loop?
- [ ] Any missing index?
- [ ] Are inserts batched?
- [ ] Are transactions too large or too small?

## Network / object storage

- [ ] Per-item API calls where batch API exists?
- [ ] Upload/download retries idempotent?
- [ ] Object keys content-addressed where useful?
- [ ] Any unbounded concurrency?

## Serialization

- [ ] Repeated ORM → dict → Pydantic → dict conversions?
- [ ] Full JSON materialization of large data?
- [ ] Compression/chunking strategy explicit?

## Resource lifecycle

- [ ] HTTP clients reused and closed?
- [ ] DB sessions scoped?
- [ ] Files closed?
- [ ] ML models loaded once per worker, not per item?
