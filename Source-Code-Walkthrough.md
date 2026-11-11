# Source Code Walkthrough

## Recommended entry point

**`RAG_Service_Corrected_2026_Diagnostics.py`** implements the most useful command-line switch set and has an extra `run_doctor()` dependency-diagnostics routine. Start here; the notebook copies its logic into separate teaching cells.

## Read the file top to bottom

| Approximate section | Symbol | Explanation |
|---|---|---|
| Constants / data | `COLLECTION`, `MODEL_ID`, `DOCUMENTS` | Qdrant name, embedding checkpoint, fictional KB |
| Contracts | `RAGRequest`, `RAGResponse` | Parameter validation and output shape |
| Privacy | `PIIFilter` | Regex rules, Luhn validation, IPv4 validation, masking |
| Input checks | `PromptInjectionDetector` | Fixed expression matching and risk-score heuristic |
| Caching | `CacheEntry`, `SemanticCache` | Scope, TTL, exact lookup, embedding similarity, eviction |
| Metrics | `SystemMonitor` | Synchronized counters, elapsed time, optional RAM |
| Retrieval core | `RAGService` | Dependencies, bootstrap collection, query and answer flow |
| HTTP | `create_app(service)` | FastAPI, tenant auth, Pydantic body, SlowAPI, routes |
| Self-test | `_run_self_test()` | Four `unittest` checks; no real model download |
| Diagnostics | `run_doctor()` | Package versions/import checks/GPU availability |
| Bootstrap | `main()` | Parse `--doctor`, `--self-test`, `--demo`, `--api` |

## Embedding and collection startup

`RAGService.__init__` optionally accepts a test encoder, Qdrant client and documents. Without them it loads `SentenceTransformer(MODEL_ID, device=EMBED_DEVICE)`, selects `QdrantClient(":memory:")` unless `QDRANT_PATH` is set, initializes components and calls `_initialize_collection()`.

`_initialize_collection()` embeds each stored passage, creates the collection if missing with cosine distance, and upserts `PointStruct` records whose payload includes tenant ID, category and text. The source contains no incremental ingestion endpoint.

## Retrieval

`retrieve()` encodes a cleaned query, calls `client.query_points` with the tenant filter, similarity threshold and requested result count, and returns dictionaries with `id`, `score` and payload.

`_extractive_stub()` joins passage texts into the response. No prompt is sent to an LLM.

## CLI and notebook

- `--self-test`: regex behavior, false positives for invalid cards, cache scope and request validation.
- `--doctor`: dependency import/version and CUDA checks (no model download).
- `--demo`: built-in sample queries; prints cache state, sources and monitor report.
- `--api`: starts local FastAPI server after embedding model and sample KB initialize.
- Notebook: nearly the same code broken into cells; real model demo opt-in via `RUN_REAL_DEMO`.

## Alternate versions

`RAG_Service_Corrected_2026_Windows.py` and `rag_fixed_v2.py` use a similar structure but do **not** include `--doctor`. Keep them as historical/alternative demos, not as three simultaneously deployed services.

See [[Low-Level-Design]] for algorithms and [[Testing-and-Validation]] for reproducible checks.
