# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3875 | 386 / 434 | 48.5 / 50.4 | 3409 / 3606 / 3606 | 20.6 |
| UD-Q2_K_XL | 2.24 | 2617 | 491 / 545 | 41.3 / 42.6 | 3060 / 3226 / 3226 | 24.2 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.17x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

- **Tốc độ & Kích thước**: `UD-Q2_K_XL` (2.24 GB) nhẹ hơn 0.73 GB và giải mã nhanh hơn 1.17× (24.2 tok/s so với 20.6 tok/s, TPOT P50 giảm từ 48.5 ms xuống 41.3 ms). Điều này phản ánh rõ cơ chế: giai đoạn decode bị giới hạn bởi băng thông bộ nhớ (memory-bandwidth-bound) nên kích thước trọng số nhỏ hơn giúp giảm lượng dữ liệu RAM cần truyền vào cache CPU ở mỗi bước sinh token.
- **Đánh đổi TTFT & Chất lượng**: Ngược lại, TTFT của bản 2-bit chậm hơn (491 ms so với 386 ms) do chi phí dequantize phức tạp hơn trên CPU khi thực hiện prefill (compute-bound). Ngoài ra, với mô hình 2B tham số, quantization 2-bit làm giảm sút độ mạch lạc câu trả lời so với 4-bit.
- **Kết luận**: Máy có 31 GB RAM nên mức tiết kiệm 0.73 GB RAM là không cần thiết so với sự suy giảm chất lượng câu trả lời. Bản 4-bit `UD-Q4_K_XL` vẫn là lựa chọn tối ưu cho serving thực tế.