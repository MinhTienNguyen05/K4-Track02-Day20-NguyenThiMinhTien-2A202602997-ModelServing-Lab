# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| :----------- | --------: | --------: | ----------------: | ----------------: | -------------------: | -------------: |
| UD-Q4_K_XL   |      2.97 |      3121 |         126 / 260 |       22.2 / 27.9 |   1520 / 1916 / 1916 |           45.1 |
| UD-Q2_K_XL   |      2.24 |      3075 |         126 / 345 |       17.3 / 23.6 |   1242 / 1645 / 1645 |           57.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.28x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

Bản 2-bit (UD-Q2_K_XL) cho tốc độ decode nhanh hơn **1.28 lần** (57.9 tok/s so với 45.1 tok/s) và tiết kiệm **0.73 GB** dung lượng lưu trữ/RAM so với bản 4-bit. Điểm đáng chú ý là thời gian TTFT (prefill) ở mức P50 của cả hai bản gần như tương đương nhau (126 ms). Điều này phản ánh đúng nguyên lý: quá trình prefill bị giới hạn bởi năng lực tính toán (compute-bound) của chip M4, trong khi quá trình decode sinh token (TPOT) bị giới hạn bởi băng thông bộ nhớ (memory-bandwidth-bound). Bản 2-bit phải di chuyển lượng dữ liệu ít hơn từ RAM vào chip cho mỗi token, dẫn đến TPOT giảm từ 22.2 ms xuống còn 17.3 ms.

Khi thử nghiệm thực tế qua API, bản 2-bit vẫn giữ được logic cơ bản nhờ cơ chế Unsloth Dynamic (giữ lại các layer nhạy cảm ở độ phân giải cao), nhưng có dấu hiệu suy giảm nhẹ về độ chi tiết và sự trôi chảy khi xử lý các câu hỏi phức tạp. Với cấu hình hệ thống hiện tại (16GB RAM) hoàn toàn dư dả cho model 2.97 GB, việc đánh đổi chất lượng để tăng một mức tốc độ vốn đã rất cao (45.1 tok/s) là không cần thiết. Bản 4-bit (UD-Q4_K_XL) vẫn là lựa chọn tối ưu để triển khai thực tế trên máy này.
