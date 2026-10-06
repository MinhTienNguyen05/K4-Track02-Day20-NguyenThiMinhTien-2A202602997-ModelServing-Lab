# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=99`

| Users | Reqs |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
| :---- | ---: | ---: | -------: | -------: | -------: | ---------------: | -------: |
| 10    |   64 | 1.13 |     7600 |    11000 |    11000 |              8.5 |     0.0% |
| 50    |   70 | 1.19 |    30000 |    43000 |    46000 |             33.5 |     0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users         |                                                           |
| :-------------------------------- | --------------------------------------------------------: |
| Offered load                      |                                                        5x |
| Throughput actually delivered     |                           **1.06x** (21% of linear) |
| P95 latency                       |                                           **3.91x** |
| Effective concurrency at 50 users | 33.5 vs`--parallel 4` slots (occupancy/slot ratio 8.38) |

**Saturated.** Throughput delivered only 1.06x for 5x the offered load, and effective concurrency (33.5) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.06x while P95 moved 3.91x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading


Server của tôi bão hòa ở mức dưới 50 users. Bằng chứng rõ ràng nhất là khi tải đầu vào (offered load) tăng gấp 5 lần, thông lượng (RPS) gần như đi ngang khi chỉ nhích thêm 1.06x (từ 1.13 lên 1.19 RPS). Trong khi đó, độ trễ P95 lại tăng vọt gấp gần 4 lần (từ 11.000 ms lên 43.000 ms).

Con số thuyết phục nhất chính là **Effective concurrency đạt 33.5** trong khi cấu hình server chỉ giới hạn ở `--parallel 4`. Tỷ lệ lấp đầy (occupancy ratio) lên tới 8.38 cho thấy tại thời điểm 50 users, có gần 30 request hoàn toàn không được xử lý mà chỉ nằm kẹt trong hàng đợi. Toàn bộ phần thời gian tăng thêm của P95 chính là queue time (thời gian chờ) chứ không phải compute time (thời gian tính toán).

Để cải thiện goodput@SLO (ví dụ: giữ P95 ở mức dưới 15 giây), knob đầu tiên tôi sẽ điều chỉnh là tăng `--parallel` (chẳng hạn lên 8 hoặc 16). Lý do là vì chip M4 sở hữu lợi thế lớn về băng thông bộ nhớ thống nhất (unified memory) và hệ thống 10 nhân xử lý; việc mở rộng số slot giúp engine gộp được nhiều request hơn vào chung một bước decode (continuous batching), qua đó trực tiếp giải phóng hàng đợi và chuyển queue time thành compute time hiệu quả.
