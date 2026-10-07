# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5784 | 92 / 227 | 13.6 / 13.9 | 948 / 1038 / 1038 | 73.6 |
| UD-Q2_K_XL | 2.24 | 4691 | 92 / 380 | 13.8 / 14.1 | 960 / 1250 / 1250 | 72.6 |

- **TTFT** đo phía client từ lúc gửi request đến token nội dung đầu tiên; bao gồm xử lý prompt và overhead HTTP/lập lịch, không chỉ riêng prefill.
- **TPOT** là thời gian decode trung bình mỗi token sau token đầu. `decode tok/s = 1000 / TPOT_p50`; tốc độ còn phụ thuộc kernel và chi phí giải lượng tử.
- `UD-Q2_K_XL` and `UD-Q4_K_XL` decode within 2% of each other here, for 0.73 GB difference on disk.

## Nhận xét từ số đo và so sánh chất lượng

Đây là lần chạy lại `./lab.ps1 bench` ngày 07/10/2026, sau lần baseline đã có và sau thử chất lượng. Không coi đây là lần cold-cache: weights đã được đọc trước đó, không xóa page cache. Mỗi bản bỏ một request warm-up, rồi đo đủ 10/10 request; giữ nguyên cấu hình mặc định trong bảng. Với nearest-rank và 10 mẫu, P95 và P99 cùng là mẫu lớn nhất, chưa đủ để mô tả ổn định đuôi phân phối.

**Tốc độ:** Q2 đạt 72,6 tok/s so với 73,6 tok/s của Q4: tỷ lệ 0,986x, tức chậm hơn khoảng 1,36%, nằm trong ngưỡng chênh lệch 2% của script. Chưa thấy lợi ích decode đáng kể từ Q2 ở lần đo này. TPOT P50 tăng từ 13,59 lên 13,78 ms/token; TTFT P50 gần như bằng nhau (92,0 và 91,7 ms), nhưng TTFT P95 của Q2 cao hơn (380,2 so với 227,1 ms).

**Dung lượng:** Q2 giảm từ 2,97 xuống 2,24 GiB (cột script ghi GB nhưng tính theo 1024^3), tiết kiệm khoảng 0,73 GiB, tương đương 24,5% theo kích thước file thực. Đây là dung lượng weights trên đĩa, không phải số đo RAM/VRAM khi chạy.

**Phần cứng và giới hạn kết luận:** máy i5-12500H, RAM 15,6 GB, RTX 3050 Laptop 4 GB, cấu hình `ngl=99`; probe xác nhận CUDA khả dụng. Không thể giải thích kết quả bằng việc máy không có GPU offload. Kernel, giải lượng tử, tải nền và trạng thái cache có thể ảnh hưởng; baseline này chưa đủ để kết luận máy bị giới hạn bởi compute hay bandwidth.

**Chất lượng:** đã gửi đúng cùng một câu hỏi cho hai model, chạy tuần tự trên port 8090 với `temperature=0`, `seed=42`, `max_tokens=256`, reasoning tắt. Đề cho request bắt đầu ở 0 ms, token đầu ở 200 ms, token thứ 41 kết thúc ở 1000 ms; yêu cầu tính TTFT, TPOT, E2E và giải thích vì sao 2-bit không luôn nhanh hơn 4-bit. Prompt và toàn bộ câu trả lời được lưu trong [01-quality-comparison.json](01-quality-comparison.json).

- Q4 tính đúng TTFT = 200 ms, TPOT = (1000 - 200)/(41 - 1) = 20 ms/token, E2E = 1000 ms; nêu overhead và phần cứng có thể làm Q2 không nhanh hơn. Câu trả lời kết thúc tự nhiên (`finish_reason=stop`, 246 token).
- Q2 tính đúng TTFT và E2E nhưng gọi sai TPOT là "Token Processing Time", lấy 800 ms mà không chia cho 40 token. Phần giải thích cuối bị cắt bởi giới hạn 256 token (`finish_reason=length`); giới hạn này ảnh hưởng đánh giá độ đầy đủ, nhưng lỗi công thức xuất hiện rõ trước khi bị cắt.

**Lựa chọn cho máy này:** ưu tiên Q4: decode gần tương đương, còn bài kiểm tra kỹ thuật trên cho đáp án đúng và đầy đủ hơn. Q2 đáng cân nhắc khi cần tiết kiệm khoảng 0,73 GiB weights và chấp nhận kiểm tra đầu ra kỹ hơn. Đây là nhận xét từ một câu hỏi, không phải kết luận chất lượng tổng quát; người làm lab nên đọc hai câu trả lời để xác nhận đánh giá của mình.
