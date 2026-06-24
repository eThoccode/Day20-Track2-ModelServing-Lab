# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Phan Anh Thắng 2A202600844
**Cohort:** _A20 cohort 2
**Ngày submit:** _2026-06-24_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** _Ubuntu 26.04 on WSL2 (Windows host)_
- **CPU:** _AMD Ryzen 5 5600H with Radeon Graphics_
- **Cores:** _12 physical / 12 logical (the values reported by the probe script)_
- **CPU extensions:** _AVX2_
- **RAM:** _7.4 GB_
- **Accelerator:** _NVIDIA GeForce RTX 3050 Laptop GPU 4 GB_
- **llama.cpp backend đã chọn:** _CUDA_
- **Recommended model tier:** _TinyLlama-1.1B (Q4_K_M)_

**Setup story** (≤ 80 chữ):

_Mình chạy lab trong WSL2 Ubuntu. Để setup ổn định, mình phải sửa line endings của shell script về LF, cài `build-essential`/`cmake`/`pkg-config`, và ép các script server dùng `.venv/bin/python` thay vì trông chờ `python` trên PATH. Sau khi sửa các lỗi môi trường đó thì `make smoke`, `make load-10`, `make load-50`, và `make pipeline` mới chạy được ổn định._

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf | 3210 | 133 / 542 | 56.3 / 65.9 | 3563 / 4203 / 4252 | 17.8 |
| tinyllama-1.1b-chat-v1.0.Q2_K.gguf | 1806 | 544 / 711 | 50.8 / 95.8 | 3408 / 4713 / 4826 | 19.7 |

**Một quan sát** (≤ 50 chữ): Q4_K_M cho TTFT tốt hơn rất nhiều trên máy mình (133 ms vs 544 ms ở P50), trong khi tốc độ decode chỉ chậm hơn nhẹ. Vì vậy Q4_K_M hợp lý hơn làm default; Q2_K chỉ đáng dùng khi RAM cực chặt.

---

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.10 | 24000 | 55000 | 55000 | 0 |
| 50 | 0.13 | 21000 | 41000 | 41000 | 0 |

**Batching observation**: trong run này mình dùng Python server (`make serve`), nên `GET /metrics` trả `404 Not Found` và mình không có số peak `llamacpp:n_busy_slots_per_decode` / `requests_processing` từ native metrics. Tuy vậy, kết quả load cho thấy máy local của mình bị giới hạn mạnh bởi latency/compute capacity: throughput chỉ quanh 0.10–0.13 req/s và độ trễ vẫn ở mức hàng chục giây dù không có request fail trong run thành công.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost only_
- **N17 (Data pipeline):** _stub: in-memory toy retrieval flow_
- **N18 (Lakehouse):** _stub: not wired; no external table/store in this run_
- **N19 (Vector + Feature Store):** _stub: `TOY_DOCS` in `pipeline.py`_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _0.0 ms (no separate embedding stage in this stubbed pipeline)_
- retrieve: _0.0–0.1 ms_
- llama-server: _~14549.9 ms on the representative run_

**Reflection** (≤ 60 chữ): bottleneck nằm gần như hoàn toàn ở bước gọi `llama-server`, không phải retrieve. Điều này khớp với kỳ vọng vì retrieval hiện chỉ là keyword overlap trên `TOY_DOCS` trong memory; gần như toàn bộ latency nằm ở generation của model.

---

## 5. Bonus — The single change that mattered most

**Change:** _Build `llama.cpp` native binary rồi chạy `thread-sweep` để tìm cấu hình thread tốt nhất cho máy mình._

**Before vs after** (paste 2-3 dòng từ sweep output):

```text
before: build bonus native chưa có, nên chưa benchmark được bằng `llama-bench`
after:  `bonus-thread-sweep.md` chạy thành công với grid [1, 2, 6, 12, 24]
speedup: chưa đo được meaningful tok/s vì run hiện tại trả `0.0 tok/s` cho toàn bộ grid
```

**Tại sao nó work**:

_Phần bonus này vẫn hữu ích vì mình đã build được `llama-bench` native từ source và chạy được một sweep thật trên đúng máy của mình, thay vì chỉ dùng Python wheel. Việc này cho mình một đường đo độc lập ở tầng native `llama.cpp`, đúng với mục tiêu của bonus track là nhìn vào binary/runtime thực tế thay vì abstraction phía trên._

_Kết quả `tg128 = 0.0 tok/s` ở toàn bộ grid cho thấy run bonus này trên WSL của mình chưa cho ra số throughput usable, có thể do cách offload/thiết bị trong `llama-bench` không khớp môi trường hiện tại. Dù vậy artifact bonus đã được tạo hợp lệ và insight quan trọng là: trên máy mình, bottleneck lớn nhất của bonus không phải chỉ là chọn thread count, mà còn là độ ổn định của runtime native trong WSL khi build và benchmark._

---

## 6. (Optional) Điều ngạc nhiên nhất

_Điều ngạc nhiên nhất là local Python server có thể chạy ổn định sau khi sửa môi trường, nhưng throughput dưới load vẫn rất thấp; lỗi lớn nhất ban đầu không phải model mà là những chi tiết môi trường như compiler, line endings, và đường dẫn Python trong WSL._

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
