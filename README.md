# Multi-Tenant RAG Service — Python, Qdrant, Sentence Transformers and FastAPI

An  multi-tenant **retrieval and extractive-answering** service built with Python, Hugging Face Sentence Transformers, Qdrant, and FastAPI. This repository includes three related Python implementations and a self-contained Jupyter notebook based on the diagnostics implementation.

> **Scope:** This is a *RAG-oriented retrieval demo*, **not a fully generative RAG system**. It retrieves tenant-filtered knowledge-base passages and returns excerpts rather than calling a generative LLM. Its security filters and example organization details are illustrative, not production-grade guarantees.

## Features

- **Tenant-filtered vector retrieval** using Qdrant payload filters.
- **Embeddings:** `BAAI/bge-small-en-v1.5` using `sentence-transformers`.
- **Semantic caching** with cosine similarity, a TTL, and tenant/session separation.
- **Input PII redaction** for illustrative email, phone, credit-card, SSN, and IPv4 patterns.
- **Heuristic prompt-injection detection** with explicitly limited coverage.
- **Monitoring:** request count, errors, latency, cache hits, blocked attempts, and optionally RAM usage.
- **FastAPI interface:** bearer-token authentication, rate limiting, health check, and tenant-specific cache statistics.
- **Offline self-test** (no model download) and dependency diagnostics.
- **Jupyter walkthrough** that avoids command-line argument parsing in notebook kernels.

## Architecture

```text
                +--------------------------+
                |  Client / Notebook       |
                +------------+-------------+
                             |
                    /v1/query or handle()
                             |
                +------------v-------------+
                | Tenant auth (API only)   |
                +------------+-------------+
                             |
                +------------v-------------+
                | Injection checks + PII   |
                +------------+-------------+
                             |
                +------------v-------------+
                | Tenant/session cache     |
                +------------+-------------+
                             | cache miss
                +------------v-------------+
                | BGE-small embeddings     |
                +------------+-------------+
                             |
                +------------v-------------+
                | Qdrant cosine retrieval  |
                | filter: tenant_id        |
                +------------+-------------+
                             |
                +------------v-------------+
                | Extractive KB excerpts   |
                +--------------------------+
```

## Repository contents

| File | Description |
|---|---|
| `RAG_Service_Corrected_2026_Diagnostics.py` | **Recommended CLI entry point:** `--doctor`, `--self-test`, `--demo`, `--api`. |
| `RAG_Service_Corrected_2026_Windows.py` | Windows-oriented variant with dependency checks and the demo/API/self-test modes. |
| `rag_fixed_v2.py` | Earlier core implementation with demo/API/self-test modes. |
| `RAG_Service_Corrected_2026_Diagnostics.ipynb` | Self-contained notebook with code and experiments from the diagnostics version. |
| `requirements.txt` | Dependencies for Python CLI, API, and notebook. |
| `.gitignore` | Excludes secrets, caches, downloaded checkpoints, and local databases. |

The three source `.py` files are preserved from the uploaded implementations; only their filenames are cleaned for GitHub (the `(2)` suffix is removed). Use the **Diagnostics** file for new runs.

## Requirements

- **Python 3.12** recommended (other recent versions may work with compatible wheels).
- Internet access for the first download of the Hugging Face embedding model.
- CPU is sufficient. A CUDA-enabled NVIDIA GPU and CUDA-enabled PyTorch are **optional**.
- FastAPI runs locally on `127.0.0.1:8000` by default. API mode requires environment variables for bearer tokens.

### Windows PowerShell installation

```powershell
cd C:\HF_Projects\1
python -m venv .venv-rag
.\.venv-rag\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip check
```

If your PowerShell execution policy prevents activation, use ` .\.venv-rag\Scripts\python.exe ` instead of activating.

**For NVIDIA GPU use:** install a CUDA-enabled PyTorch wheel matching your hardware and supported Python version from the [official PyTorch instructions](https://pytorch.org/get-started/locally/). `nvidia-smi` showing a CUDA version is *not* proof that the installed `torch` can access CUDA. Verify via:

```powershell
python -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available())"
```

If CUDA is available, choose GPU embeddings for the current PowerShell session:

```powershell
$env:EMBED_DEVICE = 'cuda'
```

Otherwise omit the variable (defaults to CPU), or use `$env:EMBED_DEVICE = 'cpu'`.

## Run the Python CLI

```powershell
# Recommended: print package/import/GPU diagnostics (no model download)
python .\RAG_Service_Corrected_2026_Diagnostics.py --doctor

# Offline tests of filtering, cache isolation, and request validation
python .\RAG_Service_Corrected_2026_Diagnostics.py --self-test

# Download/load BGE embeddings, initialize Qdrant, run five demonstration queries
python .\RAG_Service_Corrected_2026_Diagnostics.py --demo
```

The demo includes two tenants (`tenant_a`, `tenant_b`) with separate sample knowledge-base documents. Repeating the same question for one tenant demonstrates caching; the other tenant cannot retrieve the first tenant's passages through the configured tenant filter.

### Alternative source files

```powershell
python .\RAG_Service_Corrected_2026_Windows.py --self-test
python .\RAG_Service_Corrected_2026_Windows.py --demo
python .\rag_fixed_v2.py --self-test
```

## Jupyter Notebook / VS Code

1. Select the Python environment where `requirements.txt` was installed.
2. Open `RAG_Service_Corrected_2026_Diagnostics.ipynb` in JupyterLab or VS Code.
3. Run cells in order. The notebook's self-test runs without downloading Hugging Face models.
4. Set `RUN_REAL_DEMO = True` in the final experiment cell to download embeddings and run real vector retrieval.

Notebook code **does not run CLI `argparse` against `ipykernel_launcher.py`**. This avoids the common `--f=...kernel.json` argument parsing error.

```powershell
python -m jupyter lab
```

## FastAPI server

**Never commit API keys to GitHub.** Create separate, sufficiently random tokens and keep them in environment variables, not source files. In PowerShell:

```powershell
# Generate keys (copy the resulting values to your own private secure store)
python -c "import secrets; print('Tenant A:', secrets.token_urlsafe(32)); print('Tenant B:', secrets.token_urlsafe(32))"

# Set your private values locally (do not paste real keys into GitHub)
$env:TENANT_A_API_KEY = 'REPLACE_WITH_YOUR_PRIVATE_TENANT_A_KEY'
$env:TENANT_B_API_KEY = 'REPLACE_WITH_YOUR_PRIVATE_TENANT_B_KEY'

python .\RAG_Service_Corrected_2026_Diagnostics.py --api
```

From a second PowerShell window with the appropriate environment variable set:

```powershell
$headers = @{ Authorization = 'Bearer ' + $env:TENANT_A_API_KEY }
$body = @{ query = 'What are the API rate limits on Pro?'; top_k = 3 } | ConvertTo-Json
Invoke-RestMethod -Uri 'http://127.0.0.1:8000/v1/query' -Method Post `
  -Headers $headers -ContentType 'application/json' -Body $body
```

API routes:

| Method | Path | Access | Purpose |
|---|---|---|---|
| GET | `/health` | Public | Basic status only |
| POST | `/v1/query` | Tenant bearer token | Tenant-filtered retrieval + excerpts |
| GET | `/v1/cache/stats` | Tenant bearer token | That tenant's cache statistics |
| GET | `/v1/metrics` | Optional admin bearer token | Metrics, only when `RAG_ADMIN_API_KEY` is set |

Optional local persistent Qdrant storage can be enabled via `QDRANT_PATH`. Without it, the collection is in memory and recreated each run.

## Educational datasets and scope

The built-in documents are **fictional examples** (e.g., subscription rates, export limits, and security statements). They are not verified facts about any actual company. The reported metrics are local demo statistics, not production benchmarks.

- The `SYSTEM_PROMPT` and `max_tokens` attributes are reserved; neither causes LLM generation in this implementation.
- Regex PII rules are incomplete and US-oriented, and can miss sensitive information or generate false positives.
- Prompt-injection checks rely on fixed patterns and are not a comprehensive defense.
- `SemanticCache` stores entries in process memory and is separated by `tenant_id`, `session_id`, and `top_k`; it is not a distributed cache.
- `QdrantClient(':memory:')` is suited to experiments, not durable multi-instance deployments.
- Rate limiting is based on client IP in this demonstration, not per-tenant billing quotas.
- No production security review, red-team assessment, or scale/load testing is included.
- API mode is configured for loopback access; use appropriate infrastructure/security review before any exposure to the internet.

## Troubleshooting

| Problem | Recommended check |
|---|---|
| `ModuleNotFoundError` | Confirm the terminal and notebook use the environment where you installed `requirements.txt` |
| Hugging Face download error | Check internet connectivity and whether `BAAI/bge-small-en-v1.5` is already cached |
| CUDA unavailable | Check `torch.version.cuda` and `torch.cuda.is_available()`; CPU is an acceptable fallback |
| Out-of-memory on a 4 GB GPU | Use CPU embeddings: `$env:EMBED_DEVICE = 'cpu'` |
| API reports no keys configured | Set `TENANT_A_API_KEY` or `TENANT_B_API_KEY` before `--api` |
| Jupyter throws `ipykernel_launcher` argument error | Run the supplied `.ipynb` or use `python file.py --demo` in a terminal, not raw `main()` in a notebook |

## References

- [Hugging Face Sentence Transformers](https://www.sbert.net/)
- [BAAI/bge-small-en-v1.5](https://huggingface.co/BAAI/bge-small-en-v1.5)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [PyTorch installation guide](https://pytorch.org/get-started/locally/)

## License

No open-source license is specified in the supplied source. Add a `LICENSE` file only after choosing the terms under which you want others to use or redistribute this repository.
