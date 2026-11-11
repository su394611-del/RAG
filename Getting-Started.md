# Getting Started

## 1. Requirements

- Python **3.12** recommended, with `pip` and internet for first-time embedding model download.
- CPU embeddings work; CUDA is optional and requires a compatible CUDA-enabled PyTorch build.
- Source code and `requirements_rag_2026.txt` from the main repository.
- The first real model run needs to download `BAAI/bge-small-en-v1.5` unless already cached.

## 2. Windows PowerShell — copy/paste

Run commands from the folder containing the `.py` files:

```powershell
py -3.12 -m venv .venv-rag
.\.venv-rag\Scripts\python.exe -m pip install --upgrade pip
.\.venv-rag\Scripts\python.exe -m pip install -r requirements_rag_2026.txt
.\.venv-rag\Scripts\python.exe -m pip check

# Check dependencies (no model download)
.\.venv-rag\Scripts\python.exe .\RAG_Service_Corrected_2026_Diagnostics.py --doctor

# Built-in unit tests (no model download, requires NumPy)
.\.venv-rag\Scripts\python.exe .\RAG_Service_Corrected_2026_Diagnostics.py --self-test

# Download model on first run and perform tenant-aware demo retrieval
.\.venv-rag\Scripts\python.exe .\RAG_Service_Corrected_2026_Diagnostics.py --demo
```

Use `.venv-rag\Scripts\python.exe` directly if activation is blocked by PowerShell execution policy.

## 3. Linux / macOS

```bash
python3.12 -m venv .venv-rag
. .venv-rag/bin/activate
python -m pip install -r requirements_rag_2026.txt
python RAG_Service_Corrected_2026_Diagnostics.py --self-test
python RAG_Service_Corrected_2026_Diagnostics.py --demo
```

## 4. Jupyter / VS Code

Open `RAG_Service_Corrected_2026_Diagnostics.ipynb`. Select the Python kernel with the required packages and execute top-to-bottom. The notebook's `RUN_REAL_DEMO` variable is **False** by default; switch it to **True** only if you want to load the real embedding model and run Qdrant queries. Notebook cells are designed to avoid CLI `argparse` conflicts with `ipykernel_launcher.py`.

## 5. Run local FastAPI

Set a **real, private** token for at least one tenant:

```powershell
# Run locally in the same terminal; do NOT commit the generated value
$env:TENANT_A_API_KEY = (& .\.venv-rag\Scripts\python.exe -c "import secrets; print(secrets.token_urlsafe(32))")
.\.venv-rag\Scripts\python.exe .\RAG_Service_Corrected_2026_Diagnostics.py --api
```

For a separate terminal, set `$env:TENANT_A_API_KEY` to the same private value through a secure local mechanism; see [[API-Reference]] for the request example. The server binds to **127.0.0.1:8000**.

## 6. GPU optional

```powershell
.\.venv-rag\Scripts\python.exe -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
$env:EMBED_DEVICE = 'cuda'  # Only if torch.cuda.is_available() is True
```

For constrained or unsupported GPUs use `$env:EMBED_DEVICE = 'cpu'`. Obtain platform-specific PyTorch installation commands from https://pytorch.org/get-started/locally/ rather than assuming a wheel matches your driver.

## 7. Expected example behavior

- Tenant A retrieves passages about example API pricing/rate limits.
- Tenant B retrieves only its own export/security passages.
- Repeating the identical request for tenant A in the same session and `top_k` can hit the in-process cache.
- A prompt containing a known injection pattern is rejected by the heuristic detector.
- The output contains **passage excerpts**, not LLM-authored prose.

Next: [[System-Architecture]] and [[Testing-and-Validation]].
