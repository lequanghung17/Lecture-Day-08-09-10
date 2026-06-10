# Data contract - Lab Day 10

> Đồng bộ với `contracts/data_contract.yaml` và run sạch `2026-06-10T08-20Z`.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|--------------------|--------------------|----------------|
| `policy_refund_v4` | CSV export từ policy KB | stale refund window 14 ngày, duplicate chunk | `refund_no_stale_14d_window`, `hits_forbidden` |
| `sla_p1_2026` | CSV export từ SLA document | P1 escalation bị retrieve kém, P2 escalation lẫn vào top-k | `p1_escalation_10min_present`, grading `gq_d10_06` |
| `it_helpdesk_faq` | CSV export từ FAQ | duplicate FAQ chunk, wrong doc retrieved | top1 doc in eval/grading |
| `hr_leave_policy` | CSV export từ HR policy | conflict HR 2025 vs 2026, `10 ngày phép năm` stale | `hr_leave_no_stale_10d_annual` |
| `access_control_sop` | CSV export từ access SOP | baseline allowlist bỏ qua nguồn hợp lệ | `access_control_present`, grading `gq_d10_10` |

Các nguồn không có canonical file như `invalid_doc_*`, `legacy_catalog_xyz_zzz`, `security_policy`, `data_privacy_guideline` bị quarantine bằng reason `unknown_doc_id`.

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| `chunk_id` | string | Có | Stable id để upsert/prune Chroma |
| `doc_id` | string | Có | Một trong 5 allowed doc ids |
| `chunk_text` | string | Có | Tối thiểu 8 ký tự; đã canonicalize một số fact |
| `effective_date` | date | Có | Chuẩn `YYYY-MM-DD`; hỗ trợ parse `DD/MM/YYYY` |
| `exported_at` | datetime | Có | Chuẩn `YYYY-MM-DDTHH:MM:SS` |

Run sạch cuối:

| Chỉ số | Giá trị |
|--------|---------|
| `run_id` | `2026-06-10T08-20Z` |
| `raw_records` | 247 |
| `cleaned_records` | 37 |
| `quarantine_records` | 210 |
| `latest_exported_at` | `2026-04-11T00:00:00` |

---

## 3. Quy tắc quarantine vs drop

Pipeline không silent drop. Mỗi row bị loại được ghi vào `artifacts/quarantine/quarantine_<run_id>.csv` với `reason`.

Quarantine reasons trong run sạch:

| Reason | Count |
|--------|------:|
| `unknown_doc_id` | 109 |
| `duplicate_chunk_text` | 48 |
| `stale_hr_policy_effective_date` | 22 |
| `missing_chunk_text` | 8 |
| `invalid_exported_at_format` | 6 |
| `missing_effective_date` | 6 |
| `stale_hr_2025_annual_leave_text` | 6 |
| `ambiguous_chunk_text` | 5 |

Record chỉ được restore khi owner nguồn xác nhận nội dung canonical và pipeline maintainer cập nhật allowlist/rule tương ứng.

---

## 4. Phiên bản & canonical

Canonical sources:

- Refund: `data/docs/policy_refund_v4.txt`, current window = 7 ngày làm việc.
- SLA P1: `data/docs/sla_p1_2026.txt`, P1 escalation = 10 phút.
- IT FAQ: `data/docs/it_helpdesk_faq.txt`.
- HR leave: `data/docs/hr_leave_policy.txt`, HR 2026 under-3-year leave = 12 ngày phép năm.
- Access control: `data/docs/access_control_sop.txt`, Level 4 Admin Access cần IT Manager/CISO.

Rule versioning quan trọng:

- HR stale rows có `effective_date < 2026-01-01` bị quarantine.
- HR text chứa `10 ngày phép năm` hoặc `bản HR 2025` bị quarantine dù `effective_date` mới.
- Refund text `14 ngày làm việc` được canonicalize thành `7 ngày làm việc` khi không chạy `--no-refund-fix`.
