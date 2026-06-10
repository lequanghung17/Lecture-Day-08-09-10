# Báo Cáo Cá Nhân - Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lê Quang Hưng
**Vai trò:** Cleaning / Quality Owner  
**Ngày nộp:** 2026-06-10  
**Run tham chiếu:** `2026-06-10T08-20Z`

---

## 1. Tôi phụ trách phần nào?

Tôi phụ trách phần cleaning và quality gate của pipeline Day 10. File chính tôi làm là `transform/cleaning_rules.py` và `quality/expectations.py`. Trong cleaning, tôi mở rộng allowlist để pipeline nhận `access_control_sop`, quarantine HR stale text chứa `10 ngày phép năm` hoặc `bản HR 2025`, quarantine `exported_at` sai ISO datetime, và loại chunk có nội dung mơ hồ. Trong quality suite, tôi thêm các expectation để đảm bảo access control có mặt trong cleaned data, không còn ambiguous chunk, timestamp export đúng format, và fact P1 escalation `10 phút` vẫn tồn tại.

**Bằng chứng:** `artifacts/manifests/manifest_2026-06-10T08-20Z.json`, `artifacts/eval/grading_run.jsonl`.

---

## 2. Một quyết định kỹ thuật

Quyết định kỹ thuật quan trọng là dùng halt expectation cho các lỗi ảnh hưởng trực tiếp tới answer quality. Ví dụ, `access_control_present` phải là halt vì grading có câu Level 4 Admin Access, nếu thiếu source này thì retrieval không thể trả lời đúng. Tương tự, `hr_leave_no_stale_10d_annual` và rule `stale_hr_2025_annual_leave_text` phải chặn bản HR 2025 dù một số row có `effective_date` mới, vì nội dung text vẫn nói `10 ngày phép năm`, mâu thuẫn với policy HR 2026 là `12 ngày phép năm`.

---

## 3. Một lỗi hoặc anomaly đã xử lý

Anomaly rõ nhất là pipeline ban đầu halt ở expectation `hr_leave_no_stale_10d_annual` với `violations=2`. Nguyên nhân là một số row HR có nội dung stale `10 ngày phép năm (bản HR 2025)` nhưng vẫn lọt qua rule chỉ dựa vào `effective_date`. Tôi bổ sung rule kiểm tra nội dung text để quarantine các row này bằng reason `stale_hr_2025_annual_leave_text`. Run sạch cuối có `hr_leave_no_stale_10d_annual OK`, `violations=0`, và quarantine reason này có 6 rows.

---

## 4. Bằng chứng trước / sau

Before/after chính nằm ở câu `q_refund_window`. Trong `artifacts/eval/after_inject_bad.csv`, bản inject có `contains_expected=yes` nhưng `hits_forbidden=yes`, nghĩa là top-k vẫn chứa stale `14 ngày`. Sau khi chạy lại pipeline chuẩn và tạo `artifacts/eval/after_fix_eval.csv`, cùng câu này có `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`. Grading chính thức `artifacts/eval/grading_run.jsonl` pass 10/10 với `contains_expected=True`, `hits_forbidden=False`, `top1_doc_matches=True`.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ chuyển custom expectation suite sang pydantic hoặc Great Expectations để validate schema chặt hơn, đặc biệt là datetime, allowed doc ids, và canonical source versioning. Tôi cũng sẽ thêm test tự động cho từng quarantine reason để tránh rule mới làm rơi mất dữ liệu hợp lệ.
