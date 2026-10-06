# Bonus B5 - MLX vs llama.cpp Metal

Host `Darwin-arm64` · arm ·
llama.cpp `b10488` · 10 prompts,
`max_tokens=64`, warm-up discarded on both sides

| Runtime           |                                Weights | TTFT P50 (ms) | TTFT P95 (ms) | Decode (tok/s) |
| :---------------- | -------------------------------------: | ------------: | ------------: | -------------: |
| llama.cpp (Metal) |     `gemma-4-E2B-it-UD-Q4_K_XL.gguf` |         129.3 |         216.5 |           42.7 |
| MLX-LM            | `unsloth/gemma-4-E2B-it-UD-MLX-4bit` |          52.8 |          57.9 |           49.3 |

MLX decode is **1.15x** llama.cpp Metal here. Faster on decode: **MLX**.

Both sides run the same model at 4-bit and stream token by token, so the gap is a
runtime difference, not a model difference. The quantization schemes are not
byte-identical, though (Unsloth Dynamic GGUF vs MLX 4-bit), so treat a gap under
~10% as noise rather than a finding.

## Your finding

Dựa trên các chỉ số đo kiểm, MLX thể hiện sức mạnh vượt trội trên phần cứng Apple Silicon. Khung thời gian chờ token đầu tiên (TTFT P50) được rút ngắn hơn một nửa (từ 129.3ms xuống 52.8ms), và tốc độ decode nhanh hơn 1.15 lần (49.3 tok/s so với 42.7 tok/s) nhờ khả năng khai thác tối đa cấu trúc bộ nhớ thống nhất (Unified Memory).

Tuy nhiên, nếu xét từ góc độ thiết kế hạ tầng và kỹ thuật dữ liệu, `llama.cpp` lại là lựa chọn ưu việt hơn để thực sự đưa vào triển khai (ship) thực tế.

Thứ nhất, `llama.cpp` hoạt động dưới dạng một khối binary C/C++ độc lập mang lại tính khả chuyển cao. Cấu trúc này rất lý tưởng để đóng gói vào các container Docker cực nhẹ, triển khai trực tiếp lên các cụm Kubernetes, và dễ dàng giao tiếp với các luồng backend Go đang xử lý stream data mà không cần phải gánh theo toàn bộ hệ sinh thái Python cồng kềnh như MLX.

Thứ hai là bài toán khóa trong hệ sinh thái (vendor lock-in). MLX bị trói buộc hoàn toàn vào phần cứng của Apple. Khi kiến trúc hệ thống cần mở rộng và di chuyển từ máy trạm M4 cục bộ lên các hạ tầng cloud nhiều tier (như AWS) chạy Linux với kiến trúc GPU khác (ví dụ: NVIDIA CUDA), `llama.cpp` cho phép bảo toàn 100% cấu trúc tương tác API và logic continuous batching. Ngược lại, việc chọn MLX từ đầu sẽ buộc hệ thống phải đập bỏ và viết lại toàn bộ mã nguồn phục vụ inference khi thay đổi nền tảng hạ tầng.
