# Group Report — Day 10: Data Pipeline & Data Observability

> **Hệ thống:** K4-L3B-Day10 Data Pipeline, Observability & Idempotent Self-Healing  
> **Lớp:** K4 - Lớp B (Ca Sáng)  
> **Chủ đề:** Data Pipeline, Data Observability with Great Expectations 1.x & RAG Degradation Measurement

---

## 1. Thông tin bài nộp

| Thông tin          | Nội dung                                                    |
| ------------------ | ----------------------------------------------------------- |
| Khóa/Lớp           | K4 - L3B                                                    |
| Tên nhóm / Cá nhân | Nguyễn Anh Hoàng                                            |
| MSSV               | 2A202602816                                                 |
| Repository         | `K4-L3B-DAY10-NguyenAnhHoang-DataPipelineDataObservability` |
| Ngày hoàn thành    | 2026-09-26                                                  |

### Thành viên và phân công

| STT | Họ và tên        | MSSV        | Vai trò chính                                       | Module/deliverable sở hữu                                                                                    |
| --: | ---------------- | ----------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
|   1 | Nguyễn Anh Hoàng | 2A202602816 | Toàn quyền thực hiện độc lập (End-to-End Architect) | Toàn bộ dự án (`core/`, `ingestion/`, `retrieval/`, `observability/`, `evaluation/`, `pipelines/`, `tests/`) |

---

## 2. Tóm tắt kết quả

Tôi đã hoàn thành 100% các yêu cầu từ Checkpoint 0 đến Checkpoint 6 cùng đầy đủ 3 hạng mục Bonus (B1: Web Dashboard, B2: Auto-Repair Pipeline, B3: Pytest CI Suite):

- **Baseline Pipeline (Pha 1):** Tải và chuẩn hóa thành công 24 bài báo học thuật từ Crossref API, làm sạch dữ liệu loại bỏ tag JATS XML, sinh cấu trúc embedding 5 phần, đánh chỉ mục vector trên ChromaDB với mô hình `all-MiniLM-L6-v2`. Hệ thống vượt qua 100% Data Quality Gate theo chuẩn Great Expectations 1.x và Freshness SLA (`is_fresh = True`, stale ratio = 4.2%). Đạt Baseline Retrieval Hit Rate = 100.0% và Token F1 = 1.0000 trên bộ 10 câu hỏi benchmark.
- **Data Corruption (Pha 2):** Triển khai đủ 6 kịch bản làm bẩn dữ liệu (drop 20% bản ghi mới, xóa rỗng summary, tiêm noise, cắt ngắn title < 8 ký tự, lùi ngày xuất bản sang 2018 gây quá hạn SLA, nhân bản bản ghi trùng lặp). Dữ liệu bẩn lập tức bị chặn bởi Data Quality Gate (GX Status = FAIL, Freshness SLA = FAIL với 52.2% stale). Hiệu năng RAG Agent bị suy thoái trầm trọng (Silent Failure: Hit Rate giảm về 60.0%, Token F1 giảm về 0.6506, Judge Accuracy giảm còn 70.0%).
- **Idempotent Repair (Pha 3):** Kích hoạt cơ chế tự phục hồi từ nguồn snapshot thô bất biến `data/raw/crossref_records.json`, tái tạo collection sạch `papers-repaired`. Toàn bộ chỉ số phục hồi 100% về mức Baseline chuẩn (Hit Rate = 100.0%, Token F1 = 1.0000, GX = PASS, Freshness = PASS).

---

## 3. Kiến trúc và luồng dữ liệu

### Luồng end-to-end

```text
Crossref Metadata API (hoặc Local Snapshot data/raw/crossref_response.json)
    │
    ▼
[Ingestion Module (crossref.py)] ──► data/raw/crossref_records.json
    │
    ▼
[Cleaning Module (cleaning.py)] ──► age_days, deduplication, text_for_embedding (5 parts)
    │
    ├────────────────────────────────────────┬────────────────────────────────────────┐
    ▼                                        ▼                                        ▼
[Observability (GX 1.x)]          [ChromaDB Local Persist]                 [Benchmark Test Set]
4 Expectations + Freshness SLA    all-MiniLM-L6-v2 (384-dim)               10 questions (4 types)
    │                                        │                                        │
    └────────────────────────────────────────┴────────────────────────────────────────┘
                                             ▼
                                  [Evaluation & Scoring]
                                  Baseline Hit Rate: 100% | Token F1: 1.0000
                                             │
    ┌────────────────────────────────────────┴────────────────────────────────────────┐
    ▼                                                                                 ▼
[Data Corruption Suite]                                                   [Automated Self-Healing]
6 kịch bản lỗi (Drop, Blank, Noise, Truncate, Stale, Duplicate)           Phát hiện GX FAIL -> Alarm
Hit Rate: 60% | Token F1: 0.6506 | GX: FAIL | Stale: 52%                  Quarantine -> Immutable Re-clean
                                                                          Phục hồi: 100% Baseline
```

### Trách nhiệm của từng khối

| Khối              | Input                              | Xử lý chính                                                          | Output/artifact                              | Owner            |
| ----------------- | ---------------------------------- | -------------------------------------------------------------------- | -------------------------------------------- | ---------------- |
| Ingestion         | Crossref API / local response JSON | Fetch, retry 429, parse fields, JATS cleaning                        | `data/raw/crossref_records.json`             | Nguyễn Anh Hoàng |
| Cleaning          | Raw `PaperRecord` objects          | Deduplication, compute `age_days`, build 5-part `text_for_embedding` | `data/clean/papers_clean.json`, `.csv`       | Nguyễn Anh Hoàng |
| Embedding/index   | Cleaned DataFrame                  | Local ChromaDB indexing qua `all-MiniLM-L6-v2`                       | `data/chroma/`, `data/embeddings/`           | Nguyễn Anh Hoàng |
| Evaluation        | Test set & Index                   | Top-k semantic search, exact match, Token F1, LLM Judge              | `data/results/*_metrics.json`                | Nguyễn Anh Hoàng |
| Observability     | Cleaned/Corrupted DataFrame        | Ephemeral GX 1.x context, 4 Expectations, Freshness SLA              | `data/quality/*_quality_report.json`         | Nguyễn Anh Hoàng |
| Corruption/repair | Cleaned DataFrame / Raw snapshot   | Tiêm 6 dạng lỗi; Idempotent repair từ raw snapshot                   | `corruption_log.json`, `repaired_clean.json` | Nguyễn Anh Hoàng |
| Orchestration     | CLI Entrypoints                    | Chạy chuỗi Phase 1 và Phase 2 end-to-end                             | `phase1_report.md`, `corruption_report.md`   | Nguyễn Anh Hoàng |

---

## 4. Cách tái hiện kết quả

### Cấu hình không chứa secret

| Biến/cấu hình             | Giá trị sử dụng                                       |
| ------------------------- | ----------------------------------------------------- |
| `LLM_PROVIDER`            | `mock` (hỗ trợ chuyển `gemini` / `openai` qua `.env`) |
| `LLM_MODEL`               | `mock-model` (hoặc `gemini-2.5-flash`)                |
| Embedding model           | `sentence-transformers/all-MiniLM-L6-v2`              |
| Số lượng Crossref records | 24                                                    |
| Retrieval `top_k`         | 4                                                     |
| Freshness threshold       | 180 ngày                                              |

### Lệnh cài đặt

Sử dụng môi trường ảo Python 3.12 (đảm bảo điều kiện `>=3.11,<3.14`):

```bash
# Tạo môi trường ảo
py -3.12 -m venv .venv
.\.venv\Scripts\activate

# Cài đặt toàn bộ dependencies
pip install -e .
pip install pytest
```

### Lệnh chạy

1. **Chạy Baseline Pipeline (Pha 1):**
   ```bash
   python script/run_phase1.py
   ```
2. **Chạy Corruption & Idempotent Repair (Pha 2):**
   ```bash
   python script/run_corruption_flow.py
   ```
3. **Chạy Automated Self-Healing (Bonus B2):**
   ```bash
   python script/run_self_healing.py
   ```
4. **Mở Interactive Dashboard (Bonus B1):**
   ```bash
   python script/run_dashboard.py
   ```
5. **Chạy kiểm thử tự động Pytest (Bonus B3):**
   ```bash
   pytest -v
   ```

### Kết quả tái hiện

| Lệnh                            | Trạng thái                      | Thời điểm chạy   | Bằng chứng                                                   |
| ------------------------------- | ------------------------------- | ---------------- | ------------------------------------------------------------ |
| `script/run_phase1.py`          | Thành công (Exit code 0)        | 2026-09-26 14:35 | `data/results/baseline_metrics.json`, `phase1_report.md`     |
| `script/run_corruption_flow.py` | Thành công (Exit code 0)        | 2026-09-26 14:36 | `data/reports/corruption_report.md`, `repaired_metrics.json` |
| `pytest -v`                     | 12/12 test PASSED (Exit code 0) | 2026-09-26 14:37 | Console output 12 passed in 22.88s                           |

---

## 5. Ingestion, cleaning và data contract

### Nguồn dữ liệu

- **Source API:** Crossref REST API (`https://api.crossref.org/works`)
- **Query:** `agentic retrieval augmented generation large language model`
- **Filter:** `from-pub-date:2026-03-30,has-abstract:true`
- **Offline Fallback Snapshot:** `data/raw/crossref_response.json` (tự động kích hoạt khi offline hoặc gặp HTTP 429)
- **Số bản ghi:** 24 bản ghi chuẩn.

### Raw và clean schema

| Trường               | Kiểu dữ liệu | Bắt buộc | Ý nghĩa                               | Xử lý khi thiếu/sai                                     |
| -------------------- | ------------ | :------: | ------------------------------------- | ------------------------------------------------------- |
| `paper_id`           | `str`        |    Có    | Định danh duy nhất (DOI)              | Bỏ qua bản ghi nếu thiếu                                |
| `title`              | `str`        |    Có    | Tiêu đề bài báo                       | Bỏ qua bản ghi nếu thiếu; chuẩn hóa whitespace          |
| `summary`            | `str`        |    Có    | Tóm tắt trừu tượng                    | Loại bỏ thẻ `<jats:p>`, chuẩn hóa khoảng trắng          |
| `authors`            | `list[str]`  |  Không   | Danh sách tác giả                     | Gộp `given` và `family`, mặc định `"Unknown"` nếu trống |
| `authors_joined`     | `str`        |    Có    | Chuỗi tác giả nối phẩy                | Phục vụ truy vấn metadata                               |
| `categories`         | `list[str]`  |  Không   | Danh mục chuyên ngành                 | Lấy từ `subject`, mặc định `["General"]`                |
| `published`          | `str`        |    Có    | Ngày xuất bản ISO (YYYY-MM-DD)        | Parse từ `date-parts`, fallback về `created`            |
| `age_days`           | `int`        |    Có    | Độ tuổi tài liệu tính theo ngày       | `(run_date - published).days`                           |
| `text_for_embedding` | `str`        |    Có    | Chuỗi chuẩn hóa 5 thành phần để embed | Ghép Title, Authors, Categories, Published, Summary     |

### Cấu trúc 5 phần của `text_for_embedding`

```text
Title: {title}
Authors: {authors_joined}
Categories: {categories_joined}
Published: {published}
Summary: {summary}
```

---

## 6. Evaluation setup

- **Số lượng câu hỏi benchmark:** 10 câu hỏi chuẩn hóa trong `data/eval/test_set.json`.
- **4 dạng nghiệp vụ:**
  1. `summary` (3 câu): Hỏi về nội dung tóm tắt và đóng góp cốt lõi của bài báo.
  2. `authors` (3 câu): Hỏi về danh sách tác giả của bài báo.
  3. `date` (2 câu): Hỏi về thời điểm xuất bản chính thức.
  4. `categories` (2 câu): Hỏi về phân loại chuyên ngành của nghiên cứu.
- **Nguyên tắc cố định test set:** Test set được giữ nguyên 100% qua cả 3 trạng thái (Baseline, Corrupted, Repaired) nhằm đảm bảo tính khách quan và khoa học, cho phép đo lường chính xác mức độ suy giảm do dữ liệu bẩn và mức độ phục hồi sau khi sửa chữa.

---

## 7. Kết quả baseline

### Artifact checklist

| Artifact                 | Đường dẫn thực tế                           | Trạng thái | Ghi chú                                  |
| ------------------------ | ------------------------------------------- | :--------: | ---------------------------------------- |
| Raw response/records     | `data/raw/crossref_records.json`            |     Có     | 24 bản ghi snapshot thô                  |
| Cleaned dataset          | `data/clean/papers_clean.json`              |     Có     | 24 dòng sạch đầy đủ `text_for_embedding` |
| Embedding manifest/index | `data/embeddings/papers_embeddings.json`    |     Có     | Nạp vào ChromaDB `papers-baseline`       |
| Evaluation set           | `data/eval/test_set.json`                   |     Có     | 10 câu hỏi test đa dạng                  |
| Baseline metrics         | `data/results/baseline_metrics.json`        |     Có     | Hit Rate = 100%, F1 = 1.0                |
| Quality/freshness        | `data/quality/baseline_quality_report.json` |     Có     | GX: PASS, Freshness: PASS                |
| Baseline report          | `data/reports/phase1_report.md`             |     Có     | Báo cáo Markdown chi tiết Pha 1          |

### Baseline metrics

- **Retrieval Hit Rate:** `100.0%` (Tất cả 10 câu hỏi đều truy xuất chính xác tài liệu nguồn chứa đáp án).
- **Mean Token F1:** `1.0000` (Câu trả lời trích xuất khớp hoàn toàn với Ground Truth).
- **LLM Judge Accuracy:** `100.0%`
- **Mean Judge Score:** `5.00 / 5.0`

---

## 8. Data quality và freshness

### Quality checks (Great Expectations 1.x)

| Check                                            | Quality dimension | Ngưỡng/kỳ vọng   |  Kết quả baseline   | Bằng chứng                     |
| ------------------------------------------------ | ----------------- | ---------------- | :-----------------: | ------------------------------ |
| `ExpectTableRowCountToBeBetween`                 | Completeness      | [20, 30] bản ghi |      PASS (24)      | `baseline_quality_report.json` |
| `ExpectColumnValuesToNotBeNull (paper_id)`       | Validity          | 0 nulls          |   PASS (0 nulls)    | `baseline_quality_report.json` |
| `ExpectColumnValuesToBeUnique (paper_id)`        | Uniqueness        | 100% unique      | PASS (0 duplicates) | `baseline_quality_report.json` |
| `ExpectColumnValuesToNotBeNull (title)`          | Completeness      | 0 nulls          |   PASS (0 nulls)    | `baseline_quality_report.json` |
| `ExpectColumnValueLengthsToBeBetween (title)`    | Validity          | Length >= 8      |        PASS         | `baseline_quality_report.json` |
| `ExpectColumnValueLengthsToBeBetween (summary)`  | Validity          | Length >= 20     |        PASS         | `baseline_quality_report.json` |
| `ExpectColumnValuesToNotBeNull (embedding_text)` | Completeness      | 0 nulls          |        PASS         | `baseline_quality_report.json` |

### Freshness SLA

- **Ngưỡng SLA:** `age_days <= 180` ngày. Cảnh báo vi phạm nếu tỷ lệ quá hạn > 25.0%.
- **Kết quả Baseline:** 1 bản ghi quá hạn trên tổng số 24 (tỷ lệ 4.2% <= 25.0%).
- **Trạng thái:** **PASS (`is_fresh = True`)**.

---

## 9. Corruption scenarios và repair

| Corruption              | Cách tạo                              | Record bị tác động | Quality signal kỳ vọng    | Tác động thực tế đến RAG                                   |
| ----------------------- | ------------------------------------- | :----------------: | ------------------------- | ---------------------------------------------------------- |
| **Drop latest records** | Cắt bỏ 20% bản ghi mới nhất           |     4 bản ghi      | Row count giảm            | Hit Rate giảm từ 100% xuống 60% do không tìm thấy tài liệu |
| **Blank summary**       | Xóa rỗng trường `summary`             |     2 bản ghi      | Summary length check FAIL | Token F1 giảm do mất ngữ cảnh trả lời                      |
| **Inject noise**        | Chèn chuỗi rác vào tóm tắt            |     3 bản ghi      | Phá vỡ ngữ nghĩa vector   | Vector similarity giảm                                     |
| **Truncate title**      | Cắt tiêu đề thành `"Bad"` (< 8 chars) |     2 bản ghi      | Title length check FAIL   | Lookup chính xác theo tên bài báo thất bại                 |
| **Stale date**          | Lùi ngày xuất bản về 2018             |     8 bản ghi      | Freshness SLA FAIL        | Tỷ lệ stale nhảy vọt lên 52.2% (> 25%)                     |
| **Duplicate rows**      | Nhân bản 3 bản ghi                    |     3 bản ghi      | Uniqueness check FAIL     | Trùng lặp vector gây loãng kết quả top-k                   |

### Cơ chế Idempotent Repair

Thay vì chỉnh sửa chắp vá trên dữ liệu bẩn, quy trình thực hiện nạp lại từ **Immutable Raw Snapshot** (`data/raw/crossref_records.json`), chạy lại toàn bộ quy trình làm sạch chuẩn, tạo lại collection `papers-repaired` trong ChromaDB. Nhờ tính Idempotent, quy trình có thể chạy bất kỳ lúc nào mà luôn đảm bảo trả về đúng 24 bản ghi sạch duy nhất.

---

## 10. So sánh baseline, corrupted và repaired

| Metric/signal                  |    Baseline     |    Corrupted     |    Repaired     | Thay đổi do corruption |    Mức phục hồi    |
| ------------------------------ | :-------------: | :--------------: | :-------------: | :--------------------: | :----------------: |
| `retrieval_hit_rate`           |   **100.0%**    |    **60.0%**     |   **100.0%**    |       Giảm 40.0%       | **Phục hồi 100%**  |
| `mean_token_f1`                |   **1.0000**    |    **0.6506**    |   **1.0000**    |      Giảm 0.3494       | **Phục hồi 100%**  |
| `judge_accuracy`               |   **100.0%**    |    **70.0%**     |   **100.0%**    |       Giảm 30.0%       | **Phục hồi 100%**  |
| `mean_judge_score`             |    **5.00**     |     **3.40**     |    **5.00**     |     Giảm 1.60 điểm     | **Phục hồi 100%**  |
| Great Expectations 1.x         |    **PASS**     |     **FAIL**     |    **PASS**     | Phát hiện toàn bộ lỗi  | **Đạt chuẩn 100%** |
| Freshness SLA (Stale &le; 25%) | **PASS (4.2%)** | **FAIL (52.2%)** | **PASS (4.2%)** |      Vi phạm SLA       | **Khôi phục SLA**  |

### Hai kết luận nhân quả:

1. **Dữ liệu lỗi &rarr; Silent Failure:** Khi dữ liệu bị drop 20% và truncate title, hệ thống không hề văng exception lúc embedding, nhưng Retrieval Hit Rate sụt giảm nghiêm trọng từ 100% về 60% và Token F1 giảm còn 0.6506. Điều này chứng minh sự nguy hiểm của Silent Failure trong hệ thống RAG nếu thiếu Data Observability.
2. **Idempotent Repair &rarr; Full Recovery:** Việc khôi phục từ snapshot thô đã đưa tất cả 4 tiêu chuẩn GX và Freshness SLA trở lại trạng thái PASS, kéo toàn bộ chỉ số Retrieval Hit Rate và Token F1 quay lại mức 100.0% và 1.0000.

---

## 11. Vấn đề tích hợp quan trọng

- **Triệu chứng:** Khi chạy môi trường mặc định trên máy Windows, hệ thống báo lỗi không tương thích phiên bản Python (`Python 3.14.7 not in '<3.14,>=3.11'`), đồng thời lệnh print Unicode tiếng Việt bị lỗi `cp1252 charmap encoding`.
- **Nguyên nhân:** Máy chủ phát triển cài sẵn Python 3.14 (chưa hỗ trợ bởi ChromaDB và PyTorch wheels), và console Windows mặc định dùng bảng mã cp1252 thay vì UTF-8.
- **Cách xử lý:**
  1. Sử dụng launcher `py -3.12 -m venv .venv` để khởi tạo môi trường ảo Python 3.12.10 tương thích hoàn toàn.
  2. Bật biến môi trường `$env:PYTHONIOENCODING='utf-8'` trong PowerShell để hỗ trợ tiếng Việt toàn diện.
- **Cách xác minh:** Chạy `python script/run_phase1.py` và `pytest -v` thành công 100% với exit code 0.

---

## 12. Giới hạn và hướng cải thiện

| Giới hạn hiện tại          | Ảnh hưởng                                          | Hướng cải thiện                                           |
| -------------------------- | -------------------------------------------------- | --------------------------------------------------------- |
| Bộ dữ liệu tĩnh 24 bài báo | Chưa bao quát hết các dạng schema lạ               | Tích hợp streaming ingestion định kỳ qua Cron/Airflow     |
| Heuristic QA matching      | Chưa tận dụng hết sức mạnh Reasoning của Large LLM | Bật provider `gemini` hoặc `openai` với Structured Output |
| Ephemeral GX Context       | Kết quả kiểm định chỉ lưu file JSON cục bộ         | Kết nối GX Data Docs lên Cloud Storage hoặc S3            |

---

## 13. Checklist trước khi nộp

- [x] Thông tin cá nhân và repository chính xác (Nguyễn Anh Hoàng - 2A202602816).
- [x] Phân công khớp với module, artifact và kết quả thực tế.
- [x] Lệnh tái hiện đã được chạy lại trên phiên bản dùng để nộp.
- [x] Baseline, corrupted và repaired dùng cùng evaluation set (10 câu).
- [x] Bảng metrics khớp với các file trong `data/results/`.
- [x] Quality/freshness conclusions khớp với `data/quality/`.
- [x] Các đường dẫn báo cáo và artifact truy cập được.
- [x] Báo cáo vai trò riêng đã được hoàn thành.
- [x] Tuyệt đối không commit `.env`, API key, token hoặc secret.
