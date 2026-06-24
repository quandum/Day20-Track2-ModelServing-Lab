# 01 — Quickstart Results

Settings: `n_threads=16`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 15030 | 38 / 44 | 11.3 / 11.5 | 747 / 759 / 759 | 88.8 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 7518 | 31 / 40 | 10.9 / 11.5 | 725 / 754 / 766 | 91.4 |

## Observations

- TTFT is the prefill cost. With short prompts this is small; with long prompts it dominates.
- TPOT is per-token decode latency. The decode rate is `1000 / TPOT_p50`.
- The bigger quantization (Q4_K_M) is usually only ~30–60% slower than Q2_K but produces noticeably better text. Q2_K is for *truly* tight RAM.
- `n_threads = physical_cores` is usually best on CPU. Hyperthreading (`logical_cores`) often hurts because the work is bandwidth-bound.

## Before/After: CUDA misconfigured → GPU working

This file was auto-generated after fixing CUDA. The **initial run** (without GPU acceleration) had:
- **Decode rate: 0.2 tok/s** (TPOT ~5.6s, TTFT ~13.7s)
- After rebuild with `-DGGML_CUDA=on`: **88.8 tok/s** (TPOT 11ms, TTFT 38ms)

**Speedup: ~440×** (from GPU offload). See `submission/REFLECTION.md` §2 for full before/after tables.

(Edit this file with your own observations before submitting.)
