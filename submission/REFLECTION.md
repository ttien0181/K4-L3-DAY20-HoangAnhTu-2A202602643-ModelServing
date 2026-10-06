# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Hoang Anh Tu
**MSSV:** 2A202602643
**Cohort:** A20-K2
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

- **OS:** Windows 11 (PowerShell 5.1)
- **CPU:** AMD64 (Intel/AMD x86_64)
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** — (không có đặc biệt)
- **RAM:** ~16 GB
- **Accelerator:** CPU only (không có GPU)
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cpu-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` (primary) + `UD-Q2_K_XL` (compare)

**Chạy ở đâu:** laptop của tôi (CPU-only, không GPU)

**Setup story:** Máy không có GPU, chọn Qwen3.5 0.8B (~0.9 GB) để chạy mượt. Runtime tải prebuilt binary Windows x64 CPU; model tải từ Hugging Face qua `download-model.py`. Không gặp blocker nào đáng kể.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2157 | 136 / 148 | 16.1 / 18.9 | 1152 / 1325 / 1325 | 62.0 |
| UD-Q2_K_XL | 0.39 | 1539 | 181 / 218 | 15.6 / 16.1 | 1153 / 1208 / 1208 | 64.1 |

**Quan sát:** Bản 2-bit decode nhanh hơn ~1.03x và nhỏ hơn 0.11 GB, nhưng TTFT P50 tệ hơn 33% (181 ms vs 136 ms) và P95 tệ hơn 47% (218 ms vs 148 ms). Trên máy CPU-only bị giới hạn bởi compute, dequantization nặng hơn ở prefill nên 2-bit không rõ ràng là chiến thắng cho latency-sensitive chat. Tôi tự so chất lượng bằng cách chạy cùng một câu hỏi trên cả hai model: câu trả lời vẫn có ý nghĩa với cả hai quantization ở model nhỏ này, nhưng bản 4-bit ổn định hơn về first-token latency.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.31 | 5900 | 10000 | 11000 | 8.2 | 0.0% |
| 50 | 1.31 | 31000 | 39000 | 41000 | 35.2 | 0.0% |

- **Offered load tăng 5×, throughput thực tế tăng:** 1.00×
- **P95 tăng:** 3.90×
- **Effective concurrency ở 50 users:** 35.2 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`:** 4.00 / 4 slots (100%)

**Saturation reading:** Server bão hoà ở mức 50 users. Bằng chứng thuyết phục nhất là effective concurrency đạt 35.2 trong khi chỉ có 4 slot (occupancy/slot ratio 8.79), đồng thời throughput không tăng dù offered load tăng 5×. Khi offered load tăng 5× mà RPS giữ nguyên 1.31, phần latency tăng thêm (P95 từ 10s lên 39s) chính là queue time, không phải compute time: request phải xếp hàng chờ slot và được ghi nhận qua `requests_deferred` peak = 46. Để nâng goodput@SLO, knob đầu tiên tôi đổi là `--parallel` (tăng số slot), vì nó nâng trần throughput trực tiếp; thứ hai là giảm `max_tokens` trong mix tải để mỗi slot giải phóng nhanh hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

| Day | Piece | Real hay stub? |
|---|---|
| N16 Cloud/IaC | stub | không triển khai |
| N17 Data pipeline | stub | không triển khai |
| N18 Lakehouse | stub | không triển khai |
| N19 Vector + features | stub | retrieval dùng keyword overlap trên TOY_DOCS, không có embedding server |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query):
- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 4262.1 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection:** Đúng như kỳ vọng với model 0.8B chạy CPU và retrieval trên 6 tài liệu ngắn: retrieval gần như free, embed cũng 0 ms vì không có embedding server. Nếu phải giảm latency pipeline 2×, tôi tấn công vào LLM stage trước: tăng thread count lên `-t 8` (tune cho ~1.02x decode speedup), hoặc giảm `max_tokens` trong generation từ 200 xuống 64–96 để cắt decode time.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** Tuning thread count `-t` từ mặc định physical cores (4) lên logical cores (8)

```
before:  66.1 tok/s  (tg128, -t 4)
after:   67.1 tok/s  (tg128, -t 8)
speedup: 1.02x
```

**Tại sao nó work:** Trên CPU decode bị giới hạn bởi memory bandwidth, không phải FLOPs. Từ 1→4 threads throughput tăng mạnh vì 4 worker có thể che latency của nhau khi đọc weights. Từ 4→8 threads đường cong flatten (66.1 → 67.1 tok/s, +1.6%): 8 logical threads trên 4 physical cores chỉ đủ che một phần latency nhỏ của hyperthreading, không đủ tạo ra throughput thêm đáng kể. Tuy nhiên, bước lên 16 threads (2× logical) throughput sụt mạnh xuống 48.3 tok/s: 16 worker tranh nhau cùng các memory channel và scheduler time-slice, nên mỗi thread tốn thời gian chờ hơn là tính toán. Điều này khớp với kiến thức về decode bị memory-bandwidth-bound: sau điểm bão hòa của physical cores, thread thừa không còn thêm việc hữu ích mà chỉ làm tăng cạnh tranh. Mặc dù speedup tuyệt đối khiêm tốn (1.02x), nó là before/after thật trên máy tôi và phù hợp làm knob tune trên CPU.

---

## 6. Bonus  *(optional)*

_(chưa làm — bỏ trống)_

---

## 7. Điều làm bạn ngạc nhiên nhất

Hai điều: (1) Bản 2-bit decode nhanh hơn bản 4-bit dù nhỏ hơn, nhưng TTFT lại tệ hơn rõ rệt — điều này minh họa rõ ràng rằng "nhỏ hơn = nhanh hơn" không đúng với prefill trên CPU. (2) Tại 50 users, throughput hoàn toàn không tăng dù có thêm 40 người dùng ảo; phần latency tăng thêm không phải do model tính chậm hơn mà hoàn toàn là queue time từ slot thiếu hụt.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** đã được thay bằng nhận xét của tôi
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing`
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

---

## 9. Khai báo sử dụng AI

Kilo (AI coding assistant) — dùng để chạy các script lab, phân tích output, và điền các section "required" trong `benchmarks/*.md` và `submission/REFLECTION.md` dựa trên số liệu thực tế thu được từ máy của tôi. Tất cả số liệu đều được sinh ra bởi script trên máy local; AI chỉ hỗ trợ định dạng và giải thích, không bịa số.
