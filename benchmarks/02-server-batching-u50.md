# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.98 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 14358 |

Highest sampled value was **3.98 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Nhận xét từ lần tải 50 user

Lần chạy ngày 07/10/2026: Locust 50 user hoạt động khoảng 10:50:47–10:51:46 (UTC+7). CSV metrics có 15 mẫu từ 10:51:08,5 đến 10:52:05,7, nên có khoảng 38 giây lấy mẫu chồng trực tiếp với tải; những mẫu cuối quan sát server xử lý nốt hàng đợi sau khi Locust dừng. Script nghỉ 2 giây sau mỗi scrape, không bảo đảm khoảng cách giữa hai timestamp đúng 2 giây; thời gian HTTP cộng thêm làm số mẫu thấp hơn 30.

Peak `n_busy_slots_per_decode` thực là **3,98062/4**, làm tròn **3,98/4** (99,5%). Đây là giá trị trung bình busy slots trên mỗi bước decode cao nhất đã lấy mẫu, không phải độ rộng batch tức thời tối đa. So với smoke một request có gauge bằng 1, mức gần 4 dưới tải là bằng chứng server gộp nhiều request vào các bước decode chung. Nó không chứng minh batching làm throughput tăng trong mọi điều kiện; ở đây throughput của lần 50 user còn thấp hơn lần 10 user.

`requests_processing` đạt 4 và `requests_deferred` đạt **45**, cho thấy các slot đã bận và có hàng đợi. P95 của lần 50 user là 31 giây, so với 6 giây ở lần 10 user; hàng đợi góp phần vào độ trễ này, nhưng các gauge không tách chính xác bao nhiêu giây là queue time hay compute time. `kv_cache_usage_ratio` giữ **n/a** vì build không xuất metric đó. Counter 14358 là tổng tích lũy từ khi server khởi động, gồm cả smoke và lần tải trước, không phải riêng token của lần 50 user.

Trong [02-server-results.md](02-server-results.md), effective concurrency = RPS × latency trung bình khoảng **37,6**, cao hơn 4 slot. Hai số không mâu thuẫn: 37,6 ước tính request trong hệ thống gồm cả request chờ; busy-slots đo công việc đang decode trong server. Để chứng minh batching, dùng gauge server; để đánh giá tổng số request đang xử lý hoặc chờ, dùng concurrency với giới hạn của phép ước tính. Lần tải chỉ 60 giây, có ramp-up và request chưa hoàn thành khi dừng, nên không coi 37,6 là trung bình chính xác của một trạng thái ổn định.
