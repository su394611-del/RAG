# Configuration and Operations

## Environment variables

| Name | Used by | Default / effect | Notes |
|---|---|---|---|
| `EMBED_DEVICE` | `RAGService.__init__` | `cpu` | Set `cuda` only with compatible CUDA PyTorch and GPU |
| `QDRANT_PATH` | `RAGService.__init__` | unset = memory | Set local filesystem path to persist Qdrant data |
| `TENANT_A_API_KEY` | `create_app()` | unset | Optional but at least one tenant key must be set in `--api` |
| `TENANT_B_API_KEY` | `create_app()` | unset | Must differ from tenant A key |
| `RAG_ADMIN_API_KEY` | `create_app()` | unset = no `/v1/metrics` | Must differ from any tenant key |

`.env.example` is for **illustration only**; the app does **not** load a `.env` file automatically. Set values in your terminal environment or implement your own secret-loading mechanism. Never commit real key values or rely on placeholder tokens for actual auth.

## Runtime defaults

| Setting | Default |
|---|---|
| Embedding model | `BAAI/bge-small-en-v1.5` |
| Vector collection | `rag_demo_kb` |
| Retrieval similarity floor | `0.45` |
| Semantic-cache similarity floor | `0.98` |
| Semantic-cache TTL | `3600` seconds |
| Semantic-cache max entries | `1000` |
| API host | `127.0.0.1` |
| API port | `8000` |
| Uvicorn workers | `1` |
| `/v1/query` rate limit | `60/minute` per remote IP |

## Diagnostics sequence

```powershell
python RAG_Service_Corrected_2026_Diagnostics.py --doctor
python RAG_Service_Corrected_2026_Diagnostics.py --self-test
python RAG_Service_Corrected_2026_Diagnostics.py --demo
```

- `--doctor` uses standard-library package metadata checks and optionally imports installed libraries.
- `--self-test` uses NumPy with fake vectors; no Sentence Transformers/Qdrant download is required.
- `--demo` downloads/loads the real model and initializes local Qdrant.

## Operational cautions

- Query and index embeddings are computed on initialization or misses; the current code does not batch requests or implement asynchronous inference.
- `SystemMonitor` metrics are **process-local**, reset on restart, and not a metrics exporter.
- The API's `/health` is a basic status literal, not a deep model/Qdrant readiness check.
- `QDRANT_PATH` enables local disk persistence but is not a cluster or managed service.
- The knowledge-base code **upserts its hard-coded sample documents at each startup**.
- The cache is independent of Qdrant persistence and always resets on process restart.

## Troubleshooting

| Symptom | Check |
|---|---|
| Import failure | `python -m pip check`; selected interpreter/kernel; `--doctor` |
| Model download timeout | Internet/Hugging Face access and local cache |
| CUDA unavailable | `torch.version.cuda`, `torch.cuda.is_available()`, compatible wheel |
| GPU OOM | Use CPU embeddings via `EMBED_DEVICE=cpu` |
| API start fails | Set at least one tenant key, ensure tenant keys distinct |
| No passages returned | Correct tenant and query, relevance score threshold `0.45` |
| Notebook `ipykernel_launcher` argument error | Execute notebook cells, not CLI `main()` |

For durable, scaled or internet-accessible operation, see [[Security-and-Limitations]] and [[Future-Roadmap]].
