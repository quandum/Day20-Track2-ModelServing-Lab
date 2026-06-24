# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Trần Mạnh Chánh Quân
**Mã học viên:** 2A202600786
**Cohort:**  _A20-K2_
**Ngày submit:** 2026-06-25

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Output của `python 00-setup/detect-hardware.py`:

- **OS:** Windows 11 (via WSL2 Ubuntu 24.04)
- **CPU:** 12th Gen Intel(R) Core(TM) i7-1260P
- **Cores:** 16 physical / 16 logical
- **CPU extensions:** AVX2 ✓ (AVX512 ✗, NEON ✗)
- **RAM:** 15.5 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Ti Laptop GPU, 4096 MiB
- **llama.cpp backend đã chọn:** CUDA
- **Recommended model tier:** Qwen2.5-1.5B-Instruct

**Setup story:** Dùng WSL2 + Ubuntu 24.04, CUDA Toolkit 12.4. Laptop có GPU RTX 3050 Ti (4GB VRAM) nhưng RAM giới hạn 15.5GB và adapter nguồn laptop dễ quá tải khi cả CPU+GPU chạy max đồng thời. Ban đầu `llama-cpp-python` từ PyPI không có CUDA (prebuilt wheel) → decode 0.2 tok/s. Sau khi rebuild với `-DGGML_CUDA=on` → 88.8 tok/s. Native build llama.cpp từ source với CUDA cho phép chạy benchmark và quant sweep với `/metrics` đầy đủ.

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

**Một quan sát:** Khi GPU hoạt động đúng, TTFT giảm từ 13.7s → 38ms, decode rate từ 0.2 → 88.8 tok/s — speedup ~440×. Q4_K_M và Q2_K gần như tương đương trên GPU (11.3 vs 10.9 ms/token) → nên dùng Q4_K_M cho quality tốt hơn. Trên CPU, Q4_K_M chậm hơn Q2_K ~12% do model lớn hơn → memory bandwidth là bottleneck chính.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50. Server sử dụng: native `llama-server` build từ source với CUDA.

| Concurrency | # reqs | P50 (ms) | P95 (ms) | P99 (ms) | Failures | Cấu hình server |
| ----------: | -----: | -------: | -------: | -------: | -------: | ------- |
|          10 |      4 |   33,000 |   41,000 |   41,000 |        0 | CPU-only, `-t 16`, `-ngl 0` (~1 tok/s) |
|          50 |     11 |   16,000 |   39,000 |   39,000 |        0 | GPU partial, `-t 4`, `-ngl 20` (~30 tok/s) |

**Phân tích:** Load-10 với CPU-only chỉ hoàn thành 4 requests trong 60s vì decode rate ~1 tok/s, mỗi request cần 60-80s. Load-50 với GPU partial offload hoàn thành 11 requests — số lượng thấp hơn kỳ vọng vì GPU bị giới hạn ở `-ngl 20` (chỉ 20/28 layers lên GPU) để tránh quá tải nguồn laptop. P95 ở mức 39-41s cho thấy tail latency bị chi phối bởi các request phải xếp hàng khi tất cả 4 slot đều bận.

**Batching observation** (từ `record-metrics.py` chạy đồng thời với `make load-10`):

```
t=1782322837  reqs_proc=0  deferred=0  busy_slots=3.75  tok_pred=2642
t=1782322850  reqs_proc=4  deferred=3  busy_slots=3.49  tok_pred=3121
t=1782322870  reqs_proc=4  deferred=6  busy_slots=3.62  tok_pred=4447
t=1782322896  reqs_proc=4  deferred=6  busy_slots=3.69  tok_pred=5668
```

- `busy_slots` duy trì ở 3.4–3.7/4 slots → gần như toàn bộ capacity được dùng
- `deferred` tăng dần từ 0→6 → request phải xếp hàng khi hết slot trống
- `reqs_proc` = 4 → 4 request được xử lý đồng thời (đúng `--parallel 4`)
- `tok_pred` tăng từ 2,642 → 5,668 → server liên tục sinh token

**Kết luận:** Đây là minh chứng trực quan cho **continuous batching**: thay vì xử lý tuần tự từng request một (static batching), server ghép nhiều request vào cùng một decode step trên nhiều slot song song. Khi slot đầy, request mới bị deferred. So với static batching (phải đợi cả batch hoàn thành mới nhận batch mới), continuous batching cho phép slot được tái sử dụng ngay khi một request hoàn thành → throughput cao hơn đáng kể dưới concurrent load.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost — server chạy local trên `http://localhost:8080`, không cần k3d/GCP/docker-compose vì lab tập trung vào model serving, không infrastructure orchestration.
- **N17 (Data pipeline):** stub: in-memory dict — `TOY_DOCS` là 5 document hardcoded trong `pipeline.py`. Không có Airflow DAG hay batch job vì dataset chỉ 5 documents tĩnh, mục tiêu chính là kiểm tra flow embed→retrieve→generate.
- **N18 (Lakehouse):** stub: không dùng — dữ liệu quá nhỏ (5 documents) nên không cần Delta Lake/Iceberg/SQLite. Trong production, 5 documents này sẽ nằm trong một Delta table với versioning.
- **N19 (Vector + Feature Store):** stub: keyword overlap — `retrieve()` dùng keyword matching (đếm từ chung) thay vì embedding + vector search. Có thể nâng cấp lên embedding-based search bằng `llama-server` embedding endpoint (`/v1/embeddings`) và lưu vector vào Qdrant/ChromaDB.

**Pipeline latency breakdown** (đo bằng `time.perf_counter()`):

| Step | Query 1 (goodput) | Query 2 (PagedAttn) | Query 3 (disagg) |
|------|--------:|--------:|--------:|
| retrieve (keyword) | 0.1ms | 0.0ms | 0.0ms |
| llama-server LLM | 4,457ms | 482ms | 1,650ms |
| **total** | **4,457ms** | **482ms** | **1,650ms** |

**Reflection:** llama-server LLM inference chiếm 99.9% total latency — đúng kỳ vọng vì model generation là nút thắt chính. Retrieve bằng keyword gần như miễn phí (< 1ms) vì chỉ duyệt 5 documents. Query 1 có latency cao nhất (4.5s) do câu trả lời dài nhất (~150 tokens). Nếu dùng embedding-based search với数千 documents thật, retrieve có thể tăng lên 50-200ms. Một optimization tiềm năng: cache prompt prefix dùng RadixAttention pattern (tương tự `/metrics` prompt cache) — system prompt giống nhau cho mọi query → có thể skip prefill hoàn toàn cho system tokens.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Build llama.cpp từ source với `-DGGML_CUDA=ON` và chọn quantization phù hợp. Native binary build từ source cho phép: (1) dùng `llama-server` với `/metrics` endpoint để quan sát continuous batching, (2) chạy `llama-bench` để sweep quantization, (3) kiểm soát chính xác `-ngl` (GPU layer offload) để cân bằng performance và power.

**Before vs after** (Q4_K_M, 64 tokens):

```
before (Python server, CPU-only):        TTFT=13,718ms  TPOT=5,623ms  decode=0.2 tok/s
after (Native build, GPU CUDA, -ngl 99): TTFT=38ms      TPOT=11ms     decode=88.8 tok/s
speedup: ~440× (decode rate), ~360× (TTFT)
```

**Quant sweep trên GPU** (native `llama-bench`, `-ngl 99`, `-t 4`):

| quant | size (MB) | tg64 (tok/s) | VRAM fit? |
|:--|--:|--:|:---:|
| Q2_K | 718.0 | 106.0 | ✅ (dư ~3.3GB) |
| Q4_K_M | 1065.6 | 105.5 | ✅ (dư ~3.0GB) |
| Q5_K_M | 1225.9 | 87.2 | ✅ (dư ~2.8GB) |
| Q6_K | 1396.3 | 76.6 | ✅ (dư ~2.6GB) |
| Q8_0 | 1806.8 | 72.8 | ✅ (dư ~2.2GB) |

**Tại sao nó work** (phần grader đọc kỹ nhất):

Mô hình transformer 1.5B tham số thực hiện ~3 tỷ phép nhân ma trận cho mỗi token sinh ra. Trên CPU (i7-1260P, 16 cores, DDR4 ~50 GB/s memory bandwidth), mỗi lần đọc toàn bộ trọng số model từ RAM mất ~20-30ms cho model Q4_K_M (1.07GB) — đó là lý do TPOT lên tới 5.6 giây cho 64 tokens. GPU RTX 3050 Ti có 4GB VRAM với băng thông ~192 GB/s (gấp ~4× DDR4) và 2,560 CUDA cores chạy song song — toàn bộ trọng số model Q4_K_M (~1.1GB) nằm gọn trong VRAM, mỗi phép nhân ma trận được vectorized trên hàng trăm cores đồng thời. Kết quả: prefill (TTFT) từ 13.7s → 38ms, decode từ 0.2 → 88.8 tok/s.

Điều thú vị nhất từ quant sweep: **Q2_K và Q4_K_M có tốc độ gần như giống hệt nhau** (106.0 vs 105.5 tok/s) trên GPU — vì cả hai đều fit hoàn toàn trong VRAM, bottleneck không phải memory bandwidth mà là compute throughput của GPU. Q4_K_M lớn hơn 48% nhưng chỉ chậm hơn 0.5% — một sự đánh đổi cực kỳ có lợi. Từ Q5_K_M trở lên, tốc độ giảm rõ rệt (87→77→73 tok/s) do kích thước model tăng → mỗi layer cần đọc nhiều bytes hơn từ VRAM → memory bandwidth bắt đầu thành bottleneck. Q8_0 (1.8GB) chậm hơn Q2_K (0.7GB) tới 31% dù cùng chạy trên GPU — minh chứng rằng ngay cả trên GPU, model size vẫn ảnh hưởng đến throughput thông qua memory traffic.

**Bài học:** Với laptop có GPU ≤4GB VRAM, Q4_K_M là sweet spot tuyệt đối — quality production-grade mà không mất throughput. Build từ source với CUDA cho phép kiểm soát hoàn toàn pipeline (native server `/metrics`, `llama-bench`, embedding server), điều mà prebuilt wheel không làm được. Thread tuning trên CPU chỉ cho speedup ~10-20%, trong khi GPU offload cho speedup ~440× — đúng như những gì slide deck nói về accelerator offload là yếu tố quyết định trong model serving.

---

## 6. Problems encountered & solutions

> **Ghi lại tất cả vấn đề đã gặp và cách xử lý.** Đây là phần thể hiện quá trình debugging thực tế.

### Vấn đề 1: `llama-cpp-python` prebuilt wheel không có CUDA
- **Triệu chứng:** `benchmark.py` báo decode rate 0.2 tok/s, server log hiện `load_tensors: layer X assigned to device CPU`.
- **Nguyên nhân:** `pip install llama-cpp-python` cài prebuilt wheel từ PyPI — wheel này không được build với `-DGGML_CUDA=ON`.
- **Cách xử lý:** Rebuild từ source: `CMAKE_ARGS="-DGGML_CUDA=on" pip install --force-reinstall --no-cache-dir llama-cpp-python`. Sau rebuild, `llama_supports_gpu_offload()` trả về `True`, benchmark đạt 88.8 tok/s.
- **Bài học:** Luôn kiểm tra `llama_supports_gpu_offload()` trước khi chạy benchmark. Prebuilt wheel trên PyPI thường là CPU-only.

### Vấn đề 2: Build llama.cpp từ source bị OOM (Out of Memory)
- **Triệu chứng:** `cmake --build build -j --config Release` báo lỗi `error: ggml.h: Cannot allocate memory`.
- **Nguyên nhân:** `-j` không giới hạn = build 16 jobs song song (trên CPU 16 cores). Mỗi file `.cu` (CUDA kernel) cần ~1-2GB RAM để compile → 16 jobs cần >16GB, vượt 15.5GB RAM của laptop.
- **Cách xử lý:** Giới hạn số job song song: `-j4` (ổn), rồi `-j2` (an toàn hơn), cuối cùng `-j1` (đảm bảo nhất). Build `-j1` thành công sau ~10 phút.
- **Bài học:** Trên laptop RAM hạn chế, LUÔN dùng `-j2` hoặc `-j1` khi build CUDA code. CUDA compilation tốn RAM hơn C++ thông thường rất nhiều.

### Vấn đề 3: Build artifact bị corrupt sau OOM
- **Triệu chứng:** Sau lần OOM, build lại báo `seed-oss.cpp.o: file format not recognized` → `ld returned 1 exit status`.
- **Nguyên nhân:** File object `.o` bị ghi dở dang khi OOM kill compiler giữa chừng. CMake không tự detect file corrupt.
- **Cách xử lý:** `rm -rf build && cmake -B build ...` — xóa sạch thư mục build và build lại từ đầu.
- **Bài học:** Sau OOM, luôn clean build (`rm -rf build`), không dùng `cmake --build` incremental.

### Vấn đề 4: Native server gây sập nguồn laptop
- **Triệu chứng:** Chạy `make serve-native` (native `llama-server`, `-ngl 99`, `-t 16`) → laptop đột ngột tắt nguồn sau vài giây.
- **Nguyên nhân:** GPU (RTX 3050 Ti, 30W TDP) + CPU (i7-1260P, 28W TDP) cùng chạy 100% → tổng công suất >65W adapter nguồn laptop → overcurrent protection ngắt nguồn.
- **Cách xử lý:** Giới hạn cả CPU và GPU: `LAB_N_THREADS=4 LAB_N_GPU_LAYERS=20 make serve-native`. Chỉ 4 CPU threads + 20/28 GPU layers → tổng công suất ~35W, an toàn. Vẫn đạt ~30 tok/s — đủ cho load test.
- **Bài học:** Trên laptop gaming, NVIDIA GPU và Intel CPU chia sẻ chung thermal/power budget. Cần giảm đồng thời cả `-t` (CPU threads) và `-ngl` (GPU layers) để tránh quá tải adapter nguồn. Không thể chạy cả hai ở 100%.

### Vấn đề 5: Python server (`make serve`) không dùng GPU
- **Triệu chứng:** `time curl` mất 19.8s cho 28 tokens (1.4 tok/s), log hiện `load_tensors: layer X assigned to device CPU`.
- **Nguyên nhân:** Mặc dù `benchmark.py` (cùng môi trường) chạy GPU được, nhưng `python -m llama_cpp.server` dường như load model khác cách — có thể do `llama-cpp-python` server không tự động detect CUDA khi launch qua `python -m`.
- **Cách xử lý:** Chuyển sang dùng **native server** (`make serve-native`) cho tất cả Track 02 và Track 03. Native server build từ source với `-DGGML_CUDA=ON` luôn dùng CUDA khi có `-ngl > 0`.
- **Bài học:** `llama-cpp-python` server không đáng tin cậy bằng native `llama-server` cho GPU workloads. Native binary cho phép kiểm soát chính xác từng flag (`-ngl`, `-t`, `--parallel`, `--cont-batching`, `--metrics`).

### Vấn đề 6: Locust load-50 trả về 0 requests
- **Triệu chứng:** Chạy `make load-50` với CPU-only server → 0 requests completed, bảng percentiles trống.
- **Nguyên nhân:** CPU-only decode rate ~1 tok/s × 80 tokens/request = ~80s/request. Locust timeout mặc định 120s, nhưng test chỉ chạy 60s (`-t 1m`) → không request nào kịp hoàn thành trước khi test dừng.
- **Cách xử lý:** (1) Chuyển sang native server với GPU partial offload (`-ngl 20`) → decode ~30 tok/s → mỗi request ~3s; (2) Luôn xác nhận server hoạt động bằng `curl` trước khi chạy locust; (3) Đảm bảo locust `--host` khớp với server address.
- **Bài học:** CPU-only inference quá chậm cho load testing với nhiều users. Cần ít nhất partial GPU offload. Nếu bắt buộc dùng CPU, phải tăng `-t` (test duration) hoặc giảm `max_tokens`.

### Vấn đề 7: Quant sweep trả về 0.0 tok/s
- **Triệu chứng:** `make sweep-quant` chạy tất cả quantization nhưng `tg128` luôn = 0.0.
- **Nguyên nhân (2 lỗi cùng lúc):**
  1. Regex trong `quant-sweep.py` tìm `tg128` nhưng `llama-bench` với `-n 64` xuất test name là `tg64`. → regex không match → mặc định 0.0.
  2. Script đọc `threads` từ `hardware.json` (16) thay vì từ env var `LAB_N_THREADS`. 16 threads gây lỗi benchmark (GPU overload).
- **Cách xử lý:** Sửa `quant-sweep.py`: (a) đổi regex từ `tg128` thành `tg[0-9]+` để match mọi test name; (b) thêm `os.environ.get("LAB_N_THREADS", ...)` để nhận env var. Sau khi sửa, sweep chạy đúng với `LAB_N_THREADS=4` → kết quả 72.8–106.0 tok/s.
- **Bài học:** Luôn kiểm tra raw output của subprocess trước khi viết regex. Hardcode test name trong regex rất dễ vỡ khi thay đổi tham số. Và luôn cho phép override bằng environment variable.

### Vấn đề 8: Build script không tự động detect CUDA flags
- **Triệu chứng:** `make build-llama` build thành công nhưng không có CUDA support (chỉ `-DGGML_NATIVE=ON`).
- **Nguyên nhân:** Makefile dùng `$(LLAMA_CMAKE_FLAGS)` — biến này rỗng nếu không được set trong `.env` hoặc environment. `hardware.json` có `"llama_cpp_cmake_flag": "-DGGML_CUDA=on"` nhưng Makefile không tự động đọc.
- **Cách xử lý:** Build thủ công: `cmake -B build -DGGML_CUDA=ON -DGGML_NATIVE=ON` thay vì dùng `make build-llama`. Hoặc thêm `LLAMA_CMAKE_FLAGS="-DGGML_CUDA=ON"` vào `.env`.
- **Bài học:** Không nên mù quáng tin vào Makefile targets — luôn kiểm tra flag thực tế được truyền vào cmake.

---

## 7. Điều ngạc nhiên nhất

Có hai điều thực sự bất ngờ:

1. **Q2_K và Q4_K_M có tốc độ giống hệt nhau trên GPU** (106.0 vs 105.5 tok/s) dù Q4_K_M lớn hơn 48%. Trên CPU, Q4_K_M chậm hơn rõ rệt (~12%) vì memory bandwidth là bottleneck. Nhưng trên GPU, khi cả hai đều fit trong VRAM, bottleneck chuyển sang compute → quantization size không ảnh hưởng nhiều đến tốc độ. Điều này có nghĩa: nếu bạn có GPU, LUÔN dùng Q4_K_M thay vì Q2_K — quality tốt hơn mà không mất gì.

2. **Laptop gaming có thể sập nguồn khi chạy inference.** Tôi không ngờ rằng model serving 1.5B parameter có thể kéo công suất vượt quá adapter nguồn 65W. Đây là bài học thực tế về power budget trong edge deployment — giống hệt những gì slide nói về `--gpu-memory-utilization` và `--max-num-seqs` trong vLLM: bạn không thể dùng 100% tài nguyên, phải chừa margin cho OS, KV cache, và power limits.

---

## 8. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-metrics.csv` đã commit (record-metrics.py output)
- [x] `benchmarks/bonus-quant-sweep.md` đã commit
- [x] Ít nhất 6 screenshots trong `submission/screenshots/`
- [x] `make verify` exit 0
- [x] Repo trên GitHub ở chế độ **public**

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
