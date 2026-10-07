# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 120 | 2.00 | 3700 | 6000 | 6600 | 7.8 | 0.0% |
| 50 | 100 | 1.67 | 27000 | 31000 | 31000 | 37.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.84x** (17% of linear) |
| P95 latency | **5.17x** |
| Effective concurrency at 50 users | 37.6 vs `--parallel 4` slots (occupancy/slot ratio 9.41) |

**Saturated.** Throughput delivered only 0.84x for 5x the offered load, and effective concurrency (37.6) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.84x while P95 moved 5.17x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Nhận xét từ hai lần tải

### SLO và goodput giữ được

Chọn mục tiêu dịch vụ **P95 ≤ 10 giây**; tính goodput@10s bằng số request thành công có latency **≤ 10 giây** mỗi giây. P95 là tiêu chí của cả phân phối, còn goodput đếm từng request đạt ngưỡng.

| Mức tải | Đạt SLO P95 ≤ 10s? | Request hoàn thành đạt ≤ 10s | Goodput@10s |
|:--|:--|:--|:--|
| 10 user | Có, P95 = 6s | 120/120 (100%); max = 7,322s, không lỗi | 2,003 req/s, bằng throughput |
| 50 user | Không, P95 = 31s | Không có số chính xác; tối đa 50/100 vì P50 = 27s | Không quá 0,836 req/s (≤ 50% throughput) |

Giới hạn trên ở 50 user được suy ra từ median, không phải goodput đo trực tiếp. CSV chỉ chứa thống kê tổng hợp, không có latency từng request/histogram để đếm chính xác số request ≤ 10s. Các con số áp dụng cho request hoàn thành trong cửa sổ test; không bao gồm request còn chờ khi Locust dừng. Không thể coi việc vi phạm SLO P95 là goodput bằng 0.

### Queue time và compute time

`37,64 = 1,67276 req/s × 22,49909s`, tương đương **9,41 lần số slot**; tỷ lệ này là occupancy ước tính gồm cả chờ, không phải GPU utilization hay batch width. Gauge `requests_deferred=45` chứng minh trực tiếp có hàng đợi, cùng `requests_processing=4` và busy-slots gần 4 cho thấy các slot bận. Vì thế phần latency tăng có đóng góp của queue time; chưa thể quy toàn bộ mức tăng P95 cho queue time hoặc lấy hai P95 trừ nhau để đo queue time. Prefill/decode và scheduling dưới tải cũng có thể thay đổi. Ngược lại, effective concurrency ≤ 4 tự nó không chứng minh hoàn toàn không có chờ: đây là trung bình từ một lần chạy hữu hạn, không phải trace từng request.

### Kết luận và knob ưu tiên

Tăng từ 10 lên 50 user, throughput giảm từ 2,00 xuống 1,67 req/s (0,84×), trong khi P95 tăng từ 6 lên 31 giây (5,17×). Metrics của lần 50 user có peak busy slots 3,98/4, processing 4 và deferred 45: server có batching nhưng hàng đợi vẫn lớn. Có bằng chứng bão hòa rõ ở 50 user; với chỉ hai mức tải, chưa xác định được chính xác ngưỡng bắt đầu bão hòa. Mức 10 user cũng có effective concurrency 7,8 vượt 4 slot, gợi ý đã có chờ đợi.

"5× offered load" trong bảng là 5× số user ảo, không phải tốc độ request đến cố định tăng 5×: Locust dùng vòng kín, mỗi user chờ request hoàn thành rồi mới nghỉ và gửi tiếp. Effective concurrency gồm cả chờ slot, không phải độ rộng batch. Số liệu chỉ gồm request hoàn thành trước khi dừng, nên 0 lỗi HTTP không có nghĩa toàn bộ request đã gửi đều hoàn thành trong 60 giây.

Nếu chọn SLO minh họa P95 ≤ 10 giây, lần 10 user đạt còn lần 50 user không đạt. Không tính được goodput chính xác theo SLO mỗi request từ các percentile tổng hợp này. Bước thử tiếp theo hợp lý là giới hạn số request được nhận đồng thời/độ dài hàng đợi, rồi đo lại số request đáp ứng SLO và số request bị từ chối; điều này nhằm giảm chờ, không làm tăng năng lực GPU. Chưa ưu tiên tăng thread CPU vì sweep trước gần phẳng; tăng slot cũng cần đo lại VRAM/context và latency trước khi kết luận.
