# Member Role Report — Day 10: Data Pipeline & Data Observability

> **Báo cáo cá nhân thực hiện độc lập (Solo Completion)**  
> **Họ và tên:** Nguyễn Anh Hoàng  
> **MSSV:** 2A202602816  
> **Lớp:** K4 - L3B

---

## 1. Thông tin cá nhân

| Thông tin       | Nội dung                                                    |
| --------------- | ----------------------------------------------------------- |
| Họ và tên       | Nguyễn Anh Hoàng                                            |
| MSSV            | 2A202602816                                                 |
| Khóa/Lớp        | K4 - L3B                                                    |
| Tên nhóm        | Solo - Nguyễn Anh Hoàng                                     |
| Vai trò chính   | End-to-End Pipeline & Data Observability Architect          |
| Repository      | `K4-L3B-DAY10-NguyenAnhHoang-DataPipelineDataObservability` |
| Ngày hoàn thành | 2026-09-26                                                  |

---

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

Tôi trực tiếp nghiên cứu, thiết kế, triển khai và kiểm thử 100% các module trong toàn bộ dự án:

| Module/deliverable           | File/hàm phụ trách                        | Input nhận vào                | Output bàn giao                        | Trạng thái |
| ---------------------------- | ----------------------------------------- | ----------------------------- | -------------------------------------- | :--------: |
| **Ingestion**                | `src/ingestion/crossref.py`               | Crossref API / raw JSON       | `data/raw/crossref_records.json`       | Hoàn thành |
| **Cleaning & Pre-embed**     | `src/ingestion/cleaning.py`               | Raw `PaperRecord`             | `data/clean/papers_clean.json`, `.csv` | Hoàn thành |
| **Vector Store & Retrieval** | `src/retrieval/index.py`, `embeddings.py` | Cleaned DataFrame             | ChromaDB collections                   | Hoàn thành |
| **Observability (GX 1.x)**   | `src/observability/quality.py`            | Cleaned / Corrupted DataFrame | `data/quality/*_quality_report.json`   | Hoàn thành |
| **Evaluation Benchmark**     | `src/evaluation/testset.py`, `metrics.py` | DataFrame & Chroma Index      | `test_set.json`, `*_metrics.json`      | Hoàn thành |
| **Corruption Suite**         | `src/ingestion/corruption.py`             | Clean DataFrame               | `data/results/corruption_log.json`     | Hoàn thành |
| **Idempotent Repair Flow**   | `src/pipelines/corruption_flow.py`        | Immutable Raw Snapshot        | `data/reports/corruption_report.md`    | Hoàn thành |
| **Bonus B1 (Dashboard)**     | `src/observability/dashboard.py`          | Metrics & quality logs        | `observability_dashboard.html`         | Hoàn thành |
| **Bonus B2 (Self-Healing)**  | `src/pipelines/self_healing.py`           | Corrupted batch stream        | Quarantine & Auto-healed Index         | Hoàn thành |
| **Bonus B3 (Pytest CI)**     | `tests/` (12 unit tests)                  | Toàn bộ các module            | 12/12 test PASSED                      | Hoàn thành |

---

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện        | File/hàm/artifact liên quan             | Kết quả bàn giao                                | Cách xác minh                                             |
| ---------------------------- | --------------------------------------- | ----------------------------------------------- | --------------------------------------------------------- |
| Ingestion & Snapshot Lineage | `src/ingestion/crossref.py`             | 24 bản ghi raw metadata                         | `fetch_source_records()` tải đủ 24 bài báo                |
| Data Cleaning & Modeling     | `src/ingestion/cleaning.py`             | 24 bản ghi sạch có `text_for_embedding` 5 phần  | `build_clean_dataframe()` loại bỏ thẻ XML, khử trùng      |
| Observability Gate (GX 1.x)  | `src/observability/quality.py`          | GX 1.x validation & Freshness SLA report        | GX `success=True`, Freshness `is_fresh=True` (4.2% stale) |
| Benchmark Testset & Indexing | `src/evaluation/testset.py`, `index.py` | 10 câu hỏi benchmark & ChromaDB collection      | Sinh đủ 10 câu qua 4 nhóm nghiệp vụ                       |
| Baseline End-to-End          | `script/run_phase1.py`                  | `baseline_metrics.json`, `phase1_report.md`     | Hit Rate = 100.0%, Token F1 = 1.0000                      |
| Corruption & Degradation     | `src/ingestion/corruption.py`           | `corrupted_metrics.json`, `corruption_log.json` | Hit Rate sụt giảm nghiêm trọng còn 60.0%                  |
| Idempotent Repair & Compare  | `script/run_corruption_flow.py`         | `corruption_report.md`, `repaired_metrics.json` | Phục hồi 100% hiệu năng Baseline                          |

---

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết

Trong các hệ thống RAG thực tế, lỗi dữ liệu (data corruption) thường diễn ra trong âm thầm (**Silent Failure**): pipeline không báo lỗi runtime, vector database vẫn nạp văn bản rác, và LLM vẫn trả lời người dùng nhưng nội dung hoàn toàn sai lệch hoặc bịa đặt (hallucination). Cần phải xây dựng một chốt kiểm dịch chất lượng tự động (**Data Quality Gate**) bằng Great Expectations 1.x và Freshness SLA, đồng thời hiện thực hóa cơ chế tự phục hồi an toàn (**Idempotent Repair**).

### Cách triển khai

1. **Chuẩn hóa Embedding 5 phần:**
   Ghép cấu trúc cố định gồm: `Title`, `Authors`, `Categories`, `Published`, `Summary` để đảm bảo dense vector bao quát toàn bộ ngữ cảnh tìm kiếm.
2. **Kiểm định chất lượng với Great Expectations 1.x:**
   Sử dụng API mới của GX 1.x thông qua Ephemeral Context:
   ```python
   context = gx.get_context(mode="ephemeral")
   data_source = context.data_sources.add_pandas(name="papers_source")
   data_asset = data_source.add_dataframe_asset(name="papers_asset")
   batch_def = data_asset.add_batch_definition_whole_dataframe("papers_batch")
   batch = batch_def.get_batch(batch_parameters={"dataframe": df})
   ```
   Thiết lập 4 Expectations: `ExpectTableRowCountToBeBetween`, `ExpectColumnValuesToNotBeNull`, `ExpectColumnValuesToBeUnique`, `ExpectColumnValueLengthsToBeBetween`.
3. **Giám sát Freshness SLA:**
   Tính `age_days = (run_date - published).days`. Nếu tỷ lệ tài liệu có `age_days > 180` vượt quá 25%, hệ thống lập tức bật cảnh báo `is_fresh = False`.
4. **Idempotent Repair:**
   Phục hồi trực tiếp từ nguồn snapshot thô bất biến `data/raw/crossref_records.json`, tái nạp collection ChromaDB riêng `papers-repaired`, đảm bảo không tạo bản ghi trùng lặp dù chạy lại nhiều lần.

---

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Lựa chọn phiên bản và kiến trúc khởi tạo Great Expectations (GX).
- **Các phương án cân nhắc:**
  - _Phương án A:_ Dùng cú pháp cũ GX 0.18.x (`from great_expectations.dataset import PandasDataset`).
  - _Phương án B:_ Dùng cú pháp chuẩn **Great Expectations 1.x** với Ephemeral Context và Data Source / Data Asset Batch Definition.
- **Phương án đã chọn:** Phương án B (GX 1.x).
- **Lý do:** GX 0.18.x đã bị deprecate và gây crash trên môi trường thư viện mới `great-expectations>=1.16.1`. GX 1.x Ephemeral Context cho phép kiểm định in-memory cực nhanh, không cần cấu hình file `great_expectations.yml` cồng kềnh, tương thích hoàn toàn với luồng CI/CD.

---

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng:** Khi chạy môi trường mặc định trên Windows, gặp lỗi `Package 'day10-data-observability-lab-student' requires a different Python: 3.14.7 not in '<3.14,>=3.11'`, và khi in chuỗi tiếng Việt bị văng lỗi `UnicodeEncodeError: 'charmap' codec can't encode characters in position 6-7`.
- **Nguyên nhân:** Python mặc định trên máy là 3.14.7 (chưa có wheel tương thích cho `torch` và `chromadb`), console Windows PowerShell mặc định dùng bảng mã cp1252.
- **Cách xử lý:**
  1. Dùng Python launcher chọn phiên bản Python 3.12: `py -3.12 -m venv .venv`.
  2. Bật biến môi trường `$env:PYTHONIOENCODING='utf-8'` trước khi thực thi script.
- **Kết quả:** Pipeline và toàn bộ 12 test Pytest chạy trơn tru, không gặp lỗi encoding hay dependency.

---

## 7. Hiểu biết về luồng end-to-end

1. **Dữ liệu đi từ Crossref đến vector index:** API trả payload JSON &rarr; parse trích xuất DOI, title, abstract &rarr; làm sạch thẻ JATS XML &rarr; tính `age_days` &rarr; ghép chuỗi `text_for_embedding` 5 phần &rarr; sinh vector dense 384 chiều qua `all-MiniLM-L6-v2` &rarr; nạp vào collection ChromaDB với metadata đầy đủ.
2. **Đo retrieval/answer quality:** Bộ benchmark testset 10 câu hỏi cố định truy vấn vào Vector Store; đo `retrieval_hit_rate` xem ID tài liệu chuẩn có nằm trong top-k hay không; trích xuất câu trả lời và so sánh với Ground Truth để tính `mean_token_f1`.
3. **Quality checks vs Freshness monitoring:** Quality checks kiểm soát tính toàn vẹn cấu trúc và schema (không null, duy nhất, độ dài tối thiểu); Freshness SLA kiểm soát tính cập nhật theo thời gian (độ tuổi tài liệu không được quá cũ).
4. **Tại sao giữ nguyên test set:** Để đảm bảo tính công bằng khoa học và tính đối chứng chuẩn (cùng một thước đo cho 3 trạng thái Baseline, Corrupted và Repaired).
5. **Tiêu chuẩn Repair thành công:** Dữ liệu sau repair phải vượt qua 100% Quality Gate của GX 1.x và Freshness SLA, đồng thời các chỉ số Retrieval Hit Rate và Token F1 phải phục hồi trọn vẹn về mức Baseline ban đầu.

---

## 8. Bảng đối chiếu định lượng 3 trạng thái

| Metric / Signal                |    Baseline     |    Corrupted     |    Repaired     | Nhận xét                                                 |
| ------------------------------ | :-------------: | :--------------: | :-------------: | -------------------------------------------------------- |
| `retrieval_hit_rate`           |   **100.0%**    |    **60.0%**     |   **100.0%**    | Sụt giảm nghiêm trọng khi dữ liệu bẩn, phục hồi trọn vẹn |
| `mean_token_f1`                |   **1.0000**    |    **0.6506**    |   **1.0000**    | Phục hồi hoàn toàn                                       |
| `judge_accuracy`               |   **100.0%**    |    **70.0%**     |   **100.0%**    | Độ chính xác ngữ nghĩa khôi phục 100%                    |
| `mean_judge_score`             |    **5.00**     |     **3.40**     |    **5.00**     | Chất lượng câu trả lời lấy lại phong độ                  |
| Great Expectations 1.x         |    **PASS**     |     **FAIL**     |    **PASS**     | Phát hiện toàn bộ lỗi schema, rỗng và trùng lặp          |
| Freshness SLA (Stale &le; 25%) | **PASS (4.2%)** | **FAIL (52.2%)** | **PASS (4.2%)** | Báo động vi phạm SLA khi dữ liệu bị cũ hóa               |

---

## 9. Điều học được và hướng phát triển

1. **Về Data Pipeline:** Hiểu rõ giá trị của tính Idempotency và Immutable Raw Storage trong thiết kế hệ thống phục hồi dữ liệu tự động.
2. **Về Data Observability:** Nhận thức rõ ràng rằng giám sát phần cứng (CPU/RAM) là không đủ; cần có Data Quality Gate tại từng tầng chuyển đổi dữ liệu để ngăn ngừa Silent Failure.
3. **Về RAG:** Chất lượng câu trả lời của LLM Agent phụ thuộc sống còn vào độ sạch của dữ liệu đầu vào. "Garbage in, garbage out" là rủi ro lớn nhất của hệ thống AI sản xuất.

---

## 10. Cam kết của học viên

- [x] Nội dung báo cáo phản ánh đúng phần việc tôi đã tự làm và nắm vững 100%.
- [x] Tôi có thể tự tin thuyết trình và bảo vệ toàn bộ kiến trúc pipeline trước Giảng viên và Hội đồng.
- [x] Mọi kết luận về kết quả đều có artifact và metric thực tế trong repository để đối chứng.
- [x] Tuyệt đối không commit API key hoặc secret lên Git.

**Người báo cáo:** Nguyễn Anh Hoàng  
**Ngày xác nhận:** 2026-09-26
