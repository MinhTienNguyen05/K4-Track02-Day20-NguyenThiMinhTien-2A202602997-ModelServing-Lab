# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Thị Minh Tiến
**MSSV:** 2A202602997
**Cohort:** A20-K4
**Ngày submit:** 6/10/2026

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS
- **CPU:** Apple M4
- **Cores:** 10 physical / 10 logical_
- **CPU extensions:** NEON
- **RAM:** 16.0 GB
- **Accelerator:**  Apple Metal
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (LAB_MODEL=gemma4-e2b)
- **Quantization:**  UD-Q4_K_XL + UD-Q2_K_XL)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Gặp lỗi SSL CERTIFICATE_VERIFY_FAILED khi script Python tải runtime do hệ điều hành thiếu chứng chỉ gốc. Đã khắc phục bằng cách chạy Install Certificates.command trong thư mục Python của macOS để cấp quyền tải, sau đó make setup diễn ra bình thường.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| ------------ | --------- | --------- | ----------------- | ----------------- | -------------------- | -------------- |
| UD-Q4_K_XL   | 2.97      | 3121      | 126 / 260         | 22.2 / 27.9       | 1520 / 1916 / 1916   | 45.1           |
| UD-Q2_K_XL   | 2.24      | 3075      | 126 / 345         | 17.3 / 23.6       | 1242 / 1645 / 1645   | 57.9           |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Quan sát (≤ 60 chữ): Bản 2-bit decode nhanh hơn 1.28 lần (57.9 vs 45.1 tok/s) và nhẹ hơn 0.73 GB. TTFT P50 ngang nhau vì bị giới hạn bởi compute. Khi test qua API, bản 2-bit trả lời kém chi tiết hơn. Với 16GB RAM, dùng 4-bit là tối ưu vì đã đủ nhanh và giữ được chất lượng tốt.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS  | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
| ----- | ---- | -------- | -------- | -------- | ---------------- | -------- |
| 10    | 1.13 | 7600     | 11000    | 11000    | 8.5              | 0.0%     |
| 50    | 1.19 | 30000    | 43000    | 46000    | 33.5             | 0.0%     |

- **Offered load tăng 5×, throughput thực tăng:** 1.06×
- **P95 tăng:** 3.91×
- **Effective concurrency ở 50 users:** 33.5 so với \--parallel` = 4 slots`

**Peak llamacpp:n_busy_slots_per_decode** (từ make metrics khi make load-50 đang chạy): 3.96 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

- Offered load tăng 5×, throughput thực tăng: 1.06×
- P95 tăng: 3.91×
- Effective concurrency ở 50 users: 33.5 so với --parallel = 4 slots
- Peak llamacpp:n_busy_slots_per_decode (từ make metrics khi make load-50 đang
  chạy): 3.96 / 4 slots
- Saturation reading (≤ 80 chữ): Server bão hòa dưới mức 50 users. Bằng chứng: RPS chỉ tăng 1.06x nhưng P95 tăng vọt 3.91x (lên 43s) và effective concurrency (33.5) vượt xa số 4 slot định mức. Phần latency tăng thêm hoàn toàn là queue time. Để nâng goodput@SLO, tôi sẽ tăng --parallel (lên 8) trước để tận dụng RAM M4, gộp thêm request vào chung một batch decode.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day                   | Piece            | Real hay stub? |
| --------------------- | ---------------- | -------------- |
| N16 Cloud/IaC         | stub             |                |
| N17 Data pipeline     | stub             |                |
| N18 Lakehouse         | stub             |                |
| N19 Vector + features | stub             |                |
| N20 Serving           | `llama-server` | real           |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 996.3 ms
- **stage chiếm nhiều nhất:**  llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck nằm 100% ở LLM đúng như kỳ vọng vì các khâu embed/retrieve đang giả lập in-memory. Để giảm 2x latency, tôi sẽ dùng Prefix Caching tái sử dụng system prompt (giảm mạnh thời gian prefill) hoặc chuyển sang model Qwen3.5 0.8B để tăng tốc TPOT.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Điều chỉnh số luồng -t từ 10 (mặc định theo physical cores) xuống 5.

```
before:  46.5 tok/s
after:   49.3 tok/s
speedup: 1.06x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Đường cong hiệu năng (knee) đạt đỉnh ở 5 thread và bắt đầu suy giảm khi tiến tới 10 thread, đặc biệt rớt thảm hại xuống 33.4 tok/s khi bị ép lên 20 thread (oversubscription). Điều này hoàn toàn đi ngược lại với trực giác thông thường là "càng nhiều core càng nhanh", nhưng lại minh chứng rõ ràng cho nguyên lý giới hạn băng thông bộ nhớ (memory-bandwidth-bound) của quá trình decode.

Trên chip Apple M4 với kiến trúc Unified Memory, toàn bộ model đã được offload lên GPU (ngl=99). CPU lúc này chỉ làm nhiệm vụ điều phối và sampling. Băng thông bộ nhớ là nút thắt cổ chai vật lý; việc gán 10 hay 20 luồng CPU không mở rộng được băng thông này mà còn ép hệ điều hành (macOS scheduler) phải liên tục context-switch giữa các luồng. Sự tranh chấp tài nguyên ảo này sinh ra độ trễ vô ích và trực tiếp kéo lùi toàn bộ quá trình decode. Mốc 5 thread là điểm cân bằng lý tưởng nhất để CPU điều phối mà không tự tạo ra overhead.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [X] `hardware.json` committed
- [X] `models/active.json` committed
- [X] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [X] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [X] `benchmarks/02-server-results.md` committed (`make load-report`)
- [X] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [X] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [X] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [X] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
  đã được thay bằng nhận xét của bạn
- [X] 5 screenshots trong `submission/screenshots/`
- [X] `make verify` → **exit 0**
- [X] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [X] Repo GitHub ở chế độ **public**
- [X] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [X] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng Gemini làm Thought Partner để phân tích cơ chế memory bandwidth (giải thích hiện tượng oversubscription trên Apple M4), lý giải nguyên lý hoạt động của định lý Little's Law, và hướng dẫn xử lý lỗi chứng chỉ SSL trên macOS.
