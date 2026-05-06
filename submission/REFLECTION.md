# Reflection — Lab 20 (Personal Report)

**Họ Tên:** Lê Minh Tuấn
**Cohort:** 2A202600379
**Ngày submit:** 2026-05-06

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** Windows 11 Home
- **CPU:** Intel Core i5-1135G7 @ 2.40GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2, FMA, F16C
- **RAM:** 16 GB
- **Accelerator:** CPU Only
- **llama.cpp backend đã chọn:** CPU (Native MSVC Build)
- **Recommended model tier:** Qwen2.5-1.5B

**Setup story:** 
Máy chạy Windows nên ban đầu gặp lỗi `UnicodeDecodeError` khi chạy script verify, tôi đã fix bằng cách thêm `encoding="utf-8"` vào lệnh mở file. Tôi đã tự tay build thành công `llama-server.exe` từ source C++ bằng bộ công cụ MSVC để tối ưu hóa tận dụng tập lệnh AVX2 của chip Intel Gen 11, giúp hỗ trợ đầy đủ Prometheus Metrics.

---

## 2. Track 01 — Quickstart numbers

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| Qwen2.5-1.5B (Q4_K_M) | 1389 | 148 / 195 | 48.2 / 50.0 | 3162 / 3309 / 3351 | 20.7 |
| Qwen2.5-1.5B (Q2_K)   | 1157 | 191 / 230 | 40.1 / 43.7 | 2721 / 2945 / 2986 | 24.9 |

**Một quan sát:** 
Q4_K_M chậm hơn Q2_K khoảng 20% nhưng chất lượng câu trả lời thông minh hơn hẳn. Với CPU i5 Gen 11, mức 20 tok/s của Q4 là quá đủ cho trải nghiệm chat thời gian thực, nên việc đánh đổi tốc độ lấy chất lượng là hoàn toàn xứng đáng.

---

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 8.5 | 120 | 4500 | 4800 | 0 |
| 50 | 6.2 | 450 | 12000 | 15000 | 0 |

**KV-cache observation:** 
Peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 tăng cao do nhiều người dùng chia sẻ bộ nhớ đệm. Nhờ PagedAttention trong llama.cpp, hệ thống vẫn ổn định mà không bị crash dù RAM CPU bị ép khá mạnh.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** batch job (scripts)
- **N18 (Lakehouse):** stub: SQLite
- **N19 (Vector + Feature Store):** stub: TOY_DOCS (Milestone integration)

**Nơi tốn nhiều ms nhất trong pipeline:**
- retrieve: 0.1 ms
- llama-server: 10930 ms (Lần 1) / 2983 ms (Lần 2 nhờ Cache)

**Reflection:** 
Bottleneck nằm ở bước "Prefill" khi LLM phải đọc context lớn lần đầu. Tuy nhiên, nhờ **Prefix Caching**, thời gian xử lý giảm 70% ở lần gọi thứ hai. Điều này cho thấy trong hệ thống RAG, việc tối ưu hóa cache quan trọng không kém việc tối ưu hóa model.

---

## 5. Bonus — The single change that mattered most

**Change:** Build llama.cpp native từ source với Release mode trên Windows.

**Before vs after:**
```
before: tpot = 55.4 ms (chạy qua bản python mặc định)
after:  tpot = 48.2 ms (chạy qua bản native exe)
speedup: ~1.15×
```

**Tại sao nó work:** 
Bản build native cho phép compiler (MSVC) sử dụng các tập lệnh vector hóa (AVX2) mạnh nhất của CPU. Nó cũng loại bỏ độ trễ của Python GIL khi server xử lý đa luồng, giúp việc phục vụ 10-50 user cùng lúc ổn định hơn, không bị treo giữa chừng.

---

## 6. (Optional) Điều ngạc nhiên nhất
Tôi rất bất ngờ khi thấy bản build native lại hỗ trợ giao diện Prometheus chuyên nghiệp đến vậy. Việc nhìn thấy biểu đồ token nhảy múa theo thời gian thực giúp tôi hiểu rõ hơn về cách "động cơ" LLM vận hành bên dưới.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` đã commit
- [x] `benchmarks/bonus-*.md` đã commit (thread sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/`
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
