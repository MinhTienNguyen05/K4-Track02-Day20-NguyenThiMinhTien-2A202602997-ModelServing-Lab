# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge                                    |                              Peak observed |
| :--------------------------------------- | -----------------------------------------: |
| `n_busy_slots_per_decode` (avg/decode) |                      3.96 of 4 slots (99%) |
| `requests_processing`                  |                                          4 |
| `requests_deferred`                    |                                         46 |
| `kv_cache_usage_ratio`                 | n/a — not exported by llama.cpp`b10488` |
| `tokens_predicted_total` (final)       |                                       7618 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Dưới đây là phần phân tích cơ chế batching so với định lý Little's Law. Bạn hãy copy toàn bộ đoạn văn này và ghi đè lên dòng `_What was the peak batch width..._` trong file `benchmarks/02-server-batching-u50.md` của bạn nhé:

Đỉnh batch width (peak `n_busy_slots_per_decode`) đo được là **3.96 trên 4 slot** (đạt 99% công suất cấu hình). Con số này **không khớp** với mức effective concurrency (33.5) đã tính bằng định lý Little's Law trong file `02-server-results.md`.

Tôi tin tưởng vào **cả hai con số** vì chúng đo lường hai khái niệm hoàn toàn khác nhau trong lý thuyết hàng đợi:

1. **Peak batch width (3.96):** Đo lường *mức độ sử dụng (utilization)* của engine tính toán^^. Engine đã bị giới hạn cứng bởi cấu hình `--parallel 4`, nên bất kể người dùng gọi nhiều đến đâu, hệ thống cũng không thể nhồi nhiều hơn 4 request vào một bước tính toán decode.
2. **Effective concurrency (33.5):** Đo lường *tổng tải (occupancy)* đang nằm trong toàn bộ hệ thống, bao gồm cả request đang được tính toán VÀ request đang xếp hàng chờ.

Chỉ số `requests_deferred = 46` chính là bằng chứng liên kết hai con số này lại với nhau^^: Server đang gộp tối đa 4 request để xử lý (phản ánh qua batch width ~3.96 và `requests_processing` = 4), trong khi hàng chục request còn lại bị đẩy vào hàng đợi chờ (phản ánh qua `requests_deferred` = 46). Khoảng cách giữa 4 và 33.5 chính là nguyên nhân tạo ra phần P95 queue time khổng lồ mà ta quan sát thấy ở bài test trước.
