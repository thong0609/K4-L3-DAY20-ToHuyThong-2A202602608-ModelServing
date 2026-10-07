# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **12 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 80.4 | 97% |
| 6 | 81.3 | 98% |
| 12 | 82.8 | 100% |
| 16 | 81.2 | 98% |
| 32 | 82.0 | 99% |

**Best**: `-t 12` at 82.8 tok/s
**Slowest tested**: `-t 1` at 80.4 tok/s (1.03x spread)
**Against the physical-core default** (`-t 12`, 82.8 tok/s): 1.00x

Use this in your run:

```powershell
$env:LAB_N_THREADS = '12'
.\lab.ps1 bench
```

## Giải thích kết quả trên máy này

Lần sweep ngày 07/10/2026 chạy `./lab.ps1 tune`, model Q4, `ngl=99`, decode `tg128`, 2 lần lặp mỗi điểm, theo thứ tự 1, 6, 12, 16, 32 thread. Runtime nhận RTX 3050 Laptop qua CUDA. Đây là benchmark decode trực tiếp bằng `llama-bench`, không phải phép đo HTTP TTFT/TPOT của `bench`; không lấy hai loại throughput này chia nhau để tính speedup.

**Knee:** không thấy knee rõ trong lưới đã đo. Đường cong gần phẳng ngay từ 1 thread: 80,37 tok/s, bằng 97,1% mức tốt nhất. Đỉnh quan sát nằm ở 12 thread (82,75 tok/s), nhưng 16 và 32 thread vẫn đạt 98,1% và 99,1% đỉnh. Không có sự sụt mạnh ở mức oversubscribe 32 thread; không nên gọi 12 là knee chỉ vì đó là điểm cao nhất.

**Before/after thực:** tăng `-t 1` lên `-t 12` đạt 80,37 → 82,75 tok/s, tức 1,0296× (+2,96%). Đây là so sánh hai điểm trong cùng sweep, không phải cải thiện so với cấu hình mặc định. Mặc định đã là 12 thread, nên speedup so với mặc định là **1,00×**. Giữ 12 thread cho các bước tiếp theo; chưa cần chạy lại baseline vì cấu hình tốt nhất trùng cấu hình baseline đang có.

**Cơ chế có thể giải thích:** với CUDA và `ngl=99`, phần tính toán được offload sang GPU có thể chi phối decode; tăng số worker CPU không làm tăng băng thông VRAM hay khả năng xử lý của GPU. Điều này phù hợp với đường cong gần phẳng, khác kỳ vọng của một sweep CPU-only. Khi dư worker, các thread CPU có thể tranh thời gian chạy trên core, cache và băng thông RAM, đồng thời phát sinh chi phí đồng bộ/lập lịch; tuy nhiên lần đo này không cho thấy tổn thất lớn do oversubscription, nên không khẳng định đây là bottleneck đã được chứng minh.

**Giới hạn:** chỉ 2 lần lặp mỗi điểm, báo cáo không lưu độ lệch chuẩn; tải nền, xung/ nhiệt độ và thứ tự đo có thể ảnh hưởng chênh lệch nhỏ. Chưa thể khẳng định +2,96% là lợi ích ổn định hay GPU đã chạm trần bandwidth thay vì compute. Kết quả đáng ghi nhận là tune thread CPU không tạo cải thiện đáng kể so với mặc định trong cấu hình CUDA này.
