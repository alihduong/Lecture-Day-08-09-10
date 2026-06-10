# Data contract — Lab Day 10

> Đồng bộ với `contracts/data_contract.yaml` (v1.1).

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_refund_v4` | CSV export từ hệ thống CS | Chunk stale "14 ngày làm việc" (v3 lẫn v4) | `quarantine_records` tăng; E3 halt nếu không fix |
| `sla_p1_2026` | CSV export từ hệ thống ticketing | Trùng lặp nhiều row; effective_date thiếu | `quarantine_records` tăng; dedup loại bỏ |
| `it_helpdesk_faq` | CSV export từ helpdesk KB | Chunk rỗng; "Nội dung không rõ ràng:" prefix | R8 quarantine; missing_chunk_text quarantine |
| `hr_leave_policy` | CSV export từ HRIS | Xung đột version 2025 vs 2026 (10d vs 12d) | R7+R3 quarantine; E6 halt nếu stale lọt qua |
| `access_control_sop` | CSV export từ IT Security | Duplicate row; row rỗng; invalid_doc_acl001 | dedup; missing_chunk_text; unknown_doc_id |
| `invalid_doc_*` | Export lỗi từ hệ thống không rõ | Toàn bộ bị quarantine | `quarantine_records`; reason=unknown_doc_id |
| `legacy_catalog_xyz_zzz` | Hệ thống catalog cũ (deprecated) | Không có tài liệu canonical | `quarantine_records`; reason=unknown_doc_id |
| `security_policy`, `data_privacy_guideline` | Nguồn ngoài phạm vi lab | Không có trong allowlist | `quarantine_records`; reason=unknown_doc_id |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| `chunk_id` | string | Có | SHA-256(doc_id + chunk_text + seq)[:16], stable |
| `doc_id` | string | Có | Phải thuộc `ALLOWED_DOC_IDS` (5 nguồn) |
| `chunk_text` | string | Có | Tối thiểu 8 ký tự; không có prefix mơ hồ |
| `effective_date` | string YYYY-MM-DD | Có | Đã chuẩn hoá từ DD/MM/YYYY nếu cần |
| `exported_at` | string datetime | Có | Dùng để tính freshness SLA |

---

## 3. Quy tắc quarantine vs drop

Records **quarantined** (ghi vào `artifacts/quarantine/*.csv` kèm `reason`):
- `unknown_doc_id`: doc_id ngoài allowlist
- `missing_effective_date` / `invalid_effective_date_format`: ngày không parse được
- `stale_hr_policy_effective_date`: HR có effective_date < 2026-01-01
- `unclear_content_prefix`: text bắt đầu bằng "Nội dung không rõ ràng:"
- `exclamation_corruption_marker`: text bắt đầu bằng "!!!"
- `missing_chunk_text`: chunk rỗng sau strip
- `stale_hr_leave_10d_annual_content`: HR chứa "10 ngày phép năm" (bản 2025)
- `duplicate_chunk_text`: nội dung trùng (giữ bản đầu tiên)

**Ai approve merge lại?** Quarantine records không được tự động re-ingest. Cần review thủ công và cập nhật source system trước khi chạy lại pipeline.

---

## 4. Phiên bản & canonical

- **Policy refund**: canonical = `data/docs/policy_refund_v4.txt`; phiên bản chính xác = 7 ngày làm việc (rule R9 fix `cleaning_rules.py`)
- **HR leave**: canonical = `data/docs/hr_leave_policy.txt` (2026); bản 2025 (10 ngày phép năm) bị quarantine bởi R3+R7
- **SLA P1**: canonical = `data/docs/sla_p1_2026.txt`
- **IT FAQ**: canonical = `data/docs/it_helpdesk_faq.txt`
- **Access control**: canonical = `data/docs/access_control_sop.txt`

Cutoff versioning HR: `hr_leave_min_effective_date = 2026-01-01` (định nghĩa trong `data_contract.yaml`).
