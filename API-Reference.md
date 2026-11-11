# FastAPI API Reference

The server runs with `--api`, listens on **`127.0.0.1:8000`**, and needs at least one environment-backed tenant bearer key. API docs are also available locally at `/docs` and `/openapi.json` if the server is running.

## Routes

| Method | Route | Authentication | Summary |
|---|---|---|---|
| `GET` | `/health` | None | Returns `{"status":"ok"}` only |
| `POST` | `/v1/query` | Tenant bearer token | Tenant-filtered excerpt retrieval |
| `GET` | `/v1/cache/stats` | Tenant bearer token | Entries in requesting tenant's cache scope |
| `GET` | `/v1/metrics` | Admin bearer token, **only if `RAG_ADMIN_API_KEY` is set** | Overall in-memory monitor report |

`/v1/query` uses **SlowAPI `60/minute` per remote IP**, not per-tenant accounting. The other routes do not have an explicit decorator-based limit in the uploaded code.

## Query request body

```json
{
  "query": "What are the Pro API rate limits?",
  "session_id": "demo-session-1",
  "top_k": 3,
  "max_tokens": 300
}
```

| Field | Required? | Validation |
|---|---|---|
| `query` | yes | Nonempty, max 4000 chars |
| `session_id` | no | String or null, max 128 chars |
| `top_k` | no | Integer 1–20, default 3 |
| `max_tokens` | no | Integer 1–2048, default 300; currently **unused** |

`tenant_id` must **not** be provided in the request body. The backend derives it from the API key.

## PowerShell request

In one terminal run `--api` with the tenant's private key. In another terminal, securely supply the **same** value, then:

```powershell
$headers = @{ Authorization = "Bearer $env:TENANT_A_API_KEY" }
$payload = @{
  query = 'What are the API rate limits on Pro?'
  session_id = 'demo-session-1'
  top_k = 3
} | ConvertTo-Json
Invoke-RestMethod -Uri 'http://127.0.0.1:8000/v1/query' -Method Post `
  -Headers $headers -ContentType 'application/json' -Body $payload
```

> **Privacy:** The response's `query` field echoes the original question verbatim, even after internal query PII redaction. Do not send personal data to this demo and do not expose the API publicly without a security review.

## Typical error handling

- Missing bearer token: rejected by FastAPI `HTTPBearer`.
- Wrong token: `401` with `Invalid API key`.
- Invalid body fields: FastAPI/Pydantic request-validation error (typically `422`).
- Rate limit exceeded: `429` from SlowAPI.
- No tenant keys configured at app construction: process raises `RuntimeError` before serving.

No API endpoint for document ingestion, deletion, tenant provisioning or generated answers exists in this code.
