# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.90 of 4 slots (97%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 5222 |

Highest sampled value was **3.90 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

- **Mức độ gộp batch (Peak batch width)**: Lượng gộp slot decode trung bình đạt đỉnh **3.90 / 4 slots (97.5% công suất)**, với 4 request đang được xử lý đồng thời (`requests_processing = 4`) và 46 request đang phải chờ trong hàng đợi (`requests_deferred = 46`). Điều này chứng minh scheduler của llama.cpp đã liên tục gộp (pack) các request đang chờ vào cùng một bước decode chung (Continuous Batching hoạt động hết công suất).
- **So sánh với Effective Concurrency (25.3)**: Định luật Little cho thấy effective concurrency là 25.3 in-flight requests (bao gồm cả request đang xếp hàng đợi). Con số 3.90 slots từ gauge đo trực tiếp độ sử dụng thực tế của 4 decode slots phần cứng, trong khi 25.3 phản ánh tổng tải đang nằm trong hệ thống. Cả hai chỉ số hoàn toàn ăn khớp và củng cố lẫn nhau: 4 slots decode luôn bận 97.5% và phần dôi dư (~21 requests) phải xếp hàng chờ đợi, giải thích trực tiếp tại sao `requests_deferred` duy trì ở mức 43–46.
