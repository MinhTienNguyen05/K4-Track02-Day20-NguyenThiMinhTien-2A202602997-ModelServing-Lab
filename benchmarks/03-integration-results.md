# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query                                           |      Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
| :---------------------------------------------- | ----------------------: | ---------: | ------------: | -------: | ---------: |
| Why is goodput more useful than raw throughp... |   goodput, paged, radix |        0.0 |           0.0 |   1009.4 |     1009.4 |
| What problem does PagedAttention actually so... |    paged, radix, disagg |        0.0 |           0.0 |    983.5 |      983.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching |        0.0 |           0.0 |    996.1 |      996.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **996.3** · total **996.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.

## Which N16-N19 pieces are real


* **N16 (Cloud/IaC):** Stub (chỉ chạy localhost).
* **N17 (Data pipeline):** Stub (dùng danh sách in-memory).
* **N18 (Lakehouse):** Stub (dùng biến từ điển `TOY_DOCS` giả lập).
* **N19 (Vector + features):** Stub (tính điểm từ khóa trùng khớp - keyword overlap thay vì vector search thực thụ, thời gian `embed` là 0.0ms).
* **N20 (Serving):** Real (sử dụng `llama-server` gọi API thật qua port 8080).

Khâu chiếm phần lớn thời gian (Dominant stage) là **LLM (100%, chiếm khoảng 996.3 ms)**. Kết quả này hoàn toàn khớp với kỳ vọng, vì các khâu Embedding và Retrieval đang được stub bằng dữ liệu cứng trên RAM nên trả về kết quả gần như tức thời (0.0 ms). Khâu duy nhất thực sự tiêu tốn năng lực tính toán và gọi qua network cục bộ là quá trình prefill và decode của LLM.

Nếu phải giảm độ trễ của pipeline này đi một nửa (giảm 2x), tôi sẽ tấn công trực tiếp vào khâu LLM. Có hai chiến lược chính:

1. **Áp dụng Prefix Caching:** Tái sử dụng KV cache cho phần System Prompt dùng chung, giúp loại bỏ hoàn toàn chi phí prefill dư thừa ở mỗi request (hiện đang tốn khoảng 250 - 330ms).
2. **Sử dụng model nhỏ hơn:** Nếu ứng dụng cho phép, chuyển sang sử dụng Qwen3.5 0.8B để tăng tốc độ token/giây (TPOT) ở khâu decode. Vì tốc độ sinh chữ đang bị giới hạn bởi băng thông bộ nhớ (memory bandwidth), việc giảm khối lượng tham số là cách trực tiếp nhất để cắt đôi thời gian xử lý.
