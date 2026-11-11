# Testing and Validation

## Built-in offline self-test

```powershell
python RAG_Service_Corrected_2026_Diagnostics.py --self-test
```

The built-in `unittest` suite covers **four test cases**:

1. Regex PII redaction (email, Luhn-valid payment card, IPv4) and invalid card rejection.
2. Known prompt-injection phrase detection and ordinary question non-detection.
3. Cache isolation by tenant, session, and `top_k`, plus source propagation.
4. Request contract validation for blank query and invalid `top_k`.

This self-test **requires NumPy** but does not require downloading the real embedding model. It does not prove Qdrant integration, PII safety or production tenant isolation.

## Additional checks

```powershell
python -m py_compile RAG_Service_Corrected_2026_Diagnostics.py RAG_Service_Corrected_2026_Windows.py rag_fixed_v2.py
python scripts/validate_docs.py
python RAG_Service_Corrected_2026_Diagnostics.py --doctor
```

The docs validator checks that linked wiki pages, Mermaid fences and standalone diagram files exist and that the notebook parses as JSON.

## Integration test plan (not supplied as passing tests)

| Scenario | Expected result |
|---|---|
| Tenant A searches for tenant B-only content | No tenant B payload is returned |
| Wrong token calls `/v1/query` | Request rejected |
| No token | Request rejected |
| Query with invalid `top_k` | Request rejected by validation |
| Two identical questions, same tenant/session/`top_k` | Second call can be a cache hit |
| Same question, different tenant or session | No cross-scope cache hit |
| Cache TTL expiry | Result recomputed |
| Input containing PII | Retrieval query is masked; **raw query still echoed in response** |
| Qdrant unavailable / model load fails | Error surfaces and should be reported correctly |
| High concurrency | Measure latency, throughput and in-process cache synchronization |

## Benchmarking guidance

Record machine CPU/GPU, PyTorch build, embedding model version, dataset size, `top_k`, cache state, P50/P95/P99 latency, throughput (RPS) and retrieval-quality measures. Current `SystemMonitor` only calculates a process-local **average** latency and does not provide benchmark evidence.

## What was validated when this documentation bundle was built

- Python sources: syntax-compiled and offline self-test executed in the packaging environment.
- Notebook: JSON validity checked.
- Wiki pages and Mermaid fence structure: checked by included validation script.
- End-to-end model download/Qdrant API integration: **not asserted** unless separately tested in an environment with all optional dependencies.
