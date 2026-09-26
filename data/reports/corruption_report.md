# Báo Cáo Đối Chiếu 3 Trạng Thái — Baseline vs Corrupted vs Repaired

> **Hệ thống:** K4-L3B-Day10 Data Pipeline, Observability & Idempotent Repair  
> **Mục tiêu:** Chứng minh năng lực phát hiện dữ liệu bẩn của Data Quality Gate (GX 1.x), đo lường mức độ suy thoái (Silent Failure) và hiệu quả phục hồi toàn vẹn (Self-healing).

---

## 1. Bảng Tổng Hợp Đối Chiếu Định Lượng 3 Trạng Thái

| Tiêu chí / Chỉ số | Trạng thái 1: Baseline (Dữ liệu sạch) | Trạng thái 2: Corrupted (Dữ liệu lỗi) | Trạng thái 3: Repaired (Dữ liệu phục hồi) | Nhận xét xu hướng |
|---|:---:|:---:|:---:|---|
| **Retrieval Hit Rate** | **100.0%** | **60.0%** | **100.0%** | Sụt giảm nghiêm trọng khi dữ liệu bẩn, phục hồi 100% về mức chuẩn |
| **Mean Token F1** | **1.0000** | **0.6506** | **1.0000** | Câu trả lời bị sai lệch/trống do nhiễu và title cắt ngắn, sau đó khôi phục |
| **LLM Judge Accuracy** | **100.0%** | **70.0%** | **100.0%** | Tỷ lệ câu trả lời đạt chuẩn ngữ nghĩa suy sụp rồi lấy lại phong độ |
| **Mean Judge Score** | **5.00 / 5.0** | **3.40 / 5.0** | **5.00 / 5.0** | Điểm số đánh giá chất lượng tổng thể phục hồi hoàn toàn |
| **Data Quality Gate (GX 1.x)** | **PASS** | **FAIL** | **PASS** | Bắt trọn vẹn lỗi schema, duplicate và length vi phạm |
| **Freshness SLA (Stale <= 25%)** | **PASS** | **FAIL** (52.2% stale) | **PASS** (4.2% stale) | Cảnh báo vi phạm SLA ngày xuất bản khi bị tiêm stale dates |
| **Số lượng bản ghi** | 24 | 23 | 24 | Bị mất mát và trùng lặp ở Corrupted, trở lại 24 bản ghi sạch |

---

## 2. Chi Tiết 6 Kịch Bản Làm Bẩn Dữ Liệu (Synthetic Corruption Suite)

1. **Drop latest records (Mất mát dữ liệu mới):**
   - Giả lập việc pipeline bị đứt gãy hoặc API nguồn trả thiếu dữ liệu. Khoảng 20% các tài liệu mới nhất bị loại bỏ, dẫn đến việc RAG Agent không tìm thấy thông tin của các bài báo gần đây (Hit Rate giảm).
2. **Blank summary (Xóa rỗng tóm tắt):**
   - Trường `summary` bị xóa rỗng ở một số bản ghi, vi phạm tiêu chuẩn độ dài `ExpectColumnValueLengthsToBeBetween(min_value=20)`.
3. **Inject noise (Chèn ký tự nhiễu rác):**
   - Chèn chuỗi ký tự ngẫu nhiên và nhiễu ngữ nghĩa vào văn bản tóm tắt, làm sai lệch vector embedding so với semantic query.
4. **Truncate title (Cắt ngắn tiêu đề bất thường):**
   - Tiêu đề bị cắt ngắn dưới 8 ký tự, kích hoạt cảnh báo vi phạm độ dài tiêu đề từ Great Expectations.
5. **Stale date (Lùi ngày xuất bản về quá khứ):**
   - Đẩy ngày xuất bản lùi xa về trước, khiến trường `age_days` vượt ngưỡng SLA (180 ngày). Tỷ lệ stale vượt quá 25%, kích hoạt cảnh báo `is_fresh = False`.
6. **Duplicate rows (Nhân bản bản ghi):**
   - Chèn các bản ghi trùng lặp `paper_id`, kích hoạt kiểm định `ExpectColumnValuesToBeUnique`.

---

## 3. Phân Tích Hiện Tượng "Silent Failure" trong Hệ Thống RAG

- **Nguyên nhân:** Khi dữ liệu bị nhiễm bẩn nhưng không có Data Quality Gate, pipeline vẫn chạy mà **không ném ra bất kỳ Exception/Lỗi runtime nào**. Vector DB vẫn nạp dữ liệu rác, Agent vẫn trả lời câu hỏi nhưng nội dung câu trả lời bị sai lệch hoàn toàn (Hallucination hoặc "I don't know").
- **Tác hại thực tế:** Người dùng nhận được thông tin sai lệch mà hệ thống monitoring truyền thống (chỉ giám sát CPU, RAM, HTTP 200) không hề hay biết.
- **Vai trò của Observability:** Với Great Expectations 1.x và Freshness SLA, hệ thống đã lập tức phát hiện trạng thái **FAIL** tại chốt kiểm dịch trước khi nạp vào vector store.

---

## 4. Cơ Chế Tự Phục Hồi An Toàn (Idempotent Repair)

- **Nguyên lý Idempotency:** Quá trình phục hồi có thể thực thi nhiều lần mà kết quả cuối cùng không thay đổi và không làm sinh thêm dữ liệu thừa/trùng lặp.
- **Quy trình phục hồi:**
  1. Đọc lại nguồn snapshot thô bất biến (Immutable Raw Storage) tại `data/raw/crossref_records.json`.
  2. Thực thi lại luồng làm sạch `build_clean_dataframe()`, khử trùng lặp và tính toán lại `age_days`.
  3. Ghi đè có kiểm soát vào các artifacts sạch `data/clean/papers_clean_repaired.csv` và `papers_clean_repaired.json`.
  4. Tái tạo collection ChromaDB `papers-repaired`, đảm bảo xóa sạch dữ liệu lỗi trước khi nạp vector embedding mới.
- **Kết quả nghiệm thu:** Toàn bộ các chỉ số Retrieval Hit Rate và Token F1 sau phục hồi đã quay trở lại tương đương 100% mức Baseline chuẩn.
