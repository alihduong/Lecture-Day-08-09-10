# Runbook — Lab Day 10

---

## Symptom 1 — Agent trả lời "14 ngày" hoặc "10 ngày phép năm"

**User thấy:** agent hoặc retrieval trả về thông tin sai phiên bản (policy refund stale 14-day, hoặc HR leave 10-day thay vì 12-day).

---

## Detection

| Metric | Dấu hiệu | File / Lệnh |
|--------|----------|-------------|
| `hits_forbidden=yes` trong eval | Chunk cũ lọt vào top-k | `artifacts/eval/*.csv` |
| `expectation[refund_no_stale_14d_window] FAIL` | "14 ngày làm việc" tồn tại sau clean | log file |
| `expectation[hr_leave_no_stale_10d_annual] FAIL` | "10 ngày phép năm" tồn tại sau clean | log file |
| `freshness_check=FAIL` | Data xuất > 24h trước | manifest JSON |

---

## Symptom 2 — PIPELINE_HALT

**Nguyên nhân phổ biến:**

| Expectation fail | Root cause | Fix |
|-----------------|------------|-----|
| `refund_no_stale_14d_window` | `--no-refund-fix` được dùng, hoặc rule R9 bỏ sót chunk "!!!" với "14 ngày" | Chạy lại không có `--no-refund-fix`; kiểm tra R9 |
| `hr_leave_no_stale_10d_annual` | HR chunk có date >= 2026-01-01 nhưng nội dung 2025 lọt qua | Kiểm tra rule R7 trong cleaning_rules.py |
| `access_control_sop_present` | `access_control_sop` bị xoá khỏi ALLOWED_DOC_IDS | Thêm lại vào allowlist; đồng bộ data_contract.yaml |
| `effective_date_iso_yyyy_mm_dd` | Parser chuẩn hoá ngày thiếu case | Thêm format mới vào `_normalize_effective_date()` |
| `no_unclear_content_prefix` | Rule R8 không bắt được variant mới | Cập nhật regex/string match trong cleaning_rules.py |

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Mở `artifacts/manifests/manifest_<run-id>.json` | Xem `raw_records`, `cleaned_records`, `quarantine_records` |
| 2 | Mở `artifacts/quarantine/<run-id>.csv` | Tìm `reason` phổ biến, xác định doc_id bị quarantine nhầm/đúng |
| 3 | Mở `artifacts/logs/run_<run-id>.log` | Xem expectation nào FAIL và detail |
| 4 | Chạy `python eval_retrieval.py --out /tmp/test.csv` | Kiểm tra `contains_expected`, `hits_forbidden` |
| 5 | Chạy `python grading_run.py --out /tmp/grading.jsonl` | Verify 10 câu grading official |

---

## Mitigation

**Stale data trong vector DB:**
```bash
# Chạy lại pipeline chuẩn — sẽ prune chunk cũ và upsert chunk mới
python etl_pipeline.py run --run-id hotfix-$(date +%Y%m%d)
```

**Inject corruption (Sprint 3 demo):**
```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/eval_after_inject_bad.csv
# Sau đó restore:
python etl_pipeline.py run --run-id restore-clean
```

**Rollback vector store:** Chroma chưa hỗ trợ rollback native — cần xoá `chroma_db/` và rerun pipeline với run-id tốt nhất.

---

## Freshness: PASS / WARN / FAIL

| Status | Ý nghĩa | Hành động |
|--------|---------|-----------|
| `PASS` | `latest_exported_at` ≤ 24h trước | Không cần làm gì |
| `WARN` | Không có timestamp trong manifest | Kiểm tra exported_at trong raw CSV |
| `FAIL` | Data cũ > 24h (SLA exceeded) | Chạy lại `etl_pipeline.py run` với raw mới |

Freshness FAIL là bình thường trong lab vì `exported_at` trong raw CSV là ngày 2026-04-xx (>24h so với now). Trong môi trường production, cần set `FRESHNESS_SLA_HOURS` phù hợp hoặc schedule pipeline chạy tự động.

---

## Prevention

- Thêm expectation mới khi phát hiện failure mode mới (ví dụ: `no_legacy_prefix` nếu có "bản cũ:" marker)
- Monitor `quarantine_records / raw_records` ratio — tỉ lệ cao (>80%) cần điều tra
- Đồng bộ `ALLOWED_DOC_IDS` với `contracts/data_contract.yaml` khi thêm tài liệu mới
- Chạy `grading_run.py` sau mỗi thay đổi pipeline trước khi commit
