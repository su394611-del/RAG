# System Architecture (High-Level Design)

## HLD: application and dependency boundaries

The application runs as **one local Python process**. A client can call `RAGService.handle` directly (CLI/notebook) or use the optional local FastAPI layer. Qdrant is constructed as either an in-memory instance or a local persisted store. `SemanticCache` and `SystemMonitor` reside in process memory.

```mermaid
flowchart TB
  client["API client / local demo / notebook"]
  subgraph HTTP["FastAPI boundary - API mode only"]
    endpoint["POST /v1/query"]
    auth["HTTPBearer: token mapped to tenant_id"]
    limits["SlowAPI: 60 requests/min per client IP"]
  end
  subgraph Runtime["Python process: RAGService"]
    guard["PromptInjectionDetector: heuristic blocking"]
    pii["PIIFilter: redact query"]
    cache["SemanticCache: tenant + session + top_k / TTL"]
    embed["SentenceTransformer: BAAI/bge-small-en-v1.5"]
    retrieve["retrieve: query_points with tenant_id filter"]
    extract["Extractive excerpt response + output redaction"]
    metrics["SystemMonitor: counters, latency, RAM"]
  end
  subgraph Storage["Local demo resources"]
    qdrant[("Qdrant local or in-memory / cosine vectors")]
    kb["4 fictional tenant-tagged KB documents"]
  end
  client --> endpoint
  endpoint --> auth --> limits --> guard
  guard -->|"allowed"| pii --> cache
  cache -->|"miss"| embed --> retrieve --> qdrant
  qdrant --> retrieve --> extract --> cache
  cache -->|"hit or stored result"| response["RAGResponse: excerpts, sources, latency, cached, request_id"]
  guard -->|"blocked"| response
  metrics -.-> response
  guard -.-> metrics
  cache -.-> metrics
  kb -->|"embedded and upserted at startup"| qdrant
```

### Diagram reading notes

- HTTP bearer authentication and IP rate limiting only exist for the **`--api`** path; the Python CLI invokes the service directly.
- API authentication derives `tenant_id` from the bearer key; the request body does not select a tenant.
- The embedding model is shared for all tenants; tenant isolation is implemented via **Qdrant payload filtering**, plus tenant-scoped cache keys.
- `--demo` initializes its own in-memory collection by default.
- Source KB content is fictional and inserted at application initialization, not via a file ingestion pipeline.

## Startup design: KB indexing

```mermaid
sequenceDiagram
  autonumber
  participant CLI as CLI (--demo / --api)
  participant S as RAGService
  participant E as SentenceTransformer
  participant Q as QdrantClient
  participant KB as DOCUMENTS list
  CLI->>S: RAGService()
  S->>E: load BAAI/bge-small-en-v1.5 (EMBED_DEVICE)
  S->>Q: open in-memory or QDRANT_PATH store
  S->>S: initialize guards, cache, monitor
  S->>KB: read 4 fictional sample documents
  loop for each document
    S->>E: encode(document.text, normalize=True)
    E-->>S: vector
  end
  S->>Q: collection_exists(rag_demo_kb)
  alt collection absent
    S->>Q: create_collection(COSINE, inferred vector size)
  end
  S->>Q: upsert(points with tenant_id, category, text)
  Q-->>S: stored
  S-->>CLI: service ready
```

### Storage and indexing assumptions

- Collection name: `rag_demo_kb`.
- Distance metric: `COSINE`.
- Vector length is taken from the encoder's first output; there is no independent schema migration.
- Each Qdrant point has an integer `id` and payload fields `tenant_id`, `category`, and `text`.
- If an existing collection has a changed vector dimension or mismatched schema, the program does not include migration logic.

## Architectural quality attributes (what is demonstrated)

| Concern | Current implementation | Engineering implication |
|---|---|---|
| Security | Environment-token auth + regex guards | Demonstration only; not a production authorization solution |
| Isolation | Query payload filter and cache segmentation | Requires adversarial testing and defense in depth before deployment |
| Scalability | Single process, local model, local Qdrant | Model replication, concurrency and scaling are not implemented |
| Availability | Local service; no clustering | Restart loses in-memory cache and default vector store |
| Observability | In-memory counters, optional `psutil` | Not Prometheus/OpenTelemetry export |
| Privacy | Some regex masking | Original query echoed in output; sensitive metadata handling not complete |

## Deployment views

**Default:** client → `127.0.0.1:8000` → FastAPI + `RAGService` + model + in-memory Qdrant (single process). **Optional:** `QDRANT_PATH` persists local storage between restarts; it does not make Qdrant a network cluster. **Production target:** not provided by the uploaded source.

See [[Low-Level-Design]], [[Configuration-and-Operations]], [[Security-and-Limitations]].
