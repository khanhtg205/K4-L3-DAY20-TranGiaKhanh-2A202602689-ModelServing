# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 5876.3 | 5876.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 5321.8 | 5321.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 4305.0 | 4305.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **5167.7** · total **5167.8**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **Khai báo Real / Stub**:
  - **N16 Cloud/IaC**: Stub (chạy local trên Windows, chưa deploy Terraform/K8s).
  - **N17 Data pipeline**: Stub (sử dụng in-memory corpus mẫu).
  - **N18 Lakehouse**: Stub.
  - **N19 Vector + features**: Stub (retrieval dùng keyword overlap fallback, embed = 0.0 ms).
  - **N20 Serving**: **Real** (`llama-server` phục vụ endpoint thật trên port 8080).
- **Phân tích Bottleneck**: Giai đoạn LLM chiếm **100% tổng thời gian** (5167.7 ms / 5167.8 ms), hoàn toàn khớp với kỳ vọng vì giai đoạn keyword retrieve chỉ mất 0.1 ms trong khi LLM phải thực hiện prefill qua hàng trăm token context và decode sinh câu trả lời.
- **Chiến lược giảm latency 2×**: Bắt buộc phải tấn công vào stage **LLM**. Các biện pháp hiệu quả nhất gồm:
  1. **Prefix Caching**: Cache sẵn KV cache của system prompt cố định để triệt tiêu thời gian prefill lặp lại.
  2. **Rút gọn Context/Prompt Compression**: Chỉ đưa top-k chunk ngắn nhất vào prompt.
  3. **Speculative Decoding**: Sử dụng draft model nhỏ để tăng tốc độ decode.
