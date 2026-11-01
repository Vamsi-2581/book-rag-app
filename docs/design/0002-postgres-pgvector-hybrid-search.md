# Study note: PostgreSQL hybrid search for book Q&A

**For:** ADR 0002   **Status:** decided in ADR 0002

## Problem

The app answers questions about one book by retrieving relevant passages (chunks) and sending them to a language model. The decision is where and how to store and search those chunks: keep relational data, vectors and keyword search in PostgreSQL, or split search into another service.

## Terms

- **Embedding and vector:** An embedding model maps text to a list of numbers. Similar meanings tend to have nearby vectors, so vector search can find a passage about feeling tired after lunch for a question about getting sleepy after eating, even when the wording differs.
- **Full-text search:** PostgreSQL turns text into searchable words and uses an index to find matching terms. It helps with names, numbers and exact phrases.
- **Hybrid search:** Combining vector and keyword results covers cases each method misses. The project merges their ranked lists with Reciprocal Rank Fusion (RRF), which uses rank positions instead of trying to add incompatible score scales.
- **HNSW:** A graph index for approximate nearest-neighbour search. It trades some recall for speed. `m` controls graph links per vector; `ef_construction` controls search breadth while building the graph.
- **HNSW iterative scan:** In pgvector 0.8+, this lets a filtered search keep walking the graph until enough rows pass the filter or a configured limit is reached.
- **BM25:** A keyword ranking method that weights rarer terms more than common ones.
- **Transaction:** An all-or-nothing database operation. It can prevent a book from becoming visible with only some of its chunks or embeddings saved.
- **Vietnamese word segmentation:** Vietnamese text separates syllables with spaces; a multi-syllable word may look like multiple tokens to a basic full-text parser.

## Requirements

| Measure | Current | 1000x scenario | Basis |
|---|---:|---:|---|
| Books | about 10 | about 10,000 | estimated |
| Chunks per book | about 1,000 | about 1,000 | estimated from a 400-page book |
| Total chunks | about 10,000 | about 10 million | estimated |
| Vector storage | about 40 MB | about 40 GB, plus a similarly sized index | estimated; 1024 dimensions × 4 bytes |
| Search traffic | a few queries/minute | tens of queries/second | assumed |
| Result target | top 8 passages | top 8 passages | assumed; evaluation planned in ADR 0005 |

Today a query is limited to one book and ranks about 1,000 chunks. A future query over selected books or all books is not measured. Other constraints are a free or near-free production tier, one developer, and a local setup with one `docker compose up`.

## Options in depth

### A. PostgreSQL with pgvector and built-in full-text search — chosen

pgvector adds a vector column and distance operators to PostgreSQL. PostgreSQL full-text search stores normalized terms in a `tsvector` and uses a GIN index. One database handles ordinary relational records, chunks, vectors, keyword data and the job queue. A chunk and its metadata can be written in one transaction, and filters such as `book_id` and `page` are SQL predicates.

The trade-offs are shared CPU, memory and connections; fewer vector tuning options than a dedicated engine; and keyword ranking with `ts_rank_cd`, which is simpler than BM25. At current size, running a second service adds cost and consistency work without a measured benefit. PostgreSQL with pgvector is available locally and on the production target, Neon.

### B. PostgreSQL plus a dedicated vector store

Qdrant, Weaviate or Pinecone can offer more recall/speed tuning, quantization and sharding. It also creates two copies of a chunk's data: an interrupted write can leave Postgres and the vector store out of sync, so ingestion needs repair logic. The extra service adds local operations, a network hop, and a bill or hosted-tier limit.

This option is not rejected. Research issue #90 proposes a bounded second `ChunkSearcher` experiment with PostgreSQL kept as the source of truth.

### C. Elasticsearch or OpenSearch for keyword and vector search

A search engine offers mature BM25 and analyzers, but the relational store is still needed. It adds a comparatively heavy service and does not fit the current free-hosting constraint.

### D. An in-process index with SQLite or files

This removes a search server, but a process-local index must be rebuilt or saved after restart, cannot be shared across API instances, and loses transactional writes with relational metadata. It does not fit scaling by adding stateless API containers.

## Measurements

The filtered HNSW behavior was investigated for issue #63 on 2026-10-31 using PostgreSQL 17, pgvector 0.8.7, the migration 0001 schema and synthetic random 1024-dimensional vectors. Test data used 10 books × 1,000 chunks (10,000 total), then 100 books (100,000 total).

- For the app's query, the planner chose the `book_id` index, read the target book's 1,000 rows and sorted by distance. It returned 30 exact results in about 8 ms for 10 books and about 29 ms for 100 books. The first run's cache state was not recorded; the latter was cold and a repeat took about 4 ms. These timings cannot establish a scaling trend.
- When only the HNSW index was available (both `book_id` indexes were dropped inside a rolled-back transaction), a sample query returned 0 rows. Across 20 random queries on 20 books, the average was 0.5 of the 30 requested rows. With default `hnsw.ef_search = 40`, most graph candidates belonged to other books and were discarded by the filter.
- With `hnsw.iterative_scan` set to `strict_order` or `relaxed_order`, each query returned 30 rows.

This is a synthetic failure case, not evidence that production queries currently lose rows: the normal planner selected the exact filtered path. Real embeddings are topic- and language-clustered, so results may differ. Millions of rows and planner choices at that size have not been measured. The local `pgvector/pgvector:pg17` tag follows releases, and the production host chooses its own extension version; check for pgvector 0.8+ before relying on iterative scans.

## At 1000x the load

At roughly 10 million chunks, vectors alone use about 40 GB; the HNSW index is estimated to be of similar size. On a small instance, memory pressure can move graph reads to disk and increase latency. Index builds and inserts also become slower, and retrieval competes with ingestion and jobs for database resources.

Cheaper steps to evaluate first:

1. Add connection pooling and, if needed, a read replica for searches.
2. Partition by book while each query searches one book; selected-book queries need planner pruning to be verified.
3. Reduce vector storage with `halfvec` or fewer dimensions.
4. Compare a dedicated store behind `ChunkSearcher` while PostgreSQL remains the source of truth.

The experiment should compare answer quality, search latency at a defined query rate, and cost on the same load test and eval set. The threshold for adopting another store is not yet set. If queries begin to span all books, the per-book partition step no longer applies.

## Read more

- [pgvector README](https://github.com/pgvector/pgvector), especially HNSW, IVFFlat and filtering
- [PostgreSQL full-text search](https://www.postgresql.org/docs/current/textsearch.html)
- [Malkov and Yashunin, HNSW paper](https://arxiv.org/abs/1603.09320)
- [Cormack, Clarke and Buettcher, Reciprocal Rank Fusion (SIGIR 2009)](https://doi.org/10.1145/1571941.1572114)
