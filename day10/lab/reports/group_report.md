# Báo Cáo Nhóm - Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Day10 Lab Team  
**Thành viên:**

| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Người thực hiện | Ingestion / Raw Owner | student@example.com |
| Người thực hiện | Cleaning & Quality Owner | student@example.com |
| Người thực hiện | Embed & Idempotency Owner | student@example.com |
| Người thực hiện | Monitoring / Docs Owner | student@example.com |

**Ngày nộp:** 2026-06-10  
**Repo:** `Lecture-Day-08-09-10/day10/lab`  
**Run sạch cuối:** `2026-06-10T08-20Z`

---

## 1. Pipeline tổng quan

Pipeline Day 10 ingest dữ liệu raw từ `data/raw/policy_export_dirty.csv`, clean bằng `transform/cleaning_rules.py`, validate bằng `quality/expectations.py`, rồi publish embeddings vào Chroma collection `day10_kb`. Run sạch cuối có `raw_records=247`, `cleaned_records=37`, `quarantine_records=210`, manifest tại `artifacts/manifests/manifest_2026-06-10T08-20Z.json`.

Pipeline đã sửa baseline để xử lý đủ 5 nguồn canonical: refund policy, SLA P1, IT helpdesk FAQ, HR leave policy, và access control SOP. Sau clean, index được upsert theo `chunk_id` ổn định và prune vector cũ để tránh giữ lại dữ liệu stale từ lần inject.

**Lệnh chạy một dòng:**

```powershell
python etl_pipeline.py run; python grading_run.py --out artifacts/eval/grading_run.jsonl
```

---

## 2. Cleaning & expectation

Nhóm thêm rule để không bỏ sót nguồn hợp lệ và chặn các lỗi dữ liệu ảnh hưởng trực tiếp tới answer quality. Thay đổi quan trọng nhất là thêm `access_control_sop` vào allowlist, quarantine HR 2025 text chứa `10 ngày phép năm`, quarantine `exported_at` sai format, quarantine chunk mơ hồ, và canonicalize chunk P1 escalation để fact `10 phút` retrievable rõ ràng.

### 2a. Bảng metric_impact

| Rule / Expectation mới | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ |
|------------------------|-----------------|-----------------------------|----------|
| `allow_access_control_sop` | `access_control_sop` bị tính vào `unknown_doc_id`; previous unknown count 117 | run sạch có `access_control_rows=6`, unknown count giảm còn 109 | `quality[access_control_present]`, `grading gq_d10_10=True` |
| `stale_hr_2025_annual_leave_text` | pipeline halt do `hr_leave_no_stale_10d_annual violations=2` | run sạch `violations=0`, quarantine reason count `stale_hr_2025_annual_leave_text=6` | `artifacts/quarantine/quarantine_2026-06-10T08-20Z.csv` |
| `invalid_exported_at_format` | rows có `exported_at` dạng `2026/04/...` có thể lọt qua | quarantine reason `invalid_exported_at_format=6`, expectation `bad_exported_at=0` | `quality[exported_at_iso_datetime]` |
| `ambiguous_chunk_text` | chunks bắt đầu bằng `Nội dung không rõ ràng` có thể được embed | quarantine reason `ambiguous_chunk_text=5`, expectation `ambiguous_rows=0` | `quality[no_ambiguous_chunk_text]` |
| `p1_escalation_10min_present` | grading `gq_d10_06` từng fail `contains_expected=False` | grading cuối `gq_d10_06 contains_expected=True`, `top1_doc_matches=True` | `artifacts/eval/grading_run.jsonl` |

**Rule chính:**

- Allowlist đủ 5 canonical doc ids.
- Quarantine unknown doc ids và duplicate chunk text.
- Quarantine HR stale theo `effective_date < 2026-01-01`.
- Quarantine HR stale theo nội dung `10 ngày phép năm` / `bản HR 2025`.
- Canonicalize refund stale `14 ngày làm việc` thành `7 ngày làm việc` trong run sạch.
- Canonicalize P1 escalation wording để giữ fact `10 phút`.

**Ví dụ expectation fail và cách xử lý:**

Trong inject run:

```text
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=2
WARN: expectation failed but --skip-validate -> tiếp tục embed
```

Đây là lỗi cố ý của Sprint 3. Sau khi chạy lại pipeline chuẩn, expectation pass và `q_refund_window` đổi từ `hits_forbidden=yes` sang `hits_forbidden=no`.

---

## 3. Before / after ảnh hưởng retrieval

Kịch bản inject dùng:

```powershell
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
```

Sau đó restore clean index:

```powershell
python etl_pipeline.py run
python eval_retrieval.py --out artifacts/eval/after_fix_eval.csv --top-k 5
python grading_run.py --out artifacts/eval/grading_run.jsonl
```

Kết quả định lượng chính:

| Question | Inject bad | After fix |
|----------|------------|-----------|
| `q_refund_window` | `contains_expected=yes`, `hits_forbidden=yes`, `top1_doc_expected=yes` | `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` |
| `q_hr_annual_leave_under3` | `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` | `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` |
| Official grading | chưa dùng làm final | 10/10 câu pass: `contains_expected=True`, `hits_forbidden=False`, `top1_doc_matches=True` |

Evidence nằm ở `artifacts/eval/after_inject_bad.csv`, `artifacts/eval/after_fix_eval.csv`, và `artifacts/eval/grading_run.jsonl`.

---

## 4. Freshness & monitoring

Run sạch cuối có freshness `FAIL`:

```text
latest_exported_at=2026-04-11T00:00:00
age_hours=1448.343
sla_hours=24.0
reason=freshness_sla_exceeded
```

Nhóm giữ kết quả này vì data mẫu cố tình cũ. Đây không phải pipeline failure mà là observability signal: hệ thống đo được snapshot đang stale so với SLA 24 giờ. Trong production, alert sẽ gửi tới owner qua `#data-observability` để yêu cầu export mới hoặc rollback index nếu user impact cao.

---

## 5. Liên hệ Day 09

Day 09 retrieval worker chỉ trả lời đúng nếu corpus sạch. Pipeline Day 10 đảm bảo các fact nghiệp vụ quan trọng được publish đúng version: refund 7 ngày, P1 escalation 10 phút, HR 2026 12 ngày phép, và Level 4 Admin Access từ `access_control_sop`. Collection `day10_kb` có thể được dùng làm corpus cho retrieval worker trong multi-agent system.

---

## 6. Rủi ro còn lại & việc chưa làm

- Freshness vẫn FAIL vì source sample cũ; chưa cập nhật timestamp nguồn.
- Chưa dùng Great Expectations/pydantic thật, mới dùng custom expectation suite.
- Eval mở rộng có `q_p1_update_frequency` top1 chưa đúng doc dù top-k có expected answer.
