# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trần Gia Khánh
**MSSV:** 2A202602689
**Cohort:** AICB-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** AMD Ryzen 7 H 255 w/ Radeon 780M Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 30.8 GB
- **Accelerator:** CPU only (Radeon 780M Vulkan thiếu extension, chạy CPU Zen4 AVX2)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Khi chạy `probe`, hệ thống phát hiện Vulkan nhưng driver GPU Radeon 780M thiếu extension `vk::PhysicalDevice::createDevice` gây crash. Tôi đã khắc phục bằng cách vô hiệu hóa `ggml-vulkan.dll` để chuyển sang chạy CPU backend `ggml-cpu-zen4.dll` (AVX2), giúp inference ổn định và đạt tốc độ decode 21.5 tok/s.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3875 | 386 / 434 | 48.5 / 50.4 | 3409 / 3606 / 3606 | 20.6 |
| UD-Q2_K_XL | 2.24 | 2617 | 491 / 545 | 41.3 / 42.6 | 3060 / 3226 / 3226 | 24.2 |

**Quan sát** (≤ 60 chữ): 2-bit decode nhanh hơn 1.17× (24.2 vs 20.6 tok/s) và tiết kiệm 0.73 GB RAM. Tuy nhiên TTFT chậm hơn và chất lượng sinh câu trả lời bị suy giảm sút. Với máy 31 GB RAM, việc đánh đổi chất lượng lấy 0.73 GB là không đáng; bản 4-bit tốt hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.67 | 13000 | 19000 | 21000 | 8.6 | 0.0% |
| 50 | 0.84 | 29000 | 55000 | 57000 | 25.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.25×
- **P95 tăng:** 2.89×
- **Effective concurrency ở 50 users:** 25.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.90 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa trước 50 users (quanh 10–15 users) vì Effective Concurrency = 25.3 vượt 6.32× số slot khả dụng. P95 phồng từ 19s lên 55s hoàn toàn là queue time vì có tới 46 request bị defer trong hàng đợi. Để tăng goodput@SLO, tôi sẽ tăng `--parallel` từ 4 lên 8 slots để phục vụ thêm request đồng thời.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Terraform/Cloud | stub |
| N17 Data pipeline | In-memory corpus | stub |
| N18 Lakehouse | Parquet/Delta | stub |
| N19 Vector + features | Keyword overlap fallback | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 5167.7 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck nằm 100% ở stage LLM do chi phí prefill context và decode. Muốn giảm 2× latency, phải tấn công vào LLM bằng prefix caching cho system prompt cố định và prompt compression rút gọn context retriever.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tăng số luồng tính toán CPU `-t` từ 1 luồng lên 8 luồng (bằng số nhân vật lý của CPU AMD Ryzen 7)

```
before:  12.5 tok/s
after:   21.5 tok/s
speedup: 1.72×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Ở cấu hình 1 luồng (`-t 1`), một nhân CPU đơn lẻ không thể bão hòa hết băng thông bộ nhớ của kênh RAM dual-channel DDR5, khiến tốc độ giải mã chỉ đạt 12.5 tok/s. Khi tăng số luồng lên 4 và 8 luồng (tương ứng với số core vật lý), năng lực đọc song song từ RAM tăng lên rõ rệt, đẩy tốc độ decode lên mức đỉnh 21.5 tok/s (tăng tốc 1.72×).

Tuy nhiên, khi tiếp tục tăng lên 16 luồng (các nhân ảo SMT) và 32 luồng, thông lượng không tăng thêm mà đi ngang rồi sụt giảm nghiêm trọng xuống 15.3 tok/s ở 32 luồng (-29%). Cơ chế ở đây là do giai đoạn decode bị nghẽn bởi băng thông bộ nhớ (memory-bandwidth-bound) chứ không phải do thiếu FLOPs tính toán: 8 luồng vật lý đã khai thác cạn kiệt bus truyền của RAM; các luồng ảo SMT dùng chung execution unit và bộ nhớ đệm cache nên không thể gia tăng thông lượng, trong khi việc oversubscription (32 luồng) gây ra chi phí chuyển ngữ cảnh (context switching) và cache thrashing. Vì vậy, lựa chọn `-t 8` chính là điểm gãy hiệu năng tối ưu nhất.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:**

**Numbers:**

```
before:
after:
speedup:
```

**Điều này nói lên gì mà deck chưa nói:**



---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều làm tôi ngạc nhiên nhất là việc tăng luồng vượt quá số nhân vật lý không những không giúp tăng tốc độ decode mà còn làm giảm tới 29% hiệu năng ở mức 32 luồng do tranh chấp băng thông bộ nhớ và chi phí chuyển ngữ cảnh.

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
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng Google Antigravity AI pair programming assistant để hỗ trợ phân tích phần cứng, sửa lỗi mã nguồn tương thích bảng mã Unicode trên Windows PowerShell, giải thích cơ chế kiến trúc bộ nhớ và hướng dẫn thực hiện tuần tự các bước trong lab.
