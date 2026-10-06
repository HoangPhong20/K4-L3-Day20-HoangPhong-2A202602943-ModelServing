# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Hoang Phong
**MSSV:** 2A202602943
**Cohort:** AI20K-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10, AMD64
- **CPU:** Intel Core i7-1065G7
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 15.8 GB
- **Accelerator:** NVIDIA GeForce MX230 (2 GB); CUDA and Vulkan detected
- **llama.cpp asset đã tải:** prebuilt Windows AMD64 binary, build b10488
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop Windows của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Setup chạy cục bộ với binary llama.cpp dựng sẵn, không compile. Windows PowerShell 5.1 cần UTF-8 BOM cho `lab.ps1`; khi in bảng có Unicode, đặt `$env:PYTHONUTF8='1'` để tránh lỗi encoding. Máy có đủ RAM cho Gemma 4 E2B.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 7433 | 873 / 3233 | 151.0 / 667.6 | 10316 / 43790 / 43790 | 6.6 |
| UD-Q2_K_XL | 2.24 | 8536 | 1435 / 4172 | 858.4 / 938.3 | 49649 / 62268 / 62268 | 1.2 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn 0.73 GB nhưng decode chậm hơn 5.5× trong lần đo này. Hai câu trả lời smoke ngắn tương tự; một câu hỏi chưa đủ đánh giá chất lượng tổng quát. Tôi chọn Q4 vì RAM đủ và tốc độ đo được tốt hơn; tiết kiệm dung lượng của Q2 chưa đáng đánh đổi.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.14 | 13000 | 47000 | 47000 | 3.5 | 0.0% |
| 50 | 0.14 | 14000 | 51000 | 51000 | 3.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.94×
- **P95 tăng:** 1.09×
- **Effective concurrency ở 50 users:** 3.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): **3.62 / 4 slots**

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Ở 50 users, throughput bằng 0.94× run 10 users dù offered load tăng 5×. P95 tăng 1.09×; metrics ghi nhận 4 requests xử lý và tới 46 deferred, chứng minh có queueing. Mỗi run chỉ hoàn tất 7 requests nên percentile còn nhiễu. Tôi sẽ thử giảm `max_tokens` để giải phóng slot sớm hơn, rồi đo lại goodput với SLO P95 60 giây.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Chỉ chạy localhost; chưa nối cluster/Compose | stub |
| N17 Data pipeline | Không có job; tài liệu mẫu nằm trong bộ nhớ | stub |
| N18 Lakehouse | `TOY_DOCS` in-memory; chưa nối lakehouse | stub |
| N19 Vector + features | `TOY_DOCS` + keyword overlap; chưa có vector index/feature store | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: **0.0 ms**
- retrieve: **0.1 ms**
- llm: **9189.7 ms**
- **stage chiếm nhiều nhất:** **llm** (**100%** của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck như dự đoán, trung bình 9.19 giây mỗi query. Muốn giảm tổng latency 2×, tôi sẽ giảm số token sinh ra nếu câu trả lời ngắn hơn vẫn đáp ứng yêu cầu, hoặc dùng model/backend nhanh hơn. Tối ưu retrieval chỉ tiết kiệm khoảng 0.1 ms nên hầu như không đổi tổng thời gian.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** đổi quantization từ UD-Q2_K_XL sang UD-Q4_K_XL

```
before:  1.2 tok/s decode (UD-Q2_K_XL)
after:   6.6 tok/s decode (UD-Q4_K_XL)
speedup: 5.50×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Trong benchmark 10 request cho mỗi quantization, Q4 có decode throughput cao hơn Q2 5.5× dù file lớn hơn 0.73 GB. Ít bit giảm lượng dữ liệu đọc, nhưng kernel phải giải mã các giá trị nén; chi phí dequantization và cách triển khai kernel có thể làm lợi ích dung lượng không chuyển thành tốc độ.

Đây là giả thuyết về cơ chế, chưa phải kết luận đã được profiler kiểm chứng. Nhiệt độ, paging hoặc tải nền cũng có thể ảnh hưởng kết quả. Vì vậy tôi chọn Q4 dựa trên số liệu hiện có và sẽ lặp lại phép đo trong điều kiện máy ổn định trước khi khẳng định nguyên nhân hay mức speedup lâu dài.

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

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

ChatGPT: hỗ trợ đọc checkpoint/rubric, xử lý lỗi PowerShell, tổng hợp các số liệu đã
đo vào bản nháp Reflection và kiểm tra danh sách file nộp. Tôi cần đọc lại và tự giải
thích được các nhận xét về hiệu năng trước khi nộp.
