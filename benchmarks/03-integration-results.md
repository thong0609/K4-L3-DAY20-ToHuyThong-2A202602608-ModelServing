# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 2795.6 | 2795.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 2699.7 | 2699.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.2 | 2662.5 | 2662.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **2719.3** · total **2719.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Thành phần thật và stub

Chạy `./lab.ps1 pipeline` ngày 07/10/2026, gọi `http://localhost:8080`, top-k = 3, không truyền `--embed-url`. Giữ hai STUB của script, không tích hợp stack N16–N19 bên ngoài.

| Day | Thành phần trong lần chạy này | Trạng thái |
|:--|:--|:--|
| N16 Cloud/IaC | Chạy trực tiếp trên laptop Windows; không triển khai cluster/IaC | Chưa tích hợp; local thay thế |
| N17 Data pipeline | Sáu đoạn văn hard-code trong `TOY_DOCS`, không ingest/ETL | Stub |
| N18 Lakehouse | Danh sách Python trong RAM, không dùng lakehouse/SQLite | Stub |
| N19 Vector + features | `retrieve()` chấm keyword overlap, lấy top-3; không embedding/vector index | Stub |
| N20 Serving | HTTP `/v1/chat/completions` gọi llama-server b10488, Gemma 4 E2B Q4 | Real |

Ba request đều có câu trả lời không rỗng và server timings có token sinh. Context đứng đầu lần lượt là `goodput`, `paged`, `disagg`, phù hợp với từng câu hỏi. Hai query đầu vẫn nhận thêm context score 0 vì script lấy đủ top-3; đây là hạn chế của toy retrieval, không phải bằng chứng semantic search. Câu trả lời bám vào tài liệu đồ chơi; lần chạy chứng minh tích hợp endpoint, không xác thực toàn bộ phát biểu kỹ thuật của TOY_DOCS.

## Latency và hướng tối ưu

Mean: embed **0,0 ms**, retrieve **0,1 ms**, llm **2719,3 ms**, total **2719,4 ms**. Embed 0 là không gọi embedding server và có làm tròn, không phải dịch vụ embedding thực chạy tức thời. Retrieval trên sáu tài liệu rất nhỏ nên kết quả không đại diện cho vector database production.

Bước `llm` chiếm khoảng **99,996%** tổng thời gian (report làm tròn 100%), phù hợp kỳ vọng vì embed/retrieve đang là stub nhẹ. Tuy nhiên `llm` đo toàn bộ HTTP phía client: trung bình prefill + decode phía server chỉ **602,97 ms**, chênh khoảng **2116,3 ms** so với client. Không thể gọi toàn bộ 2719,3 ms là compute hoặc decode. Phần chênh có thể gồm kết nối/HTTP, chờ và xử lý ngoài hai timer server; chưa đo riêng để xác định nguyên nhân.

Nếu cần giảm total 2×, ưu tiên kiểm tra đường gọi HTTP trong stage llm: thử địa chỉ `127.0.0.1` so với `localhost`, tái sử dụng `httpx.Client`, rồi đo trước/sau với cùng query và cấu hình. Đây là thí nghiệm đề xuất, chưa thực hiện và chưa khẳng định đạt 2×. Chỉ tối ưu retrieve sẽ tiết kiệm khoảng 0,1 ms, không đủ; ngay cả loại bỏ toàn bộ prefill/decode khoảng 603 ms cũng chưa tiết kiệm được một nửa total hiện tại.
