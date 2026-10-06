# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **10 physical · 10 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
| :----------- | ------------: | ------: |
| 1            |          47.5 |     96% |
| 5            |          49.3 |    100% |
| 10           |          46.5 |     94% |
| 20           |          33.4 |     68% |

**Best**: `-t 5` at 49.3 tok/s
**Slowest tested**: `-t 20` at 33.4 tok/s (1.47x spread)
**Against the physical-core default** (`-t 10`, 46.5 tok/s): 1.06x

Use this in your run:

```bash
LAB_N_THREADS=5 make bench
```

## Your explanation


Đường cong hiệu năng gần như đi ngang từ 1 đến 10 thread (dao động 46.5 - 49.3 tok/s), đạt đỉnh (knee) ở mốc 5 thread, sau đó tụt dốc mạnh xuống 33.4 tok/s ở mốc 20 thread.

Kết quả đi ngang này trái với kỳ vọng thông thường (tăng dần và đạt đỉnh ở physical core) do đặc thù kiến trúc Apple Silicon M4 và cấu hình offload:

1. **Giới hạn băng thông bộ nhớ trên Unified Memory:** Model đã được offload toàn bộ lên GPU (`ngl=99` qua Metal). Quá trình sinh token (decode) vốn bị giới hạn bởi băng thông bộ nhớ (memory-bandwidth-bound), và GPU đang đảm nhận khối lượng tính toán đó trên bộ nhớ thống nhất. CPU lúc này chỉ lo việc điều phối và sampling. Việc có 1 hay 10 luồng CPU không thay đổi được nút thắt cổ chai về băng thông, nên tốc độ gần như không suy xuyển.
2. **Oversubscription (Cấp phát luồng quá mức):** Khi đẩy lên 20 thread — gấp đôi số core vật lý của máy — tốc độ lập tức tụt hơn 30%. Các luồng thừa này không có thêm dữ liệu để xử lý nhưng lại ép bộ lập lịch (OS scheduler) của macOS phải liên tục context-switch (chuyển ngữ cảnh)^^. Sự tranh giành tài nguyên này sinh ra độ trễ vô ích và trực tiếp kéo lùi toàn bộ quá trình decode.
