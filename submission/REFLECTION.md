# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Tô Huy Thông
**MSSV:** 2A202602608
**Cohort:** _A20-K4_
**Ngày hoàn thiện báo cáo:** 2026-10-07 (UTC+7);

---

## 1. Hardware & runtime _(rubric 1, 2 — 10 điểm)_

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows AMD64; probe Python 3.10 ghi `release=10` trong `hardware.json`.
- **CPU:** 12th Gen Intel Core i5-12500H
- **Cores:** 12 physical / 16 logical
- **CPU extensions:** Probe Windows không ghi trường AVX2/AVX-512; không có số đo trong `hardware.json` để khai báo.
- **RAM:** 15,6 GB theo probe
- **Accelerator:** NVIDIA GeForce RTX 3050 Laptop GPU, 4096 MiB VRAM; runtime CUDA. Probe cũng nhận Vulkan.
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip`, kèm CUDA runtime DLLs
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** primary `UD-Q4_K_XL` + compare `UD-Q2_K_XL`

**Chạy ở đâu:** laptop local Windows, không dùng Colab/Kaggle.

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

PowerShell đọc lỗi dấu gạch Unicode nên đổi sang ASCII, đặt UTF-8 cho Python và thêm kiểm tra mã thoát. Setup tạo `.venv`, cài package và runtime CUDA. Bộ tải Hugging Face bị chững; tải hai GGUF bằng curl rồi đối chiếu SHA-256 đều khớp. Chạy lại setup tạo manifest và báo Setup complete.

---

## 2. Đo lường _(rubric 3, 4, 5 — 20 điểm)_

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| ------------ | --------: | --------: | ----------------: | ----------------: | -------------------: | -------------: |
| UD-Q4_K_XL   |      2.97 |      5784 |          92 / 227 |       13.6 / 13.9 |    948 / 1038 / 1038 |           73.6 |
| UD-Q2_K_XL   |      2.24 |      4691 |          92 / 380 |       13.8 / 14.1 |    960 / 1250 / 1250 |           72.6 |

Lần chạy lại ngày 07/10/2026, sau khi weights đã được đọc; không phải cold-cache. Cả hai bản đủ 10/10 request, bỏ warm-up; `threads=12`, `ngl=99`, `ctx=2048`, `max_tokens=64`. Cột Size dùng đơn vị GiB (script ghi GB). Với 10 mẫu nearest-rank, P95/P99 cùng lấy mẫu lớn nhất.

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 chậm hơn 1,36%, nhỏ hơn 24,5% (khoảng 0,73 GiB). Cùng câu hỏi, Q4 tính TPOT đúng 20 ms/token; Q2 trả 800 ms, thiếu chia số token, và bị cắt ở giới hạn 256 token. Ưu tiên Q4; một câu hỏi chưa đại diện chất lượng tổng quát.

---

## 3. Serving under load _(rubric 8, 9, 10 — 20 điểm)_

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency |   Failures |
| ----: | ---: | -------: | -------: | -------: | ---------------: | ---------: |
|    10 | 2,00 |     3700 |     6000 |     6600 |              7,8 | 0/120 (0%) |
|    50 | 1,67 |    27000 |    31000 |    31000 |             37,6 | 0/100 (0%) |

- **Số user tăng 5×, tỷ lệ throughput sau/trước:** **0,84×** (giảm khoảng 16,5%). Đây là tải vòng kín, không phải tốc độ request đến cố định tăng 5×.
- **P95 tăng:** **5,17×**
- **Effective concurrency ở 50 users:** **37,6** so với `--parallel` = **4** slots; concurrency tính cả request chờ, không phải độ rộng batch.

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): **3,98 / 4** slots. Đây là trung bình busy slots trên mỗi decode cao nhất đã lấy mẫu, không phải batch tức thời tối đa. Peak `requests_deferred` = **45**; `kv_cache_usage_ratio` = **n/a** vì build không xuất gauge này.

**Bằng chứng:** [10 user](screenshots/04-locust-10.png), [50 user](screenshots/05-locust-50.png), [báo cáo batching](../benchmarks/02-server-batching-u50.md). Hai ảnh do người làm lab chụp từ output thật. Metrics có 15 mẫu, chồng khoảng 38 giây với lần tải 50 user; các mẫu cuối gồm giai đoạn xử lý nốt hàng đợi.

**Goodput với ngưỡng 10 giây/request, SLO P95 ≤ 10 giây:** 10 user đạt 120/120 request hoàn thành trong ngưỡng (max 7,322 giây), goodput **2,003 req/s**. 50 user có P50 = 27 giây, nên tối đa 50% request đạt ngưỡng: goodput **≤ 0,836 req/s**; đây là giới hạn trên, không phải số đo chính xác. CSV tổng hợp không đủ để đếm chính xác ở 50 user. Các số chỉ xét request hoàn thành trong cửa sổ test.

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Bão hòa rõ ở 50 user: RPS giảm 16,5%, P95 tăng 5,17×; processing đạt 4, deferred đạt 45. Hàng đợi góp phần tăng latency, nhưng chưa tách được queue/compute time. Với SLO P95 ≤ 10 giây, sẽ thử giới hạn request đồng thời/hàng đợi để giảm chờ, rồi đo lại goodput và số bị từ chối. Hai mức tải chưa xác định chính xác ngưỡng bão hòa.

---

## 4. Integration _(rubric 12, 13 — 15 điểm)_

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day                   | Piece                                               | Real hay stub?                |
| --------------------- | --------------------------------------------------- | ----------------------------- |
| N16 Cloud/IaC         | Laptop Windows local, không cluster/IaC             | Chưa tích hợp; local thay thế |
| N17 Data pipeline     | `TOY_DOCS`: 6 đoạn hard-code, không ingest/ETL      | stub                          |
| N18 Lakehouse         | Danh sách Python trong RAM, không lakehouse         | stub                          |
| N19 Vector + features | Keyword overlap top-3, không embedding/vector index | stub                          |
| N20 Serving           | `llama-server`                                      | real                          |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: **0,0 ms** (không gọi embedding server)
- retrieve: **0,1 ms**
- llm: **2719,3 ms**
- total: **2719,4 ms**
- **stage chiếm nhiều nhất:** **llm**, khoảng **100%** của total sau làm tròn.

Chạy ngày 07/10/2026; cả 3 query trả lời thật, context đứng đầu lần lượt `goodput`, `paged`, `disagg`. Chi tiết và câu trả lời tại [03-integration-results.md](../benchmarks/03-integration-results.md), số liệu gốc trong file JSON cùng tên.

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Stage llm chiếm gần 100%, đúng kỳ vọng với retrieval stub. Nhưng HTTP mất 2719 ms, còn prefill+decode chỉ 603 ms. Muốn giảm 2×, ưu tiên đo overhead kết nối, thử 127.0.0.1 và tái sử dụng client; chưa khẳng định nguyên nhân hay speedup. Tối ưu retrieve 0,1 ms không đáng kể.

---

## 5. The single change that mattered most _(rubric 11 — 10 điểm)_

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** thử tăng số thread CPU từ `-t 1` lên `-t 12` khi chạy Gemma 4 E2B Q4, giữ `ngl=99` (CUDA) và metric `tg128`. Số liệu từ cùng sweep ngày 07/10/2026, 2 lần lặp mỗi điểm; xem [báo cáo thread sweep](../benchmarks/01-tuning-tg128.md).

```
before:  80,37 tok/s (-t 1)
after:   82,75 tok/s (-t 12)
speedup: 1,0296× (+2,96%)
so với mặc định -t 12: 1,00× (không cải thiện)
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Sweep 1/6/12/16/32 thread lần lượt đạt 80,37/81,27/82,75/81,18/82,02 tok/s. Đường cong gần phẳng từ 1 thread, không có knee rõ; 12 thread là đỉnh quan sát, còn 32 thread vẫn đạt 99,1% đỉnh. Vì vậy không thể nói oversubscription gây tụt mạnh hay đổi thread đã cải thiện mặc định: mặc định vốn là 12. Trên máy dùng RTX 3050 với CUDA và `ngl=99`, phần decode được offload có thể chi phối thời gian; thêm worker CPU không tăng băng thông VRAM hoặc năng lực GPU. Đây là lời giải thích phù hợp với số đo, chưa phải chứng minh bottleneck GPU cụ thể.

Các thread CPU dư có thể tranh core, cache và băng thông RAM, đồng thời tăng đồng bộ/lập lịch; nhưng tác động đó không rõ trong sweep này. Chênh lệch tốt nhất–chậm nhất chỉ khoảng 3%, với 2 lần lặp mỗi điểm và không có độ lệch chuẩn trong báo cáo, nên chưa khẳng định speedup ổn định. Giữ 12 thread là lựa chọn hợp lý; finding chính là thread count chưa phải thay đổi đem lại lợi ích lớn trên cấu hình CUDA này. Không dùng throughput HTTP ở baseline để tính speedup của `llama-bench`.

---

## 6. Bonus _(optional — tối đa 10 điểm)_

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

Không làm bonus; không khai báo điểm bonus.

---

## 7. Điều làm bạn ngạc nhiên nhất _(optional)_

Thread sweep gần phẳng dù tăng từ 1 lên 32 thread. Đây là cấu hình CUDA, nên tăng worker CPU không nhất thiết tăng decode throughput.

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
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] Đủ 5 nhóm screenshots trong `submission/screenshots/`; nhóm 03 dùng hai ảnh `03a-serve.png` và `03b-smoke.png` theo hướng dẫn.
- [ ] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu: `K4-L3-DAY20-ToHuyThong-2A202602608-ModelServing`
- [x] Repo GitHub **public**, kiểm tra qua GitHub API không đăng nhập ngày 07/10/2026.
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI _(xem `docs/RULES.md` §3)_

Đã dùng Codex để hỗ trợ setup, sửa lỗi PowerShell/verify Windows, chạy benchmark, thread sweep, smoke, load test, metrics và pipeline trên máy này; đối chiếu câu trả lời hai quantization. Codex hỗ trợ soạn, giải thích và đối chiếu REFLECTION §1–§5 cùng các báo cáo. Phần lập luận có hỗ trợ AI.
