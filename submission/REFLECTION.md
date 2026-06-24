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

**Setup story** (≤ 80 chữ): Dùng WSL2 với Ubuntu 24.04, cài CUDA Toolkit 12.x cho RTX 3050 Ti. Build llama-cpp-python với flag `-DGGML_CUDA=on`. Download model Qwen2.5-1.5B-Instruct GGUF (Q4_K_M) từ Hugging Face qua script `download-model.py`.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`. Settings: `n_threads=16`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
| --- | ---: | ---: | ---: | ---: | ---: |
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 1640 | 13718 / 16974 | 5623.5 / 6310.7 | 367928 / 410873 / 423312 | 0.2 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 1347 | 11591 / 25217 | 5018.8 / 10066.5 | 328474 / 658241 / 678010 | 0.2 |

**Một quan sát** (≤ 50 chữ): Q4_K_M có TTFT P50 cao hơn Q2_K ~18% và TPOT cao hơn ~12%, nhưng decode rate bằng nhau (0.2 tok/s). Với GPU 4GB, Q4_K_M cho quality tốt hơn mà không mất thêm throughput đáng kể.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
| ----------: | --------: | ------------: | -----------: | -----------: | -------: |
|          10 |           |               |              |              |          |
|          50 |           |               |              |              |          |

**Batching observation** (từ `record-metrics.py`): peak `llamacpp:n_busy_slots_per_decode` / `requests_processing` ở concurrency 50 = _<…>_, nghĩa là …

_Answer here._

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _<piece you connected — k3d cluster / GCP project / docker-compose / "stub: localhost only">_
- **N17 (Data pipeline):** _<piece — Airflow DAG / batch job / "stub: in-memory dict">_
- **N18 (Lakehouse):** _<piece — Delta Lake table / Iceberg / "stub: SQLite">_
- **N19 (Vector + Feature Store):** _<piece — Qdrant index / Feast / "stub: TOY_DOCS">_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _`<ms>`_
- retrieve: _`<ms>`_
- llama-server: _`<ms>`_

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

_Answer here._

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** _<vd: rebuild llama.cpp với `-DGGML_NATIVE=ON -DGGML_BLAS=ON`; vd: hạ `-t` từ 12 xuống 6; vd: bật Metal trên M2>_

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: <số liệu>
after:  <số liệu>
speedup: ~<X.Y>×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với một bạn cùng lớp đang ngồi cạnh. Tránh "vibes-based" reasoning — bám vào mô hình mental của hardware (memory bandwidth? compute? cache?). Nếu kết quả khác kỳ vọng từ deck, nói rõ — đó là phần grader thưởng điểm._

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

_Answer here._

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [ ] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit — chưa chạy Track 02
- [ ] `benchmarks/bonus-*.md` đã commit — chưa chạy Bonus
- [x] Ít nhất 6 screenshots trong `submission/screenshots/`: đã có `01-hardware-probe.png`
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
