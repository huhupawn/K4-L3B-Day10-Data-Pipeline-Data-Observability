# Báo Cáo Pha 1 — Data Pipeline & Observability Baseline

> **Hệ thống:** K4-L3B-Day10 Data Pipeline & Data Observability for RAG  
> **Trạng thái:** Hoàn thành Baseline End-to-End (Pha 1)

---

## 1. Tổng Quan Thu Thập & Chuẩn Hóa Dữ Liệu (Ingestion & Lineage)

- **Nguồn dữ liệu:** Crossref REST API
- **Tổng số bản ghi thô (Raw Records):** 24
- **Tổng số bản ghi làm sạch (Clean Records):** 24
- **Tập tin lưu trữ thô (Artifacts):**
  - `data/raw/crossref_response.json` (Full payload API)
  - `data/raw/crossref_records.json` (Normalized snapshot)
- **Tập tin dữ liệu sạch:**
  - `data/clean/papers_clean.csv`
  - `data/clean/papers_clean.json`

---

## 2. Kiểm Định Chất Lượng Dữ Liệu (Great Expectations 1.x & Freshness SLA)

| Hạng mục kiểm định | Chỉ số | Tiêu chuẩn đạt | Kết quả thực tế | Trạng thái |
|---|---|---|---|---|
| **Chất lượng cấu trúc (GX 1.x)** | 7/7 Expectations đạt | 100% | 100.0% | **PASS** |
| **Row Count Range** | 20 - 30 bản ghi | [20, 30] | 24 | **PASS** |
| **Tính duy nhất (Unique `paper_id`)** | Không trùng lặp | 0 trùng | 0 trùng | **PASS** |
| **Tính hợp lệ (`title`, `summary`)** | Không null & độ dài chuẩn | Title >= 8, Summary >= 20 | Đạt chuẩn | **PASS** |
| **Freshness SLA (`age_days <= 180`)** | Tỷ lệ quá hạn <= 25% | <= 25.0% | 4.2% (1/24) | **PASS** |

- **Ngày xuất bản mới nhất:** `2026-07-22`
- **Ngày xuất bản cũ nhất:** `2026-03-28`

---

## 3. Hiệu Năng RAG Retrieval & Benchmark Metrics (Baseline)

- **Vector Database:** ChromaDB (Local persistent)
- **Embedding Model:** `sentence-transformers/all-MiniLM-L6-v2` (384 dimensions)
- **Collection Name:** `papers-baseline`
- **Tập test benchmark:** 10 câu hỏi đa dạng qua 4 dạng nghiệp vụ (`summary`, `authors`, `date`, `categories`).

| Chỉ số đánh giá | Kết quả Baseline | Ghi chú |
|---|---|---|
| **Retrieval Hit Rate** | **100.0%** | Tỷ lệ tài liệu chuẩn xuất hiện trong top-k retrieval |
| **Mean Token F1** | **1.0000** | Độ chính xác token-level giữa câu trả lời và ground truth |
| **LLM Judge Accuracy** | **100.0%** | Đánh giá tính đúng đắn ngữ nghĩa qua LLM / Heuristic Judge |
| **Mean Judge Score** | **5.00 / 5.0** | Điểm số chất lượng câu trả lời |

---

## 4. Kết Luận Pha 1

Pipeline dữ liệu đã hoàn thiện từ Ingestion thô, Data Cleaning chuẩn hóa 5 thành phần `text_for_embedding`, vượt qua toàn bộ Data Quality Gate theo chuẩn Great Expectations 1.x và Freshness SLA, thiết lập chuẩn đối chiếu tin cậy cho Pha 2 (Corruption & Repair).
