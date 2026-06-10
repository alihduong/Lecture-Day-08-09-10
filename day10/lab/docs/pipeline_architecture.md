# Kiến trúc pipeline — Lab Day 10

**Nhóm:** HoangDuong (solo) — AI in Action AICB-P1  
**Cập nhật:** 2026-06-10

---

## 1. Sơ đồ luồng

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Raw Export                                                              │
│  data/raw/policy_export_dirty.csv  (247 rows, 5 hệ thống nguồn)        │
└─────────────────┬───────────────────────────────────────────────────────┘
                  │ load_raw_csv()
                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  INGEST (etl_pipeline.py → cmd_run)                                     │
│  • Ghi run_id (UTC timestamp hoặc tham số --run-id)                    │
│  • Log raw_records count                                                │
└─────────────────┬───────────────────────────────────────────────────────┘
                  │ clean_rows()
                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  TRANSFORM (transform/cleaning_rules.py)                                │
│  Rule 1: doc_id allowlist (5 nguồn hợp lệ)                             │
│  Rule 2: chuẩn hoá effective_date → YYYY-MM-DD                         │
│  Rule 3: HR stale by date (< 2026-01-01)                               │
│  Rule R8: quarantine "Nội dung không rõ ràng:" prefix                  │
│  Rule R9: quarantine "!!!" corruption marker                            │
│  Rule 6: empty chunk_text                                               │
│  Rule R7: HR stale content "10 ngày phép năm" (kể cả date >= 2026)    │
│  Rule 8: deduplicate chunk_text                                         │
│  Rule 9: fix stale refund 14d→7d (có thể tắt với --no-refund-fix)     │
│                                                                         │
│  Output: cleaned/*.csv + quarantine/*.csv                               │
│  Log: cleaned_records, quarantine_records                               │
└─────────────────┬───────────────────────────────────────────────────────┘
                  │ run_expectations()
                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  QUALITY / VALIDATE (quality/expectations.py)                           │
│  E1: min_one_row (halt)                                                 │
│  E2: no_empty_doc_id (halt)                                             │
│  E3: refund_no_stale_14d_window (halt)                                  │
│  E4: chunk_min_length_8 (warn)                                          │
│  E5: effective_date_iso_yyyy_mm_dd (halt)                               │
│  E6: hr_leave_no_stale_10d_annual (halt)                                │
│  E7: no_unclear_content_prefix (halt) ← MỚI                            │
│  E8: no_exclamation_prefix (halt) ← MỚI                                │
│  E9: access_control_sop_present (halt) ← MỚI                           │
│                                                                         │
│  Nếu halt: PIPELINE_HALT (exit 2), không embed                         │
│  Nếu warn: tiếp tục embed                                               │
└─────────────────┬───────────────────────────────────────────────────────┘
                  │ cmd_embed_internal()
                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  EMBED (chromadb + SentenceTransformer all-MiniLM-L6-v2)               │
│  • Prune: xoá id cũ không còn trong cleaned (index snapshot)           │
│  • Upsert theo chunk_id (idempotent)                                    │
│  • Metadata: doc_id, effective_date, run_id                             │
│  • Collection: day10_kb (CHROMA_COLLECTION env)                         │
└─────────────────┬───────────────────────────────────────────────────────┘
                  │ write manifest
                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  MANIFEST (artifacts/manifests/manifest_<run-id>.json)                  │
│  Ghi: run_id, raw/cleaned/quarantine count, latest_exported_at          │
└─────────────────┬───────────────────────────────────────────────────────┘
                  │ check_manifest_freshness()
                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  MONITOR (monitoring/freshness_check.py)                                │
│  • Đọc latest_exported_at từ manifest                                   │
│  • So sánh với now() → PASS/WARN/FAIL theo SLA_HOURS (mặc định 24h)   │
└─────────────────────────────────────────────────────────────────────────┘
```

**Điểm đo freshness:** trường `latest_exported_at` trong manifest (max của `exported_at` trong cleaned data)  
**run_id:** ghi trong manifest JSON và mỗi dòng log  
**Quarantine:** `artifacts/quarantine/quarantine_<run-id>.csv`

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | File chính |
|------------|-------|--------|------------|
| Ingest | CSV raw path | List[Dict] rows | `etl_pipeline.py:load_raw_csv` |
| Transform | List[Dict] raw rows | (cleaned, quarantine) | `transform/cleaning_rules.py` |
| Quality | List[Dict] cleaned rows | (results, halt) | `quality/expectations.py` |
| Embed | cleaned CSV path | Chroma upsert | `etl_pipeline.py:cmd_embed_internal` |
| Monitor | manifest JSON path | PASS/WARN/FAIL | `monitoring/freshness_check.py` |

---

## 3. Idempotency & rerun

Pipeline **idempotent** theo thiết kế:
- `chunk_id` được sinh bằng `SHA-256(doc_id + chunk_text + seq)[:16]` → stable
- Embed dùng **upsert** theo `chunk_id` → rerun 2 lần không tạo duplicate vector
- Trước mỗi upsert, pipeline **prune** (xoá) các id cũ không còn trong cleaned run hiện tại → index luôn là snapshot của run cuối

---

## 4. Liên hệ Day 09

Lab Day 09 dùng cùng `data/docs/` (5 file TXT). Lab Day 10 thêm tầng ingest từ CSV raw export, clean, validate trước khi embed vào Chroma. Collection `day10_kb` là bản cleaned mới nhất và có thể thay thế corpus Day 09 để agent trả lời chính xác hơn (không còn chunk stale "14 ngày" hay "10 ngày phép").

---

## 5. Rủi ro đã biết

- Python 3.14 chưa tương thích với sentence-transformers — cần Python 3.11/3.12
- `exported_at` có format không nhất quán (`2026/04/07T00:00:00`) ảnh hưởng freshness SLA nếu dùng max
- Allowlist cứng trong code — cần đồng bộ manual với `contracts/data_contract.yaml`
- Stale HR content có thể tồn tại với effective_date tương lai nếu không có rule R7
