# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Trần Mạnh Chánh Quân
**Mã học viên:** 2A202600786
**Cohort:**  _A20-K2_
**Ngày submit:** 2026-06-24

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Output của `python 00-setup/detect-hardware.py`:

- **OS:** Windows 11 (via WSL2 Ubuntu)
- **CPU:** 12th Gen Intel(R) Core(TM) i7-1260P
- **Cores:** 16 physical / 16 logical
- **CPU extensions:** AVX2 ✓ (AVX512 ✗, NEON ✗)
- **RAM:** 15.5 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Ti Laptop GPU, 4096 MiB
- **llama.cpp backend đã chọn:** CUDA
- **Recommended model tier:** Qwen2.5-1.5B-Instruct

**Setup story** (≤ 80 chữ): Dùng WSL2 + Ubuntu 24.04, cài CUDA Toolkit 12.x. Lần đầu build llama-cpp-python không đúng cách (cài prebuilt wheel từ PyPI) → GPU không được dùng → 0.2 tok/s. Fix: rebuild với `CMAKE_ARGS="-DGGML_CUDA=on" pip install --force-reinstall` → GPU hoạt động → 88.8 tok/s.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`. Settings: `n_threads=16`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

### Lần chạy 1 — CUDA misconfigured (CPU-only, 0.2 tok/s)
_llama-cpp-python không được build với CUDA → GPU không được dùng → chạy trên CPU thuần._

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
| --- | ---: | ---: | ---: | ---: | ---: |
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 1640 | 13718 / 16974 | 5623.5 / 6310.7 | 367928 / 410873 / 423312 | 0.2 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 1347 | 11591 / 25217 | 5018.8 / 10066.5 | 328474 / 658241 / 678010 | 0.2 |

### Lần chạy 2 — GPU CUDA hoạt động (88.8 tok/s)
_Sau khi rebuild `llama-cpp-python` với `-DGGML_CUDA=on`. GPU NVIDIA RTX 3050 Ti (4GB) được dùng đầy đủ._

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
| --- | ---: | ---: | ---: | ---: | ---: |
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 15030 | 38 / 44 | 11.3 / 11.5 | 747 / 759 / 759 | 88.8 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 7518 | 31 / 40 | 10.9 / 11.5 | 725 / 754 / 766 | 91.4 |

### So sánh trước-sau (Q4_K_M)

| Metric | CPU-only | GPU CUDA | Speedup |
|--------|--------:|--------:|-------:|
| TTFT P50 | 13,718 ms | **38 ms** | **~360×** |
| TPOT P50 | 5,623 ms | **11 ms** | **~511×** |
| Decode rate | 0.2 tok/s | **88.8 tok/s** | **~440×** |
| E2E P50 | 367,928 ms (~6 phút) | **747 ms** | **~492×** |

**Một quan sát** (≤ 50 chữ): Khi GPU hoạt động đúng, TTFT giảm từ 13.7s → 38ms, decode rate từ 0.2 → 88.8 tok/s. Q4_K_M và Q2_K gần như tương đương trên GPU (11.3 vs 10.9 ms/token) — nên dùng Q4_K_M cho quality tốt hơn.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50.

| Concurrency | # reqs | P50 (ms) | P95 (ms) | P99 (ms) | Failures | Ghi chú |
| ----------: | -----: | -------: | -------: | -------: | -------: | ------- |
|          10 |      4 |   33,000 |   41,000 |   41,000 |        0 | Native server, CPU-only (~1 tok/s) |
|          50 |     11 |   16,000 |   39,000 |   39,000 |        0 | Native server, partial GPU (`-ngl 20`, 4 threads) |

**Batching observation** (từ `record-metrics.py`): peak `busy_slots` = 3.69/4 slots, `deferred` peak = 6 requests, `reqs_proc` duy trì ở 4. Với `--parallel 4` và `--cont-batching`, server xử lý 4 request đồng thời trong 4 slot. Khi tất cả slot bận, request mới bị deferred (xếp hàng). Đây là minh chứng trực quan cho continuous batching: thay vì xử lý tuần tự từng request một, server ghép nhiều request vào cùng một decode step, tăng throughput đáng kể so với static batching.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost — server chạy local trên port 8080, không cần k3d/GCP/docker-compose vì lab tập trung vào model serving, không infrastructure.
- **N17 (Data pipeline):** stub: in-memory dict — `TOY_DOCS` là 5 document hardcoded, không có Airflow/batch job vì dataset nhỏ, mục tiêu chính là test pipeline end-to-end.
- **N18 (Lakehouse):** stub: không dùng — không có Delta Lake/Iceberg/SQLite vì dữ liệu chỉ là 5 document tĩnh.
- **N19 (Vector + Feature Store):** stub: keyword overlap — `retrieve()` dùng keyword matching thay vì embedding + Qdrant. Có thể nâng cấp lên embedding-based search bằng `llama-server` embedding endpoint.

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

| Step | Query 1 | Query 2 | Query 3 |
|------|--------:|--------:|--------:|
| retrieve | 0.1ms | 0.0ms | 0.0ms |
| llama-server | 4,457ms | 482ms | 1,650ms |
| **total** | **4,457ms** | **482ms** | **1,650ms** |

**Reflection** (≤ 60 chữ): llama-server LLM inference chiếm 99.9% latency — đúng kỳ vọng vì model là nút thắt chính. Retrieve bằng keyword gần như miễn phí (0.1ms). Nếu dùng embedding-based search, retrieve có thể tăng lên vài chục ms. Cache prompt prefix (RadixAttention pattern) sẽ giảm LLM latency khi query có chung system prompt.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Rebuild `llama-cpp-python` với `CMAKE_ARGS="-DGGML_CUDA=on"` thay vì dùng prebuilt wheel từ PyPI — chuyển từ CPU-only sang GPU CUDA.

**Before vs after** (Q4_K_M, 64 tokens):

```
before (CPU-only):         TTFT=13,718ms  TPOT=5,623ms  decode=0.2 tok/s
after (GPU CUDA, -ngl 99): TTFT=38ms      TPOT=11ms     decode=88.8 tok/s
speedup: ~440× (decode rate)
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Mô hình transformer 1.5B tham số có ~3 tỷ phép nhân ma trận cho mỗi token. Trên CPU (i7-1260P, 16 cores, ~50 GB/s memory bandwidth), mỗi lần đọc trọng số model từ RAM qua bus DDR4 mất ~30ms — đó là lý do TPOT 5.6 giây. GPU RTX 3050 Ti có 4GB VRAM với băng thông ~192 GB/s và 2560 CUDA cores — trọng số 1.5B model (~1.1GB ở Q4_K_M) nằm gọn trong VRAM, mỗi phép nhân ma trận được xử lý song song trên hàng nghìn cores. Kết quả: prefill (TTFT) từ 13.7s → 38ms. Bài học: với model ≤7B, GPU offload là thay đổi duy nhất có impact lớn nhất — thread tuning chỉ cho speedup ~10-20%.

---

## 6. (Optional) Điều ngạc nhiên nhất

Điều ngạc nhiên nhất là khi GPU hoạt động đúng, Q4_K_M và Q2_K gần như tương đương về tốc độ (11.3 vs 10.9 ms/token) — trên CPU thì Q4_K_M chậm hơn ~12%. Với GPU, quality cao hơn mà không mất throughput, nên không có lý do gì để dùng Q2_K nếu đã có GPU.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-metrics.csv` đã commit (record-metrics.py output)
- [ ] `benchmarks/bonus-*.md` đã commit — chưa chạy Bonus sweep
- [x] Ít nhất 6 screenshots trong `submission/screenshots/`
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
