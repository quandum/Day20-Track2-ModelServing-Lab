# Day 20 Lab — Numbers Scratchpad

> **This is NOT your graded report.** The graded report is **[`submission/REFLECTION.md`](../submission/REFLECTION.md)** — fill that out for the rubric.
>
> This file is a *scratchpad* for raw numbers as you generate them. Some of the lab scripts (`benchmark.py`, `record-metrics.py`, the bonus sweeps) auto-write derived markdown files under `benchmarks/01-quickstart-results.md`, `benchmarks/02-server-metrics.csv`, `benchmarks/bonus-*.md`. Use this `results.md` to keep loose notes / hand-copied numbers / observations as you work, then transfer the polished version into `submission/REFLECTION.md` at the end.

## Hardware

- Platform: Windows 11 via WSL2 Ubuntu 24.04
- CPU: 12th Gen Intel i7-1260P (16 cores, AVX2)
- RAM (GB): 15.5
- GPU/accelerator: NVIDIA RTX 3050 Ti Laptop GPU, 4096 MiB
- llama.cpp build backend: CUDA (`-DGGML_CUDA=on`)

## Track 01 — Quickstart

Settings: `n_threads=16`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 1640 | 13718 / 16974 | 5623.5 / 6310.7 | 367928 / 410873 / 423312 | 0.2 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 1347 | 11591 / 25217 | 5018.8 / 10066.5 | 328474 / 658241 / 678010 | 0.2 |

**One observation:** Q4_K_M có TTFT P50 cao hơn Q2_K ~18% và TPOT cao hơn ~12%, decode rate bằng nhau (0.2 tok/s). Với GPU 4GB, Q4_K_M tốt hơn về quality mà không mất throughput đáng kể.

## Track 02 — llama-server load test

Run `locust -f 02-llama-cpp-server/load-test.py --headless -u N -r 1 -t 1m` for two values of N.

| Concurrency | RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | | | | | |
| 50 | | | | | |

**Continuous-batching observation:** _peak `llamacpp:n_busy_slots_per_decode` / `requests_processing` from `record-metrics.py` was _ at concurrency 50, which means…_

## Track 03 — Milestone Integration

- N16 piece used:
- N17 piece used:
- N18 piece used:
- N19 piece used:
- One-paragraph reflection on where the latency goes (embed / retrieve / llama-server):

## Bonus — llama.cpp optimization

Pick one or two sweep results to highlight.

### Thread sweep
| threads | tg128 (tok/s) |
|--:|--:|
| | |

### Quant sweep
| quant | size (MB) | tg128 (tok/s) |
|:--|--:|--:|
| | | |

### The one change that mattered most

_Two-to-three sentences. Be specific: what change, what the before/after numbers were, and why you think it worked. The grade weights this paragraph more than the raw numbers._

## Bonus — MLX (macOS only, optional)

| runtime | TTFT P50 (ms) | decode (tok/s) |
|---|--:|--:|
| llama.cpp Metal | | |
| MLX-LM | | |

## Notes / pitfalls / things you'd do differently

(Free-form. The most useful section for the grader.)
