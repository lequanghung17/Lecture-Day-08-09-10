# Runbook - Lab Day 10

**Run sạch tham chiếu:** `2026-06-10T08-20Z`  
**Collection:** `day10_kb`

---

## Symptom

User hoặc agent thấy một trong các triệu chứng sau:

- Refund answer nói khách có `14 ngày làm việc`, trong khi policy hiện hành là `7 ngày làm việc`.
- HR answer nói nhân viên dưới 3 năm kinh nghiệm có `10 ngày phép năm`, trong khi policy HR 2026 là `12 ngày phép năm`.
- Câu hỏi Level 4 Admin Access không retrieve được `access_control_sop`.
- Câu hỏi P1 auto escalation không trả về `10 phút`.

---

## Detection

Metric/check cần xem trước khi debug prompt/model:

- `expectation[refund_no_stale_14d_window]`
- `expectation[hr_leave_no_stale_10d_annual]`
- `expectation[access_control_present]`
- `expectation[p1_escalation_10min_present]`
- Eval CSV: `hits_forbidden`, `contains_expected`, `top1_doc_expected`
- Grading JSONL: `contains_expected`, `hits_forbidden`, `top1_doc_matches`

Evidence chính:

- Inject bad: `artifacts/eval/after_inject_bad.csv`
- After fix: `artifacts/eval/after_fix_eval.csv`
- Grading: `artifacts/eval/grading_run.jsonl`

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Mở `artifacts/manifests/manifest_<run_id>.json` | Thấy `raw_records`, `cleaned_records`, `quarantine_records`, `latest_exported_at` |
| 2 | Mở `artifacts/quarantine/quarantine_<run_id>.csv` | Xác nhận stale/invalid rows đã có `reason` cụ thể |
| 3 | Chạy `python eval_retrieval.py --out artifacts/eval/after_fix_eval.csv --top-k 5` | `contains_expected=yes`, `hits_forbidden=no` cho các câu chính |
| 4 | Chạy `python grading_run.py --out artifacts/eval/grading_run.jsonl` | 10/10 câu official pass |
| 5 | Nếu fail, inspect top-k cho câu fail | Xác định chunk đúng bị thiếu, bị stale, hay rơi ngoài top-k |

Run sạch cuối có:

- `cleaned_records=37`
- `quarantine_records=210`
- `grading_run.jsonl`: 10/10 câu có `contains_expected=True`, `hits_forbidden=False`, `top1_doc_matches=True`

---

## Mitigation

Nếu phát hiện stale data hoặc retrieval sai:

1. Dừng publish index mới nếu expectation halt fail.
2. Nếu đang incident production, rollback về manifest/index sạch gần nhất.
3. Sửa rule trong `transform/cleaning_rules.py` hoặc expectation trong `quality/expectations.py`.
4. Rerun pipeline chuẩn:

```powershell
python etl_pipeline.py run
```

5. Rerun grading:

```powershell
python grading_run.py --out artifacts/eval/grading_run.jsonl
```

6. Chỉ publish khi grading chính thức pass và không còn forbidden stale text trong top-k.

---

## Prevention

Action items đã thêm trong pipeline:

- Add `access_control_sop` vào allowlist để không quarantine nhầm nguồn hợp lệ.
- Quarantine HR 2025 annual leave text bằng `stale_hr_2025_annual_leave_text`.
- Quarantine `exported_at` sai ISO bằng `invalid_exported_at_format`.
- Quarantine ambiguous chunks bằng `ambiguous_chunk_text`.
- Thêm expectation `p1_escalation_10min_present` để không mất fact SLA P1 quan trọng.

Freshness hiện `FAIL` vì sample data cũ:

```text
latest_exported_at=2026-04-11T00:00:00
sla_hours=24
reason=freshness_sla_exceeded
```

Trong lab, đây là kết quả hợp lý để ghi nhận observability. Trong production, cần alert owner qua `#data-observability` và yêu cầu source export mới.
