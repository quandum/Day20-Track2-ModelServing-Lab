# Work Plan — Day 20 Lab: Model Serving & Inference Optimization

> **Kế hoạch chi tiết dựa trên phân tích `README.md`, `rubric.md`, `HARDWARE-GUIDE.md`, và toàn bộ cấu trúc repo.**
>
> **Nguyên tắc quan trọng:**
> - Lab chạy trên laptop cá nhân của bạn — số liệu chỉ so sánh **before vs after trên chính máy bạn**, không so với bạn cùng lớp.
> - Grade rubric dựa trên **độ rõ ràng của setup + tuning + writeup**, không phải tốc độ tuyệt đối.
> - Core path: ~2.5 giờ. Bonus tracks: thêm 1–3 giờ.

---

## 📋 Tổng quan các yêu cầu

### Core (100 pts — bắt buộc)

| # | Track | Yêu cầu | Điểm |
|---|-------|---------|------|
| 1 | 00-setup | `hardware.json` committed; `detect-hardware.py` chạy sạch | 5 |
| 2 | 00-setup | `models/active.json` committed; GGUF file path hợp lệ | 5 |
| 3 | 01-quickstart | Bảng P50/P95/P99 cho **cả Q4_K_M và Q2_K** | 10 |
| 4 | 01-quickstart | TTFT và TPOT được báo cáo riêng biệt | 5 |
| 5 | 02-server | `llama-server` chạy & phục vụ OpenAI-compat `/v1/chat/completions` | 10 |
| 6 | 02-server | `/metrics` từ native server hiển thị `tokens_predicted_total > 0` | 5 |
| 7 | 02-server | locust `-u 10` trong 60s, P95 được báo cáo | 10 |
| 8 | 02-server | locust `-u 50` trong 60s, P95 được báo cáo | 10 |
| 9 | 02-server | Quan sát continuous-batching dưới load (peak `n_busy_slots_per_decode`) | 5 |
| 10 | 03-integration | `pipeline.py` chạy end-to-end (3 queries) + in provenance | 10 |
| 11 | 03-integration | Ít nhất 3 trong số N16/N17/N18/N19 được kết nối (hoặc stub rõ ràng) | 5 |
| 12 | submission | `REFLECTION.md` điền đầy đủ — mọi section có nội dung | 10 |
| 13 | submission | Paragraph "Single change that mattered most" (REFLECTION.md §5) | 10 |
| 14 | repo | Reproducibility — `make setup && make bench && make verify` exit 0 | 10 |

### Bonus (20 pts — tùy chọn)

| # | Yêu cầu | Điểm |
|---|---------|------|
| B1 | Build llama.cpp từ source thành công | 4 |
| B2 | Ít nhất 1 sweep (`thread/quant/ctx-len/batch-size/gpu-offload`) + commit kết quả | 4 |
| B3 | Bonus speedup quantified (before/after numbers trong REFLECTION.md §5) | 4 |
| B4 | Ít nhất 1 open challenge từ `CHALLENGES.md` (C1–C9) | 4 |
| B5 | MLX comparison (Apple Silicon only) | 4 |

---

## 🔄 Quy trình tổng thể (theo BMAD Method)

Bài lab khuyến nghị dùng **BMAD Method** với 5 persona. Đây là flow đề xuất:

```
PM/Spec → Architect → Developer → QA/Verify → Ops/Reflect
```

| Persona | Vai trò trong lab này |
|---------|----------------------|
| **PM/Spec** | Xác định mục tiêu đo lường được: TTFT, TPOT, P95 targets |
| **Architect** | Chọn model tier, quantization, backend phù hợp với hardware |
| **Developer** | Chạy code, implement pipeline |
| **QA/Verify** | Kiểm tra số liệu hợp lý, chạy `make verify` |
| **Ops/Reflect** | Viết REFLECTION.md, giải thích insight |

---

## 📅 Kế hoạch chi tiết từng bước

---

### PHASE 0: Chuẩn bị & Đọc hiểu (15 phút)

#### Bước 0.1 — Đọc rubric
- 📄 Đọc kỹ [`rubric.md`](rubric.md) — hiểu grader chấm gì
- ✅ Output: Nắm được 14 tiêu chí core + 5 tiêu chí bonus

#### Bước 0.2 — Đọc Hardware Guide
- 📄 Đọc [`HARDWARE-GUIDE.md`](HARDWARE-GUIDE.md)
- Xác định path phù hợp với laptop của bạn:
  - Model tier (theo RAM): TinyLlama-1.1B / Qwen2.5-1.5B / Llama-3.2-3B / Qwen2.5-7B
  - Backend: CUDA / Metal / Vulkan / ROCm / CPU
- ✅ Output: Biết trước model nào sẽ được chọn, backend nào sẽ được dùng

#### Bước 0.3 — Đọc VIBE-CODING.md (BMAD primer)
- 📄 Đọc [`VIBE-CODING.md`](VIBE-CODING.md) — hiểu cách dùng 5 persona
- ✅ Output: Sẵn sàng áp dụng BMAD workflow

#### Bước 0.4 — Fork/Clone repo lên GitHub
- Fork repo này lên GitHub account của bạn
- **Set repo PUBLIC** (bắt buộc — nếu private, grader không xem được → 0 điểm)
- Clone về máy
- ✅ Output: Repo public trên GitHub, sẵn sàng làm việc

---

### PHASE 1: Track 00 — Setup (20 phút) — [Rubric #1, #2]

#### Bước 1.1 — Probe hardware
```bash
# Windows (dùng PowerShell):
python 00-setup/detect-hardware.py

# macOS/Linux:
make probe
```
- ✅ Output: `hardware.json` được tạo ở root
- 📸 Screenshot: `submission/screenshots/01-hardware-probe.png`

#### Bước 1.2 — Cài đặt dependencies + build llama-cpp-python + download model
```bash
# Windows:
pwsh -ExecutionPolicy Bypass -File 00-setup/windows-setup.ps1

# macOS:
make setup    # hoặc bash 00-setup/macos-setup.sh

# Linux:
make setup    # hoặc bash 00-setup/linux-setup.sh
```
- Script sẽ:
  1. Tạo `.venv/`
  2. Cài `requirements.txt`
  3. Build `llama-cpp-python` với backend flag đúng cho hardware
  4. Chạy `download-model.py` → tải GGUF model phù hợp với RAM
- ⚠️ Nếu mạng chặn Hugging Face: làm theo [`00-setup/MANUAL-DOWNLOAD.md`](00-setup/MANUAL-DOWNLOAD.md)
- ✅ Output: `models/active.json` tồn tại, `.gguf` file tồn tại
- ✅ Rubric #1, #2 hoàn thành

---

### PHASE 2: Track 01 — llama.cpp Quickstart (30 phút) — [Rubric #3, #4]

#### Bước 2.1 — Chạy benchmark cơ bản
```bash
# Windows:
python 01-llama-cpp-quickstart/benchmark.py

# macOS/Linux:
make bench
```
- Script sẽ:
  - Load primary GGUF model từ `models/active.json`
  - Đo single-prompt latency (TTFT + TPOT)
  - Chạy 20-request batch → P50/P95/P99
  - So sánh Q4_K_M vs Q2_K (side-by-side)

#### Bước 2.2 — Kiểm tra kết quả
- ✅ Output: `benchmarks/01-quickstart-results.md` được tạo
- Kiểm tra bảng có đủ:
  - TTFT và TPOT riêng biệt (không chỉ E2E)
  - P50/P95/P99 cho cả Q4_K_M và Q2_K
  - Decode rate (tok/s)
- 📸 Screenshot: `submission/screenshots/02-quickstart-bench.png`

#### Bước 2.3 — (Tùy chọn) Thử các knob
Có thể edit `benchmark.py` hoặc set env vars để thử:
- `LAB_N_THREADS`: thử `cores//2`, `cores`, `cores*2`
- `LAB_N_CTX`: thử context window khác nhau
- Ghi lại observation cho REFLECTION.md

#### Bước 2.4 — Ghi observation cho REFLECTION.md §2
- Q4_K_M vs Q2_K trên máy bạn ra sao?
- Quality có đáng đánh đổi không?
- ✅ Rubric #3, #4 hoàn thành

---

### PHASE 3: Track 02 — llama-server (60 phút) — [Rubric #5, #6, #7, #8, #9]

Track này có 2 server path — bạn cần cả hai:

| Server | Command | Mục đích | Có /metrics? |
|--------|---------|----------|:---:|
| **Python server** | `make serve` | Chat API + load test | ❌ |
| **Native server** | `make serve-native` | Observability với `/metrics` | ✅ |

#### Bước 3.1 — Khởi động Python server + smoke test
```bash
# Terminal 1: Khởi động Python server
make serve
# (hoặc python -m llama_cpp.server --model ... --port 8080)

# Terminal 2: Smoke test
make smoke
# (hoặc python 02-llama-cpp-server/smoke-test.py)
```
- ✅ Server lắng nghe trên `http://0.0.0.0:8080`
- ✅ `/v1/chat/completions` hoạt động
- 📸 Screenshot: `submission/screenshots/03-server-running.png` (cả server log + curl thành công)

#### Bước 3.2 — Build llama.cpp từ source (để có native server)
```bash
make build-llama
```
- Clone + cmake + build llama.cpp cho backend phù hợp
- ⚠️ Cần `cmake` — nếu chưa có: `make setup` sẽ cài, hoặc cài thủ công
- ✅ `BONUS-llama-cpp-optimization/llama.cpp/build/bin/llama-server` tồn tại
- ✅ Đây cũng là **Bonus B1** (4 pts)

#### Bước 3.3 — Khởi động Native server + kiểm tra /metrics
```bash
# Terminal 1: Dừng Python server, khởi động native server
make serve-native

# Terminal 2: Gửi 1 request test
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"local","messages":[{"role":"user","content":"Hello"}]}'

# Terminal 2: Kiểm tra /metrics
curl -s http://localhost:8080/metrics | grep tokens_predicted_total
```
- ✅ `llamacpp:tokens_predicted_total` > 0 sau khi có request
- ✅ Rubric #5, #6 hoàn thành

#### Bước 3.4 — Locust load test @ 10 users
```bash
# Terminal 2 (native server vẫn chạy ở Terminal 1):
make load-10
```
- Chạy 60 giây với 10 concurrent users
- ✅ Xuất hiện bảng P50/P95/P99
- 📸 Screenshot: `submission/screenshots/04-locust-10.png`
- ✅ Rubric #7 hoàn thành

#### Bước 3.5 — Locust load test @ 50 users
```bash
make load-50
```
- Chạy 60 giây với 50 concurrent users
- ✅ Xuất hiện bảng P50/P95/P99
- 📸 Screenshot: `submission/screenshots/05-locust-50.png`
- ✅ Rubric #8 hoàn thành

#### Bước 3.6 — Record metrics trong lúc load test (continuous batching observation)
```bash
# Terminal 2: Bắt đầu record metrics (60s)
make metrics

# Terminal 3 (cùng lúc): Chạy load test
make load-50
```
- `record-metrics.py` poll `/metrics` mỗi 5s → `benchmarks/02-server-metrics.csv`
- ✅ Quan sát peak `n_busy_slots_per_decode` / `requests_processing`
- Ghi observation vào REFLECTION.md §3
- ✅ Rubric #9 hoàn thành

#### Bước 3.7 — Ghi kết quả vào REFLECTION.md §3
- Copy bảng locust 10u và 50u
- Ghi observation về continuous batching
- ✅ Rubric #7, #8, #9 hoàn thành

---

### PHASE 4: Track 03 — Milestone Integration (30 phút) — [Rubric #10, #11]

#### Bước 4.1 — Đảm bảo server đang chạy
```bash
# Khởi động lại Python server nếu cần:
make serve
```

#### Bước 4.2 — Chạy pipeline
```bash
make pipeline
# (hoặc python 03-milestone-integration/pipeline.py)
```
- Pipeline chạy 3 example queries:
  1. Embed query → retrieve top-K → build prompt → gọi llama-server → trả lời
  2. In ra retrieved-context provenance (nguồn gốc context)
  3. Đo latency từng bước với `time.perf_counter()`

#### Bước 4.3 — Xác định N16–N19 connections
Trong `pipeline.py`, xác định pieces nào được kết nối thật, pieces nào stub:

| Milestone | Piece | Real hay Stub? |
|-----------|-------|----------------|
| N16 | Cloud/IaC | K3d cluster / GCP / docker-compose / "stub: localhost" |
| N17 | Data Pipeline | Airflow DAG / batch job / "stub: in-memory dict" |
| N18 | Lakehouse | Delta Lake / Iceberg / "stub: SQLite" |
| N19 | Vector + Feature Store | Qdrant / Feast / "stub: TOY_DOCS" |

- ✅ Ít nhất 3/4 được wired (hoặc stub có lý do rõ ràng)
- 📸 Screenshot (optional): `submission/screenshots/09-pipeline-output.png`

#### Bước 4.4 — Ghi kết quả vào REFLECTION.md §4
- Liệt kê N16–N19 connections
- Bảng latency: embed / retrieve / llama-server
- Bottleneck nằm ở đâu? Có khớp kỳ vọng không?
- ✅ Rubric #10, #11 hoàn thành

---

### PHASE 5: Bonus Tracks (Tùy chọn, 60–120 phút) — [Bonus B1–B5]

> **Chiến lược:** Không cần làm hết. Chọn **1 sweep + 1 challenge** phù hợp nhất với hardware của bạn. Một insight rõ ràng > năm sweeps lủng củng.

#### Bước 5.1 — Bonus: Build llama.cpp từ source (nếu chưa làm ở Bước 3.2)
```bash
make build-llama
```
- ✅ Bonus B1 (4 pts)

#### Bước 5.2 — Bonus: Chạy sweep phù hợp với hardware

Chọn **1–2 sweeps** dựa trên hardware của bạn:

| Hardware | Sweep nên chạy | Command | Insight |
|----------|---------------|---------|---------|
| **CPU only** | Thread sweep | `make sweep-thread` | Curve tokens/s peak ở physical-core count rồi drop |
| **Tight RAM** | Quant sweep | `make sweep-quant` | Q2_K → Q8_0: file size vs decode speed vs quality |
| **Long-context** | Context length sweep | `make sweep-ctx` | Prefill ~O(N²) — TTFT trong long-context |
| **Có GPU** | GPU offload sweep | `make sweep-gpu` | -ngl 0,8,16,...,99 — partial vs full offload |
| **Server workload** | Batch size sweep | `make sweep-batch` | Chunked prefill: throughput vs TTFT tradeoff |

- ✅ Output: `benchmarks/bonus-<sweep-name>.md` được tạo
- 📸 Screenshot: `submission/screenshots/06-bonus-sweep.png`
- ✅ Bonus B2 (4 pts)

#### Bước 5.3 — Bonus: Chọn 1 challenge từ CHALLENGES.md

| Challenge | Mô tả | Phù hợp với |
|-----------|-------|-------------|
| **C1** | Speculative decoding (draft + target model) | Có GPU, muốn thử latency optimization |
| **C2** | KV-cache quantization (q8_0) | Tight RAM, muốn tiết kiệm memory |
| **C3** | Multi-LoRA serving | Có kinh nghiệm fine-tuning |
| **C4** | Best-of-N parallel sampling + reranker | Muốn thử quality-vs-latency |
| **C5** | "Weakest laptop" challenge | Laptop yếu — tìm model nhỏ nhất vẫn useful |
| **C6** | Vulkan vs CUDA head-to-head | Có NVIDIA GPU |
| **C7** | CPU instruction set archaeology | CPU-only Linux |
| **C8** | Semantic caching | Muốn thử cache stack 3 tầng |
| **C9** | Embedding & reranker serving | Muốn hiểu prefill-bound regime |

- ✅ Bonus B4 (4 pts)

#### Bước 5.4 — Bonus: MLX comparison (chỉ Apple Silicon)
```bash
pip install mlx mlx-lm
make mlx-compare
```
- So sánh MLX vs llama.cpp Metal trên 10 prompts
- ✅ Output: `benchmarks/bonus-mlx-vs-llama-cpp.md`
- 📸 Screenshot: `submission/screenshots/08-mlx-vs-llamacpp.png`
- ✅ Bonus B5 (4 pts)

#### Bước 5.5 — Bonus: Embedding server + Semantic cache
```bash
make serve-embed &      # embedding server trên :8081
make embed-demo         # embedding/reranker demo
make semantic-cache     # semantic cache demo
```
- Có thể chạy với `--offline` (không cần server)
- ✅ Ghi hit-rate + threshold tradeoff

---

### PHASE 6: Submission — Viết REFLECTION.md & Verify (30 phút) — [Rubric #12, #13, #14]

#### Bước 6.1 — Điền REFLECTION.md
Mở [`submission/REFLECTION.md`](submission/REFLECTION.md) và điền **TẤT CẢ** các section:

| Section | Nội dung cần điền |
|---------|-------------------|
| **Header** | Họ tên, Cohort, Ngày submit |
| **§1 Hardware spec** | Paste output `detect-hardware.py` + setup story (≤80 chữ) |
| **§2 Track 01 numbers** | Paste bảng từ `benchmarks/01-quickstart-results.md` + observation (≤50 chữ) |
| **§3 Track 02 load test** | Bảng locust 10u + 50u + batching observation |
| **§4 Track 03 integration** | N16–N19 connections + latency breakdown + bottleneck reflection (≤60 chữ) |
| **§5 Bonus — Single change** | **QUAN TRỌNG NHẤT** — before/after numbers + giải thích WHY (1–2 đoạn) |

> ⚠️ **Lưu ý đặc biệt cho §5:** Đây là section grader đọc kỹ nhất. Giải thích bằng mental model về hardware (memory bandwidth, compute, cache) — không "vibes-based". Nếu kết quả khác kỳ vọng, nói rõ — grader thưởng điểm cho sự trung thực.

#### Bước 6.2 — Thu thập screenshots
Đảm bảo có trong `submission/screenshots/`:

| # | Filename | Nội dung | Bắt buộc? |
|---|----------|----------|:---:|
| 1 | `01-hardware-probe.png` | Output `detect-hardware.py` | ✅ |
| 2 | `02-quickstart-bench.png` | Output `benchmark.py` | ✅ |
| 3 | `03-server-running.png` | Server log + curl thành công | ✅ |
| 4 | `04-locust-10.png` | Locust -u 10 summary | ✅ |
| 5 | `05-locust-50.png` | Locust -u 50 summary | ✅ |
| 6 | `06-bonus-sweep.png` | Ít nhất 1 chart/table từ bonus sweep | ✅ |
| 7 | `07-grafana-or-prom.png` | Prometheus UI (optional) | ❌ |
| 8 | `08-mlx-vs-llamacpp.png` | MLX comparison (Apple Silicon) | ❌ |
| 9 | `09-pipeline-output.png` | Pipeline end-to-end output | ❌ |

#### Bước 6.3 — Chạy verify
```bash
make verify
```
- Script kiểm tra: artifacts tồn tại, REFLECTION.md đã edit, screenshots có mặt
- ✅ Phải exit code 0 — nếu không, grader có thể trừ điểm Q14
- ✅ Rubric #12, #14 hoàn thành

#### Bước 6.4 — Push lên GitHub
```bash
git add .
git commit -m "Complete Day 20 Lab submission"
git push origin main
```
- ✅ Repo PUBLIC
- ✅ Rubric #13 hoàn thành (grader đọc REFLECTION.md từ repo)

#### Bước 6.5 — Submit lên VinUni LMS
- Paste public GitHub URL vào ô submission của Day 20
- ✅ HOÀN THÀNH!

---

## 📊 Bảng tổng kết Deliverables

| Deliverable | Đường dẫn | Rubric |
|-------------|-----------|--------|
| Hardware probe | `hardware.json` | #1 |
| Model active config | `models/active.json` | #2 |
| Quickstart results | `benchmarks/01-quickstart-results.md` | #3, #4 |
| Server running screenshot | `submission/screenshots/03-server-running.png` | #5 |
| /metrics evidence | curl output trong REFLECTION.md | #6 |
| Locust 10u screenshot | `submission/screenshots/04-locust-10.png` | #7 |
| Locust 50u screenshot | `submission/screenshots/05-locust-50.png` | #8 |
| Metrics CSV | `benchmarks/02-server-metrics.csv` | #9 |
| Pipeline output | Terminal output / screenshot | #10, #11 |
| REFLECTION.md | `submission/REFLECTION.md` | #12, #13 |
| verify exit 0 | `make verify` | #14 |
| Bonus sweep results | `benchmarks/bonus-*.md` | B2, B3 |
| Bonus challenge writeup | REFLECTION.md §5 hoặc `bonus/<challenge>.md` | B4 |

---

## ⏱️ Timeline ước tính

| Phase | Nội dung | Thời gian |
|-------|----------|:---:|
| Phase 0 | Chuẩn bị & đọc hiểu | 15 min |
| Phase 1 | Track 00 — Setup | 20 min |
| Phase 2 | Track 01 — Quickstart | 30 min |
| Phase 3 | Track 02 — Server | 60 min |
| Phase 4 | Track 03 — Integration | 30 min |
| Phase 5 | Bonus tracks (tùy chọn) | 60–120 min |
| Phase 6 | Submission & verify | 30 min |
| **Total core** | | **~3 giờ** |
| **Total với bonus** | | **~4–5 giờ** |

---

## 🎯 Mẹo để đạt điểm cao

1. **Đọc rubric trước khi làm** — biết grader chấm gì để tập trung đúng chỗ
2. **§5 REFLECTION.md là quan trọng nhất** — grader đọc kỹ phần "Single change that mattered most"
3. **Giải thích WHY, không chỉ WHAT** — dùng mental model về hardware (memory bandwidth, compute bound, cache miss)
4. **Trung thực nếu kết quả khác kỳ vọng** — grader thưởng điểm cho insight, không phải số đẹp
5. **1 insight sâu > 5 sweeps nông** — đừng cố chạy hết bonus tracks
6. **Chạy `make verify` trước khi push** — đảm bảo exit code 0
7. **Repo PUBLIC** — nếu private, 0 điểm
8. **Screenshots crop tight** — grader muốn thấy data, không phải wallpaper

---

## 🔗 Tài liệu tham khảo nhanh

| File | Dùng cho |
|------|----------|
| `README.md` | Tổng quan lab + quick start |
| `rubric.md` | Biết grader chấm gì |
| `HARDWARE-GUIDE.md` | Chọn model + backend |
| `VIBE-CODING.md` | BMAD workflow guide |
| `00-setup/README.md` | Setup troubleshooting |
| `01-llama-cpp-quickstart/README.md` | Track 01 knobs |
| `02-llama-cpp-server/README.md` | Track 02 server options |
| `03-milestone-integration/README.md` | Track 03 integration target |
| `BONUS-llama-cpp-optimization/README.md` | Bonus track overview |
| `BONUS-llama-cpp-optimization/CHALLENGES.md` | 9 open challenges |
| `submission/screenshots/README.md` | Required screenshots checklist |
| `submission/REFLECTION.md` | Personal report template |

---

> **Chúc bạn hoàn thành lab thành công! Bắt đầu từ Phase 0 và làm tuần tự từng bước.** 🚀
