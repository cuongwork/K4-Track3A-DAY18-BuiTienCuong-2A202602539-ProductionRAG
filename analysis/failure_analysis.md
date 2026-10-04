# Failure Analysis — Lab 18: Production RAG

**Họ và tên:** Bùi Tiến Cường
**MSSV:** 2A202602539
**Khóa:** K4 - Track 3A

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|---|---:|---:|---:|
| Faithfulness | 0.6842 | 0.8042 | +0.1200 |
| Answer Relevancy | 0.6721 | 0.7537 | +0.0816 |
| Context Precision | 0.9333 | 0.8958 | -0.0375 |
| Context Recall | 0.9250 | 0.7917 | -0.1333 |

Điểm lấy từ `reports/naive_baseline_report.json` và `reports/ragas_report.json` cho 20 câu hỏi. Production cải thiện chất lượng câu trả lời nhưng retrieval recall giảm. `save_report()` hiện không lưu nguyên văn đáp án và contexts từng câu. Vì thế mục “Got” bên dưới ghi rõ là **không được lưu**; chẩn đoán nguyên nhân là giả thuyết dựa trên metric thấp nhất và tài liệu nguồn.

## Bottom-5 Failures

### 1. Senior 9 năm thâm niên

- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** 18 ngày phép (15 + 3); lương Senior 20–35 triệu VNĐ/tháng.
- **Got:** Không được lưu trong báo cáo lần chạy.
- **Worst metric:** Faithfulness; điểm trung bình bốn metric `0.2083`.
- **Error Tree:** Output sai hoặc thiếu → kiểm tra hai nguồn `nghi_phep_nam_v2024.md` và `bang_luong_2024.md` có cùng xuất hiện trong top-3 contexts không → nếu có, kiểm tra phép tính và prompt.
- **Root cause (giả thuyết):** Câu hỏi multi-hop cần kết hợp phép năm, thâm niên và bảng lương; faithfulness thấp cho thấy đáp án có thể vượt quá bằng chứng trong contexts.
- **Suggested fix:** Retrieve cả hai nguồn, yêu cầu đáp án kèm phép tính `15 + 9/3` và trích dẫn từng nguồn.

### 2. Mua thiết bị 55 triệu

- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Tổng Giám đốc (CEO), theo `mua_sam.md` vì đơn hàng trên 50 triệu.
- **Got:** Không được lưu trong báo cáo lần chạy.
- **Worst metric:** Context recall; điểm `0.5777`.
- **Error Tree:** Kiểm tra chunk chứa bảng thẩm quyền mua sắm có trong top-20 và top-3 không.
- **Root cause (giả thuyết):** Hierarchical child 256 ký tự có thể tách bảng giữa ngưỡng tiền và người phê duyệt.
- **Suggested fix:** Structure-aware chunking giữ nguyên bảng; thêm kiểm tra retrieval@3 cho `mua_sam.md`.

### 3. Nghỉ không lương 20 ngày

- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Giám đốc điều hành (CEO); nếu nghỉ trên 14 ngày, nhân viên tự đóng phần bảo hiểm của mình.
- **Got:** Không được lưu trong báo cáo lần chạy.
- **Worst metric:** Context precision; điểm `0.6966`.
- **Error Tree:** Kiểm tra top-3 có đoạn không liên quan và score sau rerank.
- **Root cause (giả thuyết):** Các văn bản nghỉ phép khác cạnh tranh trong hybrid search; đoạn phê duyệt từ `nghi_phep_khong_luong.md` có thể không nằm đủ cao.
- **Suggested fix:** Lọc metadata theo loại nghỉ không lương và giữ nguyên mục “Quy trình phê duyệt”.

### 4. Hạn mức PVI

- **Question:** Bảo hiểm sức khỏe PVI có hạn mức bao nhiêu cho nhân viên?
- **Expected:** 200.000.000 VNĐ/năm, gồm nội trú, ngoại trú và nha khoa.
- **Got:** Không được lưu trong báo cáo lần chạy.
- **Worst metric:** Faithfulness; điểm `0.7159`.
- **Error Tree:** Kiểm tra có nhầm hạn mức nhân viên 200 triệu với gói gia đình 50 triệu trong `bao_hiem_suc_khoe.md` không.
- **Root cause (giả thuyết):** Hai hạn mức ở gần nhau trong cùng tài liệu; câu trả lời có thể kết hợp sai đối tượng.
- **Suggested fix:** Prompt buộc nêu rõ “nhân viên chính thức” và trích đúng đoạn “Bảo hiểm cho nhân viên”.

### 5. Quyền lợi PVI khi thử việc

- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** Không. Chỉ tham gia bảo hiểm xã hội bắt buộc, chưa hưởng PVI.
- **Got:** Không được lưu trong báo cáo lần chạy.
- **Worst metric:** Answer relevancy; điểm `0.7500`.
- **Error Tree:** Kiểm tra `thu_viec.md` có vào contexts và câu trả lời có mở đầu bằng “Không” không.
- **Root cause (giả thuyết):** Chính sách PVI cho nhân viên chính thức dễ lấn át ngoại lệ dành cho thử việc.
- **Suggested fix:** Ưu tiên chunk chứa điều kiện “thử việc” và prompt trả lời trực tiếp có/không trước phần giải thích.

## Case Study

**Question:** Senior 9 năm thâm niên có bao nhiêu ngày phép và mức lương nào?

1. **Output đúng?** Chưa kiểm chứng được nguyên văn vì báo cáo không lưu đáp án.
2. **Context đúng?** Cần cả `nghi_phep_nam_v2024.md` và `bang_luong_2024.md`.
3. **Query rewrite OK?** Pipeline hiện chưa có query decomposition cho hai ý độc lập.
4. **Fix ưu tiên:** Tách query thành hai nhánh retrieval rồi hợp nhất contexts trước khi sinh đáp án.

**Nếu có thêm một giờ:** lưu per-question answer/context/metric vào report ở lần chạy tiếp theo, kiểm tra recall@3 cho năm câu trên, rồi chạy ablation giữa child chunking và structure-aware chunking cho bảng.
