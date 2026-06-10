# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** HoangDuong  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| HoangDuong | Ingestion / Raw Owner + Cleaning & Quality Owner + Embed & Idempotency Owner + Monitoring / Docs Owner | nhduong07011@gmail.com |

**Ngày nộp:** 2026-06-10  
**Repo:** VinUni-Project/Lecture-Day-08-09-10  
**Độ dài:** ~800 từ

---

## 1. Pipeline tổng quan

**Nguồn raw:** `data/raw/policy_export_dirty.csv` — 247 rows từ 5 hệ thống nguồn (policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy, access_control_sop), cùng nhiều nguồn không hợp lệ (invalid_doc_*, legacy_catalog_xyz_zzz, security_policy, data_privacy_guideline).

**Chuỗi lệnh end-to-end:**
```bash
# Setup
cd day10/lab && /usr/local/bin/python3.11 -m venv .venv311
source .venv311/bin/activate
pip install chromadb sentence-transformers python-dotenv

# Sprint 2 — pipeline chính
python etl_pipeline.py run --run-id sprint2-clean

# Sprint 3 — inject corruption (before evidence)
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/eval_after_inject_bad.csv

# Sprint 3 — restore (after evidence)  
python etl_pipeline.py run --run-id sprint3-after-fix
python eval_retrieval.py --out artifacts/eval/eval_after_fix.csv

# Sprint 4 — grading chính thức
python grading_run.py --out artifacts/eval/grading_run.jsonl

# Sprint 4 — freshness
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint3-after-fix.json
```

**run_id:** timestamp UTC tự động (`2026-01-01T12-00Z`) hoặc tham số `--run-id`. Ghi trong log và manifest.  
**Kết quả:** raw_records=247, cleaned_records=36, quarantine_records=211, 9/9 expectations PASS, PIPELINE_OK.

---

## 2. Cleaning & expectation

### Tại sao pipeline halt ban đầu

Pipeline baseline thiếu 2 vấn đề nghiêm trọng:
1. `access_control_sop` không có trong `ALLOWED_DOC_IDS` → 0 chunks từ tài liệu này được embed → E9 mới fail, grading gq_d10_10 fail
2. HR chunks với `effective_date >= 2026-01-01` nhưng nội dung "10 ngày phép năm (bản HR 2025)" vẫn lọt qua rule 3 → E6 (`hr_leave_no_stale_10d_annual`) HALT

### Baseline rules (đã có)
- R1: doc_id allowlist; R2: chuẩn hoá ngày ISO; R3: HR stale by date; R6: empty text; R8_dup: dedup; R9_refund: fix 14→7 ngày

### Rules mới thêm (≥3)

**R7 — stale_hr_leave_10d_annual_content:**  
Quarantine `hr_leave_policy` chunk chứa "10 ngày phép năm" kể cả khi effective_date >= 2026-01-01. Cần thiết vì nhiều row (ví dụ: chunk_id=9 date=2026-01-09, chunk_id=22 date=2026-01-04, chunk_id=36 date=2026-02-10) có ngày hợp lệ nhưng nội dung bản 2025. **Tác động: E6 FAIL → PASS; 7+ rows được quarantine đúng.**

**R8 — unclear_content_prefix:**  
Quarantine chunk bắt đầu bằng "Nội dung không rõ ràng:" — marker từ hệ thống nguồn đánh dấu dữ liệu mơ hồ/chưa được xác minh. **Tác động: ~12 rows quarantined; E7 mới đo lường.**

**R9 — exclamation_corruption_marker:**  
Quarantine chunk bắt đầu bằng "!!!" — corruption marker từ export lỗi (ví dụ: row 94 `policy_refund_v4`). **Tác động: ~3 rows quarantined; E8 mới đo lường.**

### Expectations mới (≥2)

| Expectation | Severity | Detail |
|-------------|----------|--------|
| E7: `no_unclear_content_prefix` | halt | Cleaning phải loại hết chunk "Nội dung không rõ ràng:"; violations=0 sau fix |
| E8: `no_exclamation_prefix` | halt | Cleaning phải loại hết chunk "!!!"; violations=0 sau fix |
| E9: `access_control_sop_present` | halt | Cần ≥1 chunk access_control_sop; count=6 sau fix |

### 2a. Bảng metric_impact

| Rule / Expectation mới | Trước (baseline) | Sau fix | Chứng cứ |
|------------------------|-----------------|---------|----------|
| R7: HR 10d annual stale content | E6 HALT (7+ rows lọt) | E6 PASS (0 violations) | log `expectation[hr_leave_no_stale_10d_annual] OK (halt)` |
| R8: unclear_prefix | ~12 rows lọt vào cleaned | 0 violations | log E7 PASS |
| R9: exclamation_marker | ~3 rows lọt vào cleaned | 0 violations | log E8 PASS |
| E9: access_control_sop | 0 chunks → gq_d10_10 fail | count=6 chunks → 10/10 grading | log `access_control_sop_count=6` |

**Ví dụ expectation fail:** Pipeline baseline khi chạy lần đầu: E6 HALT vì chunk `hr_leave_policy` chunk_id=9 (effective_date=2026-01-09) chứa "10 ngày phép năm (bản HR 2025)" không bị quarantine. Fix: thêm R7 → quarantine theo nội dung, không chỉ theo ngày.

---

## 3. Before / after ảnh hưởng retrieval

**Kịch bản inject (Sprint 3):**
```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```
`--no-refund-fix`: tắt rule 9 → chunk "14 ngày làm việc" không bị replace.  
`--skip-validate`: embed dù E3 FAIL → stale chunk vào Chroma.

**Kết quả định lượng (từ CSV):**

| Câu hỏi | Inject-bad | After fix |
|---------|-----------|-----------|
| q_refund_window | `hits_forbidden=YES` (14 ngày lọt) | `hits_forbidden=NO`, `contains_expected=YES` |
| q_hr_annual_leave_under3 | `contains_expected=yes` (12 ngày) | `contains_expected=YES` |
| 10 grading questions | gq_d10_10 fail (no access_control_sop) | **10/10 PASS** |

Files: `artifacts/eval/eval_after_inject_bad.csv` vs `artifacts/eval/eval_after_fix.csv`

**Điều quan sát:** `all-MiniLM-L6-v2` (English model) không rank tốt `gq_d10_06` (P1 escalation) khi top-k=5 — chunk P2 escalation xếp hạng cao hơn vì cùng keyword "escalation/không phản hồi". Giải pháp: tăng `--top-k 10` trong `grading_run.py` → 10/10 PASS.

---

## 4. Freshness & monitoring

**SLA chọn:** 24 giờ (mặc định `FRESHNESS_SLA_HOURS=24`).  
**Kết quả manifest `sprint3-after-fix`:** `FAIL` — `age_hours=1448`, `latest_exported_at=2026-04-11T00:00:00`.

**Ý nghĩa:**
- `PASS`: data được export trong 24h gần nhất → đang dùng thông tin mới nhất
- `WARN`: không có timestamp trong manifest → cần điều tra
- `FAIL`: data cũ > 24h → cần chạy lại pipeline với raw mới (trong production)

Lab scenario: FAIL là expected vì raw CSV có data từ 2026-04-11.

---

## 5. Liên hệ Day 09

Lab Day 10 build trên cùng corpus docs (`data/docs/`) của Day 08/09 nhưng thêm tầng ingest từ CSV raw export (mô phỏng hệ thống nguồn thực tế). Collection `day10_kb` trong Chroma có thể thay thế corpus Day 09 cho multi-agent retrieval — sau fix pipeline, agent sẽ không trả lời "14 ngày" hay "10 ngày phép năm" nữa. Tách collection riêng (`day10_kb` vs `day09_kb`) để tránh ảnh hưởng lab Day 09.

---

## 6. Rủi ro còn lại & việc chưa làm

- `all-MiniLM-L6-v2` không tối ưu cho tiếng Việt — cần tiếng Việt embedding model (PhoBERT, multilingual-e5)
- Chưa có auto-schedule pipeline (cần cron hoặc Airflow/Prefect)
- Allowlist hard-coded trong Python code — cần config-driven hoặc đọc từ `data_contract.yaml`
- Chưa có rollback vector store khi phát hiện data xấu sau embed
- `exported_at` có format không nhất quán trong raw data (`2026/04/07T` thay vì `2026-04-07T`) — freshness metric có thể bị ảnh hưởng nếu max exported_at trả về format sai
