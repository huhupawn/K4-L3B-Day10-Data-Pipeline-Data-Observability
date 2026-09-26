# Danh Sách Thành Viên & Báo Cáo Phân Công

- **Tên Nhóm:** `Solo - Nguyễn Anh Hoàng`
- **Mã Nhóm / Lớp:** `K4-L3B-DAY10`
- **Tên Repository Nộp Bài:** `K4-L3B-DAY10-NguyenAnhHoang-DataPipelineDataObservability`

---

## 1. Thành Viên Thực Hiện

| STT | Họ và tên | MSSV | Email | Vai trò & Phân công công việc | Báo cáo cá nhân |
|---:|---|---|---|---|---|
| 1 | Nguyễn Anh Hoàng | 2A202602816 | hoang.na@vinuni.edu.vn | Toàn quyền phụ trách End-to-End: Data Ingestion, Cleaning, ChromaDB, Observability (GX 1.x), Corruption & Idempotent Self-Healing | [`report/2A202602816_NguyenAnhHoang.md`](../report/2A202602816_NguyenAnhHoang.md) |

---

## 2. Báo Cáo Đóng Góp Chi Tiết Cá Nhân

### 2.1. NguyenAnhHoang-2A202602816
- **Vai trò:** End-to-End Pipeline & Data Observability Architect (Độc lập thực hiện 100% dự án).
- **Công việc chi tiết đã hoàn thành:**
  - **Module Ingestion & Cleaning:** Hoàn thiện `crossref.py` (parse Crossref payload, fallback offline snapshot) và `cleaning.py` (chuẩn hóa schema, tính `age_days`, tạo `text_for_embedding` cấu trúc 5 phần, khử trùng lặp).
  - **Module Vector Indexing & RAG:** Thiết lập ChromaDB vector store, embedding MiniLM (`sentence-transformers/all-MiniLM-L6-v2`), xây dựng QA router đa nhà cung cấp trong `src/retrieval/`.
  - **Module Observability:** Thiết lập ephemeral context chuẩn Great Expectations 1.x với 4 Expectations thiết yếu + giám sát Freshness SLA (`age_days > 180`) trong `src/observability/quality.py`.
  - **Module Corruption & Repair:** Xây dựng 6 kịch bản làm bẩn dữ liệu trong `src/ingestion/corruption.py`, đo lường mức độ suy giảm Silent Failure và hiện thực hóa cơ chế Idempotent Repair khôi phục 100% hiệu năng từ raw snapshot trong `src/pipelines/corruption_flow.py`.
  - **Hạng mục Bonus vượt chuẩn:**
    - **Bonus B1:** Xây dựng Interactive Dashboard HTML/JS Chart.js hiển thị trạng thái Data Quality & Drift Monitor (`script/run_dashboard.py`).
    - **Bonus B2:** Xây dựng Automated Self-Healing Pipeline tự động bắt lỗi Quality Gate và kích hoạt sửa chữa (`script/run_self_healing.py`).
    - **Bonus B3:** Xây dựng bộ kiểm thử tự động 12 unit tests đạt chuẩn Pytest CI pass 100% (`tests/`).
- **Điều học được / Đóng góp chính:**
  - Nắm vững kiến trúc Data Lineage, tầm quan trọng sống còn của Data Observability để ngăn chặn Silent Failure trong RAG production, và kỹ thuật thiết kế Idempotent Pipeline đảm bảo tính toàn vẹn dữ liệu.
