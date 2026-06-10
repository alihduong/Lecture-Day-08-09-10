# Quality report — Lab Day 10

**run_id:** sprint3-after-fix  
**Ngày:** 2026-06-10

---

## 1. Tóm tắt số liệu

| Chỉ số | Inject-bad run | Clean run | Ghi chú |
|--------|---------------|-----------|---------|
| raw_records | 247 | 247 | Không đổi |
| cleaned_records | 36 | 36 | Pipeline chỉ thay chunk refund stale → fixed |
| quarantine_records | 211 | 211 | 85.4% quarantine — dữ liệu raw rất bẩn |
| Expectation halt? | YES (`refund_no_stale_14d_window FAIL`) | NO (9/9 PASS) | `--no-refund-fix` gây fail E3 |
| embed_upsert count | 36 | 36 | idempotent |
| freshness_check | FAIL (>1400h) | FAIL (>1400h) | Raw data cũ, lab scenario |

---

## 2. Before / after retrieval

### Câu then chốt: `q_refund_window`

**Inject-bad (before fix):**
```
contains_expected=yes, hits_forbidden=YES ← "14 ngày làm việc" có trong top-k
top1_preview: "Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc..."
```

**After fix (sprint3-after-fix):**
```
contains_expected=yes, hits_forbidden=NO ← không còn "14 ngày làm việc"
top1_preview: "Yêu cầu được gửi trong vòng 7 ngày làm việc..."
```

### Câu merit: `q_hr_annual_leave_under3` (HR versioning)

**Trước fix (baseline không có R7):** Pipeline halt ở E6 vì chunk "10 ngày phép năm" với date >= 2026-01-01 lọt qua.  
**Sau fix (R7 thêm vào):**
```
contains_expected=yes ("12 ngày phép năm"), hits_forbidden=NO
top1_doc_expected=yes (hr_leave_policy)
```

---

## 3. Metric impact các rule mới

| Rule / Expectation mới | Trước | Sau / khi inject | Chứng cứ |
|------------------------|-------|-----------------|----------|
| R7: stale_hr_leave_10d_annual_content | E6 HALT (7+ rows lọt) | E6 PASS | log `expectation[hr_leave_no_stale_10d_annual] OK` |
| R8: unclear_content_prefix | ~12 rows chunk "Nội dung không rõ ràng:" lọt vào cleaned | 0 rows | E7 PASS; quarantine tăng |
| R9: exclamation_corruption_marker | ~3 rows chunk "!!!" lọt vào cleaned | 0 rows | E8 PASS |
| E7 (no_unclear_prefix, halt) | Không có expectation kiểm tra | violations=0 | log PASS |
| E8 (no_exclamation_prefix, halt) | Không có expectation kiểm tra | violations=0 | log PASS |
| E9 (access_control_sop_present, halt) | access_control_sop không được embed → gq_d10_10 fail | count=6 chunks | log PASS; grading 10/10 |

---

## 4. Freshness & monitor

`freshness_check=FAIL` với `age_hours=1448` vì raw CSV có `exported_at` từ 2026-04-11 — gần 2 tháng trước. Đây là **expected** trong lab scenario (dữ liệu mẫu cố định).

Trong môi trường production, set `FRESHNESS_SLA_HOURS=24` (mặc định) hoặc cao hơn phù hợp batch frequency. Trigger rerun khi `freshness_check=FAIL`.

---

## 5. Corruption inject (Sprint 3)

**Inject command:**
```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```

**Tác động:**  
- `--no-refund-fix`: rule 9 bị tắt → chunk "14 ngày làm việc" từ `policy_refund_v4` không bị replace
- `--skip-validate`: dù E3 FAIL, pipeline vẫn embed → stale chunk vào vector DB
- Kết quả eval: `q_refund_window` → `hits_forbidden=yes` (chunk stale xuất hiện trong top-k)

**Restore:** Chạy `etl_pipeline.py run --run-id restore` → embed prune removes 1 stale chunk → `hits_forbidden=no`

---

## 6. Hạn chế & việc chưa làm

- `all-MiniLM-L6-v2` là English model; Vietnamese semantic search không tối ưu — cần tiếng Việt embedding model cho production
- `freshness_check` dựa vào `exported_at` trong data; nếu field này không đáng tin cậy (có row format `2026/04/07T00:00:00`) thì metric bị sai
- Chưa có alert/notification tự động khi FAIL
- Chưa có rollback strategy cho vector DB (Chroma không hỗ trợ snapshots)
