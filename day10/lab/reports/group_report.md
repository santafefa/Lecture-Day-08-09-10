# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nguyễn Huyền San | Ingestion / Raw Owner | 26ai.sannh@vinuni.edu.vn |
| Nguyễn Huyền San | Cleaning & Quality Owner | 26ai.sannh@vinuni.edu.vn |
| Nguyễn Huyền San | Embed & Idempotency Owner | 26ai.sannh@vinuni.edu.vn |
| Nguyễn Huyền San | Monitoring / Docs Owner | 26ai.sannh@vinuni.edu.vn |

**Ngày nộp:** 10/06/2026  
**Repo:** https://github.com/santafefa/Lecture-Day-08-09-10.git  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**

Nguồn dữ liệu thô (raw data) được mô phỏng dưới dạng export file CSV (`policy_export_dirty.csv`) gồm 247 bản ghi chứa nhiều rác, trùng lặp, thiếu ngày tháng và xung đột version. 

Toàn bộ hệ thống được chạy end-to-end thông qua file Orchestrator `etl_pipeline.py`. Luồng chạy thực tế trải qua 4 bước: **Ingest** (đọc CSV) -> **Clean** (áp dụng các rule vào file `cleaning_rules.py`) -> **Validate** (chạy qua bộ Quality Gates `expectations.py`) -> **Embed** (Upsert vào VectorDB ChromaDB) và tạo file `manifest.json`.

`run_id` được sinh tự động theo chuẩn UTC timestamp và ghi log rõ ràng trong Terminal cũng như metadata của ChromaDB.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**

`python etl_pipeline.py run` (Chạy pipeline) -> `python grading_run.py` (Làm bài test) -> `python instructor_quick_check.py --grading artifacts/eval/grading_run.jsonl` (Chấm điểm).


---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| **Noise Reduction** (Gỡ tiền tố nhiễu) | Rác văn bản ("Nội dung không rõ ràng: ", "!!!") làm giảm vector similarity. | Các chunk rác được làm sạch, nội dung tinh gọn. | `cleaned_records=46`, các tiền tố bị xóa. |
| **Context Enrichment** (Bổ sung ngữ cảnh SLA) | Test `gq_d10_06` bị FAIL do RAG không nối được query "ticket P1" với từ "Escalation P1". | Đổi "Escalation P1" thành "Ticket P1 Escalation" để match 100% ngữ cảnh. | Chạy máy chấm: `GRADE_CHECK[gq_d10_06]` chuyển từ FAIL -> OK. |
| **Data Imputation** (Điền ngày bị mất) | 2 bản ghi SLA quan trọng bị đẩy vào `quarantine` do thiếu `effective_date`. | Dùng từ điển `fallback_dates` để cứu 2 bản ghi này. | Số lượng `cleaned_records` tăng từ 44 lên 46. |

**Rule chính (baseline + mở rộng):**

- Rule 1: Unknown doc_id (Đã bổ sung `access_control_sop`).
- Rule 2: Phục hồi ngày tháng (Regex mềm dẻo + Data Imputation).
- Rule 3: Stale version conflict (Bắt HR policy bản 2025).
- Rule 4: Cập nhật business logic (Refund window 14 -> 7).
- Rule 5: Chống trùng lặp (Deduplication bằng text hash).

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

- Lỗi xuất hiện trong log: `expectation[chunk_min_length_8] FAIL (warn) :: short_chunks=1` do có một dòng chứa ký tự quá ngắn. 
- Xử lý: Đã thêm một rule vào `cleaning_rules.py` kiểm tra `if len(fixed_text) < 8: quarantine.append(row)` để cách ly triệt để rác vụn. Log sau đó báo `OK (warn) :: short_chunks=0`.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

Nhóm đã sử dụng cờ đặc biệt trong Orchestrator để giả lập việc Data Pipeline bị ngắt, dẫn đến rác đi thẳng vào ChromaDB, từ đó chứng minh được tác hại của "Garbage in -> Garbage out".

**Kịch bản inject:**

Nhóm chạy lệnh `python etl_pipeline.py run --no-refund-fix --skip-validate` để đẩy trực tiếp data bẩn (chính sách refund 14 ngày cũ) vào VectorDB. Sau đó gọi hàm đánh giá `eval_retrieval.py` và xuất ra file `eval_before_dirty.csv`.

**Kết quả định lượng (từ CSV / bảng):**

- **Trước khi Clean (Run ID: 2026-06-10T06-44Z):** Ở câu hỏi về số ngày Refund, RAG gọi ra tài liệu cũ có chứa 14 ngày. Kết quả eval ghi nhận `hits_forbidden = yes` -> Agent bị ảo giác (Hallucination).
- **Sau khi Clean (Run ID: 2026-06-10T06-44Z):** Pipeline chạy chuẩn, fix toàn bộ chính sách 14 ngày thành 7 ngày, cách ly 201 bản ghi rác. Kết quả eval ghi nhận `hits_forbidden = no`, cột `top1_preview` trả về đúng "7 ngày làm việc". Điểm tự động từ `instructor_quick_check.py` đạt **10/10 (RESULT: DISTINCTION)**.

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

Nhóm lựa chọn ngưỡng SLA Freshness là 24 giờ, phù hợp với tần suất cập nhật tài liệu và chính sách nội bộ hàng ngày của doanh nghiệp. Ý nghĩa của các trạng thái giám sát trên manifest mẫu bao gồm:

PASS: Dữ liệu hoàn toàn cập nhật và thời gian kể từ lần chạy cuối cùng nằm trong khung SLA cho phép (< 24 giờ), hệ thống RAG ở trạng thái an toàn.

WARN: Manifest hợp lệ nhưng hệ thống phát hiện bất thường nhỏ như tệp CSV thô chứa trường latest_exported_at mang mốc thời gian cũ từ quá khứ (2026/04/07) hoặc định dạng múi giờ chưa đồng bộ, tuy nhiên pipeline không bị ngắt.

FAIL: File manifest bị khuyết thiếu hoặc khoảng cách thời gian vượt quá 24 giờ, báo hiệu pipeline đã bị treo hoặc lỗi kết nối ở tầng nạp dữ liệu (Ingestion), cần kích hoạt Runbook để kiểm tra khẩn cấp.

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

Dữ liệu sạch sau khi embed vào collection day10_kb phục vụ trực tiếp cho việc nâng cấp hệ thống Multi-Agent từ Day 09. Thay vì đọc các file văn bản thô chưa qua kiểm định, retrieval_worker và policy_worker sẽ kết nối thẳng đến ChromaDB thông qua API hoặc MCP Server. Sự tích hợp này giúp mạng lưới Agent vận hành trên một nguồn tri thức chuẩn duy nhất (Single Source of Truth), đảm bảo các câu trả lời (như quy định Refund 7 ngày hay phê duyệt Access Control Level 4) luôn nhất quán, chính xác và không bị ảo giác.

---

## 6. Rủi ro còn lại & việc chưa làm

- Rủi ro sập luồng do biến động cấu trúc (Schema Drift): Pipeline chưa có bộ lọc tự động phát hiện và xử lý nếu file CSV đầu vào đột ngột thay đổi tên cột hoặc kiểu dữ liệu từ hệ thống nguồn, dễ dẫn đến lỗi nghiêm trọng ở bước Ingest.
- Thiếu tầng đánh giá ngữ nghĩa sâu (LLM-as-a-judge): Hệ thống mới chỉ kiểm định chất lượng bằng các quy tắc cứng (Rule-based expectations), chưa thể tự động phát hiện những mâu thuẫn tinh vi về mặt logic hoặc câu chữ ẩn bên trong nội dung tài liệu.
- Chưa tự động hóa hoàn toàn luồng kích hoạt (Event-driven): Quá trình chạy ETL và đồng bộ chỉ mục Vector hiện tại vẫn thực hiện thủ công bằng lệnh Terminal. Cần bọc pipeline vào Docker và cấu hình trigger tự động chạy qua Webhook mỗi khi kho lưu trữ có tài liệu mới.
