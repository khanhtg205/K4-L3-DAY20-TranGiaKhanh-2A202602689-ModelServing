# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 39 | 0.67 | 13000 | 19000 | 21000 | 8.6 | 0.0% |
| 50 | 49 | 0.84 | 29000 | 55000 | 57000 | 25.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.25x** (25% of linear) |
| P95 latency | **2.89x** |
| Effective concurrency at 50 users | 25.3 vs `--parallel 4` slots (occupancy/slot ratio 6.32) |

**Saturated.** Throughput delivered only 1.25x for 5x the offered load, and effective concurrency (25.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.25x while P95 moved 2.89x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

- **Điểm bão hoà & Bằng chứng**: Server bão hoà rõ rệt trước mức 50 users (quanh 10–15 users). Con số thuyết phục nhất là **Effective Concurrency = 25.3**, vượt gấp **6.32×** số slot khả dụng (`--parallel 4`). Khi offered load tăng gấp 5× (từ 10 lên 50 users), thông lượng RPS chỉ nhích nhẹ từ 0.67 lên 0.84 RPS (tăng 1.25×, đã chạm trần plateau), trong khi độ trễ P95 phồng to gấp **2.89×** (từ 19s lên 55s).
- **Queue time vs Compute time**: Khoảng chênh lệch latency khổng lồ (thêm 36 giây ở P95) hoàn toàn là **queue time** (thời gian xếp hàng đợi slot trống), không phải compute time. Điều này được chứng minh bằng metric `requests_deferred = 46` trong suốt quá trình load-50.
- **Knob ưu tiên để nâng Goodput@SLO**: Nếu đặt mục tiêu SLO là P95 ≤ 25s, knob tôi sẽ thay đổi đầu tiên là **tăng `--parallel` từ 4 lên 8 hoặc 12 slots**. Vì máy có tới 31 GB RAM và 8 core CPU Ryzen 7 mạnh mẽ, việc mở rộng slot decode sẽ giải phóng hàng đợi, cho phép continuous batching phục vụ nhiều request song song hơn, giảm thiểu queue time và đưa P95 về dưới ngưỡng SLO.
