# Low-Level Design (LLD)

This design maps to **`RAG_Service_Corrected_2026_Diagnostics.py`**. Other Python variants share the same major classes; prefer the diagnostics variant for investigation.

## 1. UML-like class diagram

```mermaid
classDiagram
  direction LR
  class RAGRequest {
    +str query
    +str tenant_id
    +str session_id
    +int top_k
    +int max_tokens
    +__post_init__()
  }
  class RAGResponse {
    +str query
    +str response
    +list sources
    +float latency_ms
    +bool cached
    +str request_id
  }
  class RAGService {
    +encoder
    +client
    +documents
    +cache
    +pii
    +injection
    +monitor
    +float retrieval_min_score
    -_initialize_collection()
    +retrieve(query, tenant_id, top_k)
    +handle(request) RAGResponse
    -_extractive_stub(docs) str
  }
  class SemanticCache {
    +float threshold
    +int ttl
    +int max_size
    -list _entries
    -RLock _lock
    +get(query, tenant_id, session_id, top_k)
    +set(query, tenant_id, response, sources, session_id, top_k)
    +stats(tenant_id)
    -_evict()
  }
  class CacheEntry {
    +str tenant_id
    +str session_id
    +str query
    +ndarray embedding
    +str response
    +list sources
    +int top_k
    +float timestamp
  }
  class PIIFilter {
    +detect(text)
    +redact(text)
    -_luhn(text)
    -_valid_match(name, match)
  }
  class PromptInjectionDetector {
    +check(text)
  }
  class SystemMonitor {
    +int requests
    +int errors
    +int cache_hits
    +int blocked_injections
    +int pii_detections
    +float total_latency_ms
    +record(latency_ms, flags)
    +report(cache)
  }
  class QdrantClient {
    +query_points(...)
    +upsert(...)
  }
  RAGService ..> RAGRequest : consumes
  RAGService ..> RAGResponse : produces
  RAGService *-- SemanticCache
  RAGService *-- PIIFilter
  RAGService *-- PromptInjectionDetector
  RAGService *-- SystemMonitor
  SemanticCache "1" o-- "0..*" CacheEntry
  RAGService --> QdrantClient : uses
```

### Responsibilities

| Unit | Main responsibilities |
|---|---|
| `RAGRequest` | Validates nonempty query (≤ 4000 chars), tenant ID, `top_k` (1–20), `max_tokens` (1–2048); immutable dataclass |
| `RAGResponse` | Contains original query, excerpt response, sources, timing, cache flag, request ID |
| `RAGService` | Wires model, Qdrant, cache, guards and metrics; startup indexing; retrieval; response assembly |
| `PIIFilter` | Detects/replaces a limited set of regex PII patterns; Luhn check for candidate card numbers; IPv4 validation |
| `PromptInjectionDetector` | Checks fixed regex patterns, emits heuristic risk details |
| `SemanticCache` | In-process TTL cache, exact hits then cosine similarity; keyed on tenant, session and `top_k` |
| `CacheEntry` | Stores normalized query vector plus response and key metadata |
| `SystemMonitor` | Tracks request counters, latency and optional system RAM usage |
| `create_app` | Builds FastAPI app and per-tenant authentication dependencies |

## 2. Query processing sequence

```mermaid
sequenceDiagram
  autonumber
  actor Caller
  participant API as FastAPI
  participant Service as RAGService.handle
  participant Injection as PromptInjectionDetector
  participant PII as PIIFilter
  participant Cache as SemanticCache
  participant Embed as SentenceTransformer
  participant Q as Qdrant
  participant Monitor as SystemMonitor
  alt HTTP API mode
    Caller->>API: POST /v1/query + Bearer token
    API->>API: authenticate tenant and validate / rate-limit request
    API->>Service: handle(RAGRequest with derived tenant_id)
  else direct Python invocation
    Caller->>Service: handle(RAGRequest)
  end
  Service->>Injection: check(raw query)
  alt injection heuristic matches
    Injection-->>Service: blocked
    Service->>Service: set fixed refusal-style response
  else query allowed
    Service->>PII: redact(raw query)
    PII-->>Service: sanitized query
    Service->>Cache: get(query, tenant, session, top_k)
    opt same-scope cache records exist, no exact match
      Cache->>Embed: encode(query) for semantic similarity
      Embed-->>Cache: normalized vector
    end
    alt cache hit
      Cache-->>Service: stored response and sources
    else cache miss
      Cache-->>Service: miss
      Service->>Embed: encode(sanitized query)
      Embed-->>Service: normalized query vector
      Service->>Q: query_points(tenant filter, score threshold 0.45)
      Q-->>Service: scored passages
      Service->>Service: form extractive excerpts + source IDs
      Service->>PII: redact(excerpt response)
      PII-->>Service: sanitized response
      Service->>Cache: set(query, tenant, session, top_k, response)
      Cache->>Embed: encode(query) for cache vector
    end
  end
  Service->>Monitor: record latency, cache, PII, block and error flags
  alt HTTP API mode
    Service-->>API: RAGResponse
    API-->>Caller: JSON response
  else direct Python invocation
    Service-->>Caller: RAGResponse
  end
```

### Decision tree / critical path

1. `handle()` starts a timer and random 12-character request ID.
2. `PromptInjectionDetector.check(request.query)` checks the **raw** query. Matched red flags lead to a fixed refusal-style string; no retrieval or cache update happens.
3. Otherwise `PIIFilter.redact()` sanitizes some input patterns and records a warning if PII was detected.
4. `SemanticCache.get(clean_query, tenant_id, session_id, top_k)` checks tenant/session/`top_k`-matched, unexpired records: exact query first, then cosine threshold `>= 0.98` for semantic similarity.
5. On miss, `retrieve()` encodes query and runs `QdrantClient.query_points(...)` with a hard `tenant_id` filter, `score_threshold=0.45`, and `limit=top_k`.
6. `_extractive_stub()` returns a no-match notice or bullet excerpts; output `PIIFilter.redact()` is applied to the newly constructed output.
7. `SemanticCache.set(...)` stores the sanitized response. Metrics are recorded in a `finally` block, including failures.
8. The returned response contains the **original** unsanitized query. This behavior is a privacy caveat, not a recommended design.

## 3. Cache algorithm (implementation-specific)

- In-memory `list[CacheEntry]` protected with `threading.RLock`.
- Deletes expired entries on cache operations using `time.monotonic()`.
- Exact matches avoid re-encoding at lookup; semantic lookup calculates an embedding if any same-scope entries exist.
- Cosine comparison uses `np.dot()` and relies on normalized embedding vectors.
- `set()` always encodes a query to store its vector, even when retrieval already encoded the same query (an optimization opportunity).
- At capacity (1000), removes the oldest remaining entry; this is **not a distributed/LRU cache**.

## 4. Failure semantics

- Invalid `RAGRequest` fields raise `ValueError` when constructed.
- Errors inside `RAGService.handle` are logged and re-raised; the monitor's error count still increases.
- Qdrant/model import and model-download errors surface during initialization of demo/API modes.
- `SystemMonitor.report()` uses an error-rate threshold to set a simple `healthy/degraded` status; that is not a readiness probe for all dependencies.

## 5. Unused data members

`max_tokens` and `SYSTEM_PROMPT` do **not** control any generation or token usage. They are placeholders for future generative RAG work. See [[Future-Roadmap]].
