# 0002: PostgreSQL with pgvector and full-text search

**Status:** Accepted   **Date:** 2026-10-31
**Decided by:** agents   **Confidence:** medium
**Review:** [#35](https://github.com/phuchaitt3/book-rag-app/issues/35), open   **Study note:** [docs/design/0002-postgres-pgvector-hybrid-search.md](../design/0002-postgres-pgvector-hybrid-search.md)

## Context

The app retrieves passages from Vietnamese books for question answering. Retrieval needs semantic matching and exact keyword matching. The current corpus is estimated at 10 books and 10,000 chunks; the 1000x scenario is an estimated 10,000 books and 10 million chunks. Searches currently cover one book at a time. The project has one developer, needs a free or near-free production tier, and should run locally with one `docker compose up`. These are estimates, not measured production loads.

## Options considered

- **PostgreSQL with pgvector and full-text search:** one database for relational data, vectors and keyword search; less specialized tuning and search shares database resources.
- **PostgreSQL plus a dedicated vector database:** more vector-search tuning and scaling options; adds a second store that ingestion must keep consistent.
- **Elasticsearch or OpenSearch:** mature BM25 keyword and hybrid search; adds a heavy service while PostgreSQL is still needed for relational data.
- **An in-process index with SQLite or files:** no search server to operate; limits sharing and durable, transactional writes across API instances.

The detailed comparison is in the [study note](../design/0002-postgres-pgvector-hybrid-search.md).

## Decision

Use PostgreSQL with pgvector and PostgreSQL full-text search in one database. The implementation uses HNSW for vector search and GIN for keyword search; the RAG package merges both result lists with Reciprocal Rank Fusion.

## Why

One database keeps chunk text, vectors, keyword data, jobs and relational records together, and supports ordinary SQL filters. The current measured query plan uses the exact book filter and returns the requested candidates; a dedicated vector store would add operational and consistency costs before a measurement shows they are needed. The retrieval interface leaves room to compare a second searcher as a bounded experiment. The supporting measurements and their limits are in the study note.

## Consequences

- One system to run, back up and monitor; retrieval and ingestion share its resources.
- The 1024-dimension schema is tied to bge-m3; changing embedding models requires re-embedding.
- Vietnamese keyword matching is accent-sensitive and does not segment syllables into words; vector search covers some, not all, of this gap.
- The worker currently commits embeddings in batches, so all-or-nothing ingestion is not yet implemented ([#81](https://github.com/phuchaitt3/book-rag-app/issues/81)).
- Filtered HNSW search has a measured failure mode without iterative scans ([#63](https://github.com/phuchaitt3/book-rag-app/issues/63)); multi-book search remains a product question ([#89](https://github.com/phuchaitt3/book-rag-app/issues/89)).
- As the index approaches available memory, measure search latency and memory; try cheaper steps first, then compare a dedicated store behind `ChunkSearcher` using the same load test and eval set.

## Change log

- 2026-11-01 — Shortened this ADR and moved its study material to the linked note; decision unchanged (part of #128).
