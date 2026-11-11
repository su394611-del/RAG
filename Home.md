# Multi-Tenant RAG Service — Project Wiki

**Python 3.12 · FastAPI · Sentence Transformers · Qdrant · Semantic Cache · Local-first demo**

This wiki documents the actual implementation in the accompanying GitHub repository: a **tenant-filtered retrieval service with extractive answering**. It is a RAG-oriented educational demonstration, **not an end-to-end generative LLM RAG application**. The service **does not call an LLM** to compose new answers.

## Project at a glance

| Capability | Implementation |
|---|---|
| Input | `RAGRequest` or FastAPI `POST /v1/query` |
| Authentication | Per-tenant bearer keys in environment variables (API mode) |
| Preprocessing | Prompt-injection regex checks, then regex-based PII redaction |
| Embeddings | `BAAI/bge-small-en-v1.5`, normalized vectors |
| Retrieval | Qdrant cosine similarity, `tenant_id` payload filter |
| Semantic cache | In-process; tenant, session, and `top_k` scoped; 1-hour TTL |
| Output | KB passage excerpts, source IDs, cache flag, latency, request ID |
| Observability | In-process counters; optional RAM metrics, admin-only API metrics |
| Persistence | In-memory by default, optional local Qdrant path |

## Read the design

- [[Project-Overview]] — capabilities, file map, and realistic boundaries
- [[Getting-Started]] — install and execute the Python CLI, notebook, or API
- [[System-Architecture]] — **high-level design (HLD)**, system and startup diagrams
- [[Low-Level-Design]] — **LLD**, class diagram, request sequence, decision paths
- [[Data-Model-and-Tenant-Isolation]] — data contracts, Qdrant payloads and scoping
- [[API-Reference]] — routes, payloads and PowerShell requests
- [[Source-Code-Walkthrough]] — file-by-file and method-by-method explanation
- [[Configuration-and-Operations]] — runtime flags, configuration and diagnostics
- [[Security-and-Limitations]] — trust boundaries and demo-only security caveats
- [[Testing-and-Validation]] — offline tests, notebook, integration test suggestions
- [[Future-Roadmap]] — optional steps toward generative, production-grade RAG

## Quick run

```powershell
py -3.12 -m venv .venv-rag
.\.venv-rag\Scripts\python.exe -m pip install -r requirements_rag_2026.txt
.\.venv-rag\Scripts\python.exe .\RAG_Service_Corrected_2026_Diagnostics.py --doctor
.\.venv-rag\Scripts\python.exe .\RAG_Service_Corrected_2026_Diagnostics.py --self-test
.\.venv-rag\Scripts\python.exe .\RAG_Service_Corrected_2026_Diagnostics.py --demo
```

> The `--demo` option loads/downloads the embedding model. The `--self-test` option avoids model downloads but still requires NumPy.

## Scope and provenance

This wiki is derived from the uploaded Python source and notebook. The knowledge-base documents are **fictional examples**, the returned answer is **extractive**, and the package is not a validated production deployment.
