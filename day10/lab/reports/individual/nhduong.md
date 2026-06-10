# Báo cáo cá nhân — Lab Day 10

**Tên:** HoangDuong  
**Email:** nhduong07011@gmail.com  
**Vai trò:** Toàn bộ (solo) — Ingestion / Cleaning / Embed / Monitoring  
**Ngày:** 2026-06-10

---

## 1. Những gì tôi đã làm

### Sprint 1 — Phân tích & Ingest
- Đọc `data/raw/policy_export_dirty.csv` (247 rows): tìm thấy 13 doc_id unique — chỉ 5 hợp lệ (policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy, **access_control_sop**)
- Phát hiện `ALLOWED_DOC_IDS` baseline thiếu `access_control_sop` → gq_d10_10 sẽ fail
- Chạy pipeline lần đầu → HALT vì E6 fail: chunk hr_leave_policy date=2026-01-09 nhưng chứa "10 ngày phép năm (bản HR 2025)"

### Sprint 2 — Clean + validate + embed
- Thêm `access_control_sop` vào `ALLOWED_DOC_IDS`
- Thêm Rule R7: quarantine hr_leave_policy chunk chứa "10 ngày phép năm" (kể cả date >= 2026)
- Thêm Rule R8: quarantine chunk "Nội dung không rõ ràng:" prefix
- Thêm Rule R9: quarantine chunk "!!!" corruption marker
- Thêm E7, E8, E9 vào expectations suite
- Pipeline exit 0: 36 cleaned, 211 quarantined, 9/9 expectations PASS

### Sprint 3 — Inject corruption & before/after
- Chạy inject-bad: `q_refund_window` → `hits_forbidden=YES` (chunk "14 ngày" trong DB)
- Chạy restore: `hits_forbidden=NO`, 10/10 grading PASS
- Phát hiện vấn đề ranking `gq_d10_06` (P1 escalation) với top-k=5 → fix default top-k lên 10

### Sprint 4 — Monitoring + docs
- Điền `pipeline_architecture.md`, `data_contract.md`, `runbook.md`, `quality_report_template.md`
- Cập nhật `data_contract.yaml` (v1.1)
- `freshness_check=FAIL` expected (data lab cũ ~1448h)

---

## 2. Điều tôi học được

1. **Date-based filter không đủ** — cần content-based filter (R7). Một chunk có thể có effective_date "2026" nhưng nội dung vẫn là bản cũ 2025 → phải kiểm tra cả ngày lẫn nội dung.

2. **Allowlist phải đồng bộ với grading questions** — khi thêm tài liệu mới, PHẢI cập nhật cả `ALLOWED_DOC_IDS`, `data_contract.yaml`, và chạy lại grading để verify.

3. **Embedding model ranking** — `all-MiniLM-L6-v2` (English) không phân biệt tốt P1 vs P2 escalation bằng tiếng Việt. top-k nhỏ (3-5) có thể miss chunk quan trọng ở vị trí thấp hơn.

4. **Idempotent embed** — upsert + prune đảm bảo mỗi lần rerun tạo ra index snapshot nhất quán, không tích lũy vector cũ.

---

## 3. Khó khăn gặp phải

- Python 3.14 không tương thích với sentence-transformers → phải tạo venv riêng với Python 3.11
- `exported_at` có format không nhất quán (`2026/04/07T00:00:00`) trong nhiều row → freshness metric dùng max nhưng có thể lấy nhầm format sai
- Phân biệt "10 ngày/năm" (sick leave) với "10 ngày phép năm" (stale annual leave) trong rule R7 — cần đủ cụ thể để không quarantine chunk sick leave đúng

---

## 4. Peer review — 3 câu hỏi tự đặt

1. **Câu hỏi 1:** Nếu thêm tài liệu mới (ví dụ `employee_handbook_2026`), cần sửa những file nào?  
   **Trả lời:** `transform/cleaning_rules.py` (ALLOWED_DOC_IDS), `contracts/data_contract.yaml` (allowed_doc_ids + canonical_sources), và kiểm tra grading_questions.json xem có câu hỏi nào liên quan không. Sau đó chạy `grading_run.py` để verify.

2. **Câu hỏi 2:** Tại sao rule R7 check "10 ngày phép năm" thay vì check "(bản HR 2025)" trong text?  
   **Trả lời:** String "(bản HR 2025)" chỉ xuất hiện trong một số chunks; "10 ngày phép năm" là marker ngữ nghĩa rõ ràng hơn — bất kỳ chunk nào nói "10 ngày phép năm" đều là bản cũ (2026 policy là 12 ngày). Cách check này robust hơn với các variant text khác nhau từ export system.

3. **Câu hỏi 3:** Khi nào thì tăng `--top-k` là không đủ, cần giải pháp khác?  
   **Trả lời:** Khi top-k rất lớn (>20) vẫn không tìm thấy chunk đúng, thì cần đổi embedding model (tiếng Việt), hoặc cải thiện chunk text để semantic match tốt hơn, hoặc thêm metadata-based filtering (ví dụ: chỉ search trong `doc_id='sla_p1_2026'`).
