# Future Roadmap — From Extractive Retrieval to Generative RAG

The following items are **proposals, not currently implemented**. The Mermaid designs in [[System-Architecture]] and [[Low-Level-Design]] represent the present source only.

## Phase 1: Complete a real RAG pipeline

1. Document ingestion from files/URLs with parsers, chunking, metadata and tenant ownership checks.
2. Versioned embedding/index updates, deletion, reindexing and permission-aware retrieval.
3. LLM generation from retrieved context with grounded citations, token budgets and source provenance.
4. Answer-quality and retrieval evaluation, including groundedness, context precision/recall, and failure-case datasets.

## Phase 2: Reliability and safety

- Identity provider integration; tenant and document authorization enforced beyond a single payload filter.
- Eliminate raw query echo or consistently sanitize response fields.
- Better PII detection, red-team evaluations, prompt-injection handling, documented data retention and audit trails.
- Shared semantic cache with invalidation/versioning and a multi-instance-friendly rate limiter.
- Distributed traces, structured metrics, health/readiness checks and alerts.

## Phase 3: Scaling and advanced RAG

- Qdrant server/cluster hosting and backup policies.
- Batch inference or model serving, queueing and capacity tests.
- Hybrid dense + keyword retrieval, reranking and metadata filters.
- Streaming responses; model routing; access-controlled observability.
- Deployment manifests and CI/CD with security and integration gates.

## Concrete next coding task

Add a **new** `GenerativeRAGService` module rather than changing `_extractive_stub` without tests. Define a retriever interface, prompt/context builder, LLM adapter, citation renderer and evaluation suite. The existing tenant-filtered retrieval would become one dependency of a fully generative pipeline.
