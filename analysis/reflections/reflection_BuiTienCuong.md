# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Bùi Tiến Cường  
**MSSV:** 2A202602539  
**Khóa:** K4 - Track 3A  
**Ngày:** 04/10/2026

## Phần 1: Mapping bài giảng

| Concept | Module / hàm | Quan sát từ triển khai |
|---|---|---|
| Semantic chunking | M1 `chunk_semantic()` | Chia câu, so cosine similarity giữa hai câu kề nhau, ngắt khi thấp hơn threshold. Test M1 đạt 13/13; chưa benchmark số chunk trên toàn corpus. |
| Hierarchical chunking | M1 `chunk_hierarchical()` | Tạo parent tối đa 2048 ký tự, child tối đa 256 ký tự; child giữ `parent_id` để truy vết context gốc. |
| BM25 + Dense fusion | M2 `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()` | BM25 xử lý khớp từ khóa tiếng Việt, dense search tìm gần nghĩa, RRF hợp nhất theo thứ hạng để không cần chuẩn hóa hai thang điểm. Test M2 đạt 5/5. |
| Cross encoder reranking | M3 `CrossEncoderReranker.rerank()` | Chấm lại từng cặp query–document và sắp xếp theo điểm; top 3 lấy từ candidate do hybrid search trả về. Test M3 đạt 5/5 trên CPU, toàn file test mất khoảng 306 giây do model lớn. |
| RAGAS 4 metrics | M4 `evaluate_ragas()`, `failure_analysis()` | Production đạt faithfulness 0.8042, answer relevancy 0.7537, context precision 0.8958, context recall 0.7917 trên 20 câu hỏi. Hai metric đầu tăng, hai metric retrieval giảm so với baseline. Test M4 đạt 4/4. |
| Contextual enrichment | M5 `_enrich_single_call()`, `contextual_prepend()` | Chạy 100 chunks trong 978 giây. Nhiều phản hồi không parse được JSON nên fallback local; đã bổ sung JSON mode và giới hạn fallback ở một API call/chunk cho lần chạy sau. Test M5 đạt 11/11. |

## Phần 2: Khó khăn và cách giải quyết

- **Lỗi gặp phải:** `WinError 10013` khi Hugging Face được gọi từ sandbox. Model cần tải trước hoặc chạy trong môi trường có quyền mạng. Không dùng lỗi này làm lý do để tạo kết quả RAGAS giả.
- **Thời gian chạy:** bộ test M3 trên CPU mất khoảng 306 giây; `check_lab.py` ban đầu timeout sau 120 giây. Đã nâng timeout của script lên 600 giây; suite riêng đạt 37/37 test trong khoảng 370 giây.
- **Giới hạn dữ liệu:** hai PDF scan (`BCTC.pdf` và nghị định bảo vệ dữ liệu cá nhân) không có text layer nên loader bỏ qua. Muốn truy xuất nội dung của chúng cần bổ sung OCR và kiểm tra chất lượng trích xuất.
- **Enrichment API:** thông báo `Expecting value: line 1 column 1 (char 0)` xuất hiện ở một số chunks. Nguyên nhân có thể là phản hồi không phải JSON; code sau chạy đã chuyển sang `response_format=json_object` và fallback local. Chưa chạy lại toàn pipeline với thay đổi này nên điểm ở trên vẫn phản ánh bản trước khi sửa.
- **Kiến thức cần bổ sung:** kiểm tra tương thích phiên bản RAGAS/LangChain, đo latency và chi phí API theo số chunks, đánh giá retrieval riêng trước khi quy lỗi cho phần sinh câu trả lời.

## Phần 3: Kế hoạch áp dụng

### Project: Production RAG cho tài liệu chính sách nội bộ (đề xuất dựa trên lab)

#### Hiện trạng

- Pipeline lab: hierarchical chunking → enrichment → BM25 + dense/Qdrant → RRF → cross encoder → LLM → RAGAS.
- Đã có RAGAS score thật: faithfulness +0.1200, answer relevancy +0.0816, context precision -0.0375, context recall -0.1333 so với baseline. Hai PDF scan vẫn chưa xử lý.

#### Kế hoạch cải tiến

1. **Chunking:** dùng hierarchical để giữ context cha; so sánh thêm structure aware cho bảng/chính sách theo mục.
2. **Retrieval:** giữ hybrid BM25 + dense/RRF và đo recall@k trên 20 câu hỏi của `test_set.json`.
3. **Reranking:** dùng `BAAI/bge-reranker-v2-m3` khi chất lượng top 3 cải thiện đủ để bù latency; đo p50/p95 thực tế.
4. **Evaluation:** dùng bốn metric RAGAS hiện có; lưu thêm answer và contexts theo từng câu để chẩn đoán chính xác, theo dõi version/negation/numeric.
5. **Enrichment:** thử contextual prepend và HyQA; ablation từng kỹ thuật để xác định cải thiện có thật.

#### Timeline

- **Tuần 1:** thêm OCR, kiểm tra dữ liệu và cải thiện context recall trên các câu hỏi nhiều nguồn.
- **Tuần 2:** phân tích bottom-5, chạy ablation, tối ưu latency và cập nhật ngưỡng chất lượng.
