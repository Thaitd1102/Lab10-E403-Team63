# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đoàn Thư Ánh
**Vai trò:** Cleaning & Quality Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài báo cáo:** ~500 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**
- `transform/cleaning_rules.py` lines 86–144 — Mở rộng thêm 3 rule mới: missing_exported_at, exported_date_in_future, chunk_length_limit.
- `quality/expectations.py` lines 115–138 — Thêm 2 expectation kiểm soát chất lượng (E7: no_missing_exported_at, E8: chunk_max_length_2000) cùng logic halt/warn.
- `artifacts/quarantine/quarantine_sprint2-final-check.csv` — File lưu vết dữ liệu rác/lỗi (quarantine records) kèm chi tiết reason.

**Quyết định nhị phân nào:**
- Expectation E7 vs E8: E7 (missing `exported_at`) được gán cờ **halt pipeline** do không thể monitor SLA freshness. E8 (chunk dài >2000) chỉ gán cờ **warn** vì không làm sập pipeline, chỉ chạy chậm hơn/tốn cost.
- Xử lý Refund Window (baseline): Chấp nhận fix policy từ `14 ngày` xuống `7 ngày` theo mặc định (override bằng cờ `--no-refund-fix`), giải quyết triệt để vấn đề version xung đột trước khi vào Vector Store.

**Bằng chứng:**
- `cleaning_rules.py` line 123: `if not exported_at or exported_at.strip() == "": ...`
- `expectations.py` line 122: `ExpectationResult("no_missing_exported_at", ok7, "halt", ...)`

---

## 2. Một quyết định kỹ thuật

**Halt vs Warn tradeoff:**

Tôi chọn:
- **E7 (no_missing_exported_at) = halt**: Thiếu timestamp export nghĩa là tính toán SLA Freshness sẽ sập ở downstream. Data team không dám chắc model dùng dữ liệu đời nào. Halt là bắt buộc.
- **E8 (chunk_max_length_2000) = warn**: Oversized chunks làm chậm tốc độ embedding và tốn API tokens, nhưng không phá vỡ logic code. Warn để ghi log cho Embed Owner điều chỉnh lại thay vì down-time.

**Lý do:** Kéo ranh giới rõ ràng giữa **data corruption** (mất trường quan trọng) cần ngắt ngay và **data suboptimal** (tốn cost) có thể điều tra sau.

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** Ban đầu không thể chắc chắn 3 rule mới có hoạt động không do dữ liệu mẫu quá "sạch", return 0 quarantine/impact.

**Phát hiện:** Khi sử dụng script để chèn 3 bad records rác vào file nguyên gốc `policy_export_dirty.csv` (1 dòng khuyết 'exported_at', 1 dòng ngày tương lai, và 1 dòng dài vượt mức), các rule/expectation mới được thử lửa. Ban đầu dòng lỗi bị lọt qua phần string do chưa cover khoảng trắng.

**Fix:** Clean kỹ biến truyền vào bằng `.strip() == ""` khi khai báo Rule #7 `missing_exported_at` ở dòng 124 trong `cleaning_rules.py`.

**Result:** Ở lần test thứ 2 (`run-id sprint2-final-check`), module bắt trọn 3 anomalies được chèn vào:
- Record id 12 rơi vào quarantine vì `missing_exported_at`.
- Record id 13 rơi vào vì `exported_date_in_future`.
- Record id 14 rơi vào vì `chunk_length_limit_exceeded`.

---

## 4. Bằng chứng trước / sau

**Trước (khi chưa inject proof):**
```
raw_records=10
cleaned_records=6 (quarantine 4 baselines)
expectations=8/8 PASS
```
Toàn bộ rule 7,8,9 không cho thấy chỉ báo hoạt động (impact = 0) do source ban đầu không có lỗi.

**Sau (khi chạy sprint2-final-check với data inject bad):**
```
raw_records=13
cleaned_records=6 (sau clean)
quarantine_records=7 (Rule cũ bắt 4, Rule 7 bắt 1, Rule 8 bắt 1, Rule 9 bắt 1)
```
Chứng tỏ Cleaning Rules hoạt động trơn tru. Dữ liệu rác chặn hoàn toàn và cách ly vào file `artifacts/quarantine/quarantine_sprint2-final-check.csv` chứa evidence string đủ 3 lý do.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ:
Tôi sẽ liên kết phần Reporting tới Alerting System. Thay vì sinh cảnh báo ra console và file csv tĩnh, tôi sẽ thêm 1 webhook đọc list `ExpectationResult` bị fail (có severity báo halt) và Slack ngay một custom message kèm theo `run_id` cho data engineers để họ can thiệp kịp thời mà không cần ssh vào server đọc log.
