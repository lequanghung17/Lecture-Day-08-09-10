# Quality report - Lab Day 10

**run_id:** `2026-06-10T08-20Z`  
**Ngày:** 2026-06-10  
**Artifacts:** `artifacts/eval/after_inject_bad.csv`, `artifacts/eval/after_fix_eval.csv`, `artifacts/eval/grading_run.jsonl`

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước / inject-bad | Sau / clean run | Ghi chú |
|--------|--------------------|-----------------|---------|
| `raw_records` | 247 | 247 | Cùng raw CSV |
| `cleaned_records` | 37 | 37 | Số row sạch giữ nguyên, nội dung refund được canonicalize ở run sạch |
| `quarantine_records` | 210 | 210 | Có reason cho từng row bị loại |
| Expectation halt? | `refund_no_stale_14d_window` FAIL nhưng `--skip-validate` | Tất cả halt expectations OK | Inject dùng để demo Sprint 3 |
| Embed count | 37 | 37 | Collection `day10_kb` |

Run sạch cuối:

```text
PIPELINE_OK
run_id=2026-06-10T08-20Z
cleaned_records=37
quarantine_records=210
embed_upsert count=37
```

---

## 2. Before / after retrieval

File evidence:

- Before/inject: `artifacts/eval/after_inject_bad.csv`
- After/fix: `artifacts/eval/after_fix_eval.csv`

**Câu then chốt:** `q_refund_window`

| Run | contains_expected | hits_forbidden | top1_doc_expected |
|-----|-------------------|----------------|-------------------|
| inject-bad | yes | yes | yes |
| after-fix | yes | no | yes |

Diễn giải: bản inject vẫn retrieve được thông tin refund, nhưng top-k còn chứa forbidden stale text `14 ngày`. Bản clean run loại bỏ forbidden hit, giữ answer đúng `7 ngày làm việc`.

**Merit evidence:** `q_hr_annual_leave_under3`

| Run | contains_expected | hits_forbidden | top1_doc_expected |
|-----|-------------------|----------------|-------------------|
| inject-bad | yes | no | yes |
| after-fix | yes | no | yes |

HR stale được kiểm soát bằng expectation `hr_leave_no_stale_10d_annual` và quarantine reason `stale_hr_2025_annual_leave_text`.

**Official grading:** `artifacts/eval/grading_run.jsonl`

```text
10/10 records: contains_expected=True, hits_forbidden=False, top1_doc_matches=True
```

---

## 3. Freshness & monitor

Freshness check trên run sạch:

```text
freshness_check=FAIL
latest_exported_at=2026-04-11T00:00:00
sla_hours=24
reason=freshness_sla_exceeded
```

Kết quả FAIL là hợp lý trong lab vì dữ liệu mẫu cố tình cũ so với ngày chạy 2026-06-10. Pipeline vẫn quan sát được vấn đề và ghi manifest đầy đủ. Trong production, đây sẽ là alert cho owner nguồn dữ liệu.

---

## 4. Corruption inject

Kịch bản inject:

```powershell
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
```

Inject tắt refund fix và bỏ qua halt validation để embed dữ liệu xấu. Expectation phát hiện:

```text
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=2
WARN: expectation failed but --skip-validate -> tiếp tục embed
```

Sau đó pipeline chuẩn được chạy lại để restore index sạch:

```powershell
python etl_pipeline.py run
python eval_retrieval.py --out artifacts/eval/after_fix_eval.csv --top-k 5
python grading_run.py --out artifacts/eval/grading_run.jsonl
```

---

## 5. Hạn chế & việc chưa làm

- Freshness SLA vẫn FAIL vì sample snapshot cũ; nhóm chỉ giải thích và ghi runbook, chưa thay source timestamp.
- Chưa tích hợp Great Expectations/pydantic thật; hiện dùng custom expectations.
- Eval mở rộng vẫn có `q_p1_update_frequency` top1 không đúng doc dù top-k có answer expected. Official grading 10 câu đã pass.
