# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

- **Họ Tên:** _Nguyen Nang Anh_
- **MHV:** _2A202600184_
- **Cohort:** _A20-K1_
- **Ngày submit:** _2026-05-06_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** _Windows 11_
- **CPU:** _AMD64_
- **Cores:** _8 / 8_
- **CPU extensions:** _AVX2_
- **RAM:** _7.7 GB_
- **Accelerator:** _CPU only_
- **llama.cpp backend đã chọn:** _CPU_
- **Recommended model tier:** _TinyLlama-1.1B_

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

_Máy dùng Python 3.13 không có pre-built wheel, lại thiếu C++ Build Tools nên cài đặt từ source bị lỗi. Phải dùng `uv` tạo virtual environment Python 3.12 để cài bản wheel đúc sẵn. Ngoài ra, phải bật `LongPathsEnabled` trong Registry để sửa lỗi giải nén đường dẫn dài của Windows._

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| (Q4_K_M) | 3458 | 386 / 758 | 57.9 / 100.8 | 3773 / 4639 / 4699 | 17.3 |
| (Q2_K)   | 903 | 451 / 548 | 44.4 / 50.2 | 3051 / 3419 / 3422 | 22.5 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

_Q4_K_M chậm hơn Q2_K khoảng 30% (17.3 vs 22.5 tok/s). Tuy nhiên trên CPU 8 nhân thì 17 tok/s vẫn đủ nhanh để đọc mượt mà, do đó mức độ đánh đổi tốc độ này là hoàn toàn xứng đáng để giữ lại chất lượng văn bản tốt hơn của bản Q4._

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.10 | 25000 | 44000 | 44000 | 0 |
| 50 | 0.10 | 43000 | 53000 | 53000 | 0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 = _<0.XX>_, nghĩa là …

_Ở mức concurrency lớn (50 users), CPU bị thắt cổ chai trầm trọng do các luồng tranh giành nhau tài nguyên (memory bandwidth). P95 tăng vọt lên tới 53 giây, cho thấy chạy backend CPU-only không bao giờ phù hợp cho môi trường Production có High Concurrency._

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _"stub: localhost only"_
- **N17 (Data pipeline):** _"stub: in-memory dict"_
- **N18 (Lakehouse):** _"stub: SQLite"_
- **N19 (Vector + Feature Store):** _"stub: TOY_DOCS"_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _0.0_
- retrieve: _0.1 ms_
- llama-server: _14757.2 ms_

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

_Bottleneck hoàn toàn nằm ở khâu gọi LLM sinh text (chiếm tới >99.9% tổng thời gian). Điều này hoàn toàn khớp với lý thuyết vì local inference trên CPU không có vRAM cực kỳ tốn thời gian tính toán so với các thao tác lookup memory thông thường._

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** _Chuyển đổi (quantization) từ Q4_K_M xuống Q2_K_

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before (Q4_K_M): 17.3 tok/s
after  (Q2_K):   22.5 tok/s
speedup: ~1.30×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

_Lý do chính là do "memory bandwidth bottleneck". Trên kiến trúc chạy CPU-only, tốc độ sinh text phụ thuộc gần như hoàn toàn vào việc CPU kéo weight của model từ RAM lên bộ nhớ đệm (cache) nhanh đến mức nào, thay vì tốc độ tính toán thuần túy. Bằng cách nén model từ Q4 xuống Q2, kích thước vật lý của model giảm đi đáng kể (từ 636MB xuống còn nhỏ hơn). Do đó, lượng dữ liệu phải di chuyển qua bus bộ nhớ cho mỗi token giảm đi, giúp CPU tốn ít thời gian chờ RAM hơn và đẩy tốc độ sinh từ 17.3 lên 22.5 tok/s. Tuy nhiên, sự tăng tốc độ này phải trả giá bằng việc text sinh ra kém tự nhiên và hay lặp từ hơn._

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

_Answer here._

---

## 7. Self-graded checklist

- [ ] `hardware.json` đã commit
- [ ] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [ ] `benchmarks/01-quickstart-results.md` đã commit
- [ ] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
