# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 12.5 | 58% |
| 4 | 21.4 | 99% |
| 8 | 21.5 | 100% |
| 16 | 21.4 | 100% |
| 32 | 15.3 | 71% |

**Best**: `-t 8` at 21.5 tok/s
**Slowest tested**: `-t 1` at 12.5 tok/s (1.72x spread)
**Against the physical-core default** (`-t 8`, 21.5 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

- **Vị trí điểm gãy (Knee of the curve)**: Điểm gãy rõ ràng xuất hiện tại khoảng **4 đến 8 luồng** (đạt đỉnh 21.5 tok/s, tương ứng với số nhân vật lý 8 physical cores của AMD Ryzen 7). Từ 1 luồng lên 4 luồng, tốc độ tăng vọt từ 12.5 lên 21.4 tok/s (tăng 1.71×).
- **Hiện tượng bão hòa (Plateau từ 4 đến 16 luồng)**: Từ 4 luồng đến 16 luồng (số nhân ảo SMT), tốc độ đi ngang hoàn toàn (~21.4 - 21.5 tok/s). Cơ chế là do giai đoạn decode bị nghẽn bởi **băng thông bộ nhớ (memory bandwidth)**, 4–8 luồng CPU đã bão hòa hoàn toàn kênh truyền RAM (dual-channel DDR5); do đó việc tăng thêm luồng không thể kéo thêm dữ liệu từ RAM nhanh hơn. Đồng thời, các luồng ảo SMT (luồng 9-16) chia sẻ chung đơn vị thực thi và bộ đệm L1/L2 của nhân vật lý nên không mang lại thêm thông lượng.
- **Hiện tượng sụt giảm do Oversubscription (32 luồng)**: Khi ép lên 32 luồng (vượt gấp đôi số core logical), tốc độ giảm mạnh xuống còn 15.3 tok/s (giảm gần 30%). Nguyên nhân là do chi phí chuyển ngữ cảnh (context switching overhead) của hệ điều hành, tranh chấp tài nguyên và hiện tượng cache thrashing khi quá nhiều luồng tranh giành nhau.
- **Knob tối ưu**: Cấu hình mặc định `-t 8` (bằng đúng số nhân vật lý) là lựa chọn tối ưu nhất cho máy này.