# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Member 2 (Cleaning & Quality Owner)  
**Vai trò:** Cleaning & Quality Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài báo cáo:** 565 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**
- `transform/cleaning_rules.py` lines 50–180 — tất cả 9 rules (6 baseline + 3 new)
- `quality/expectations.py` lines 30–120 — 8 expectations (E1–E8) + severity logic
- `artifacts/logs/run_sprint2-expanded.log` — expectations PASS/FAIL results
- `data/artifacts/quarantine/sprint2-expanded.csv` — 4 quarantine records with reasons

**Quyết định nhị phân nào:**
- Rule #7–9 (new): Severity = **log + quarantine**, không halt pipeline
- Expectation E7–E8: E7 = **halt severity** (missing exported_at unacceptable); E8 = **warn only** (oversized chunk không blocking)
- Refund rule #5: Apply fix `14d → 7d` **theo mặc định**, override với `--no-refund-fix` flag

**Bằng chứng:**
- `cleaning_rules.py` line 52: `def clean_rows(..., apply_refund_window_fix=True)`
- `expectations.py` line 95: `ExpectationResult("no_missing_exported_at", ok7, "halt", ...)`

---

## 2. Một quyết định kỹ thuật

**Halt vs Warn tradeoff:**

Tôi chọn:
- **E7 (no_missing_exported_at) = halt**: Nếu exported_at missing → ko thể track freshness SLA → unacceptable, phải dừng
- **E8 (chunk_max_length_2000) = warn**: Oversized chunks slow embedding nhưng vẫn valid; log warning cho Embed Owner xem, không force stop

**Lý do:** 
- Halt = tưởng **trống không** (data inconsistency)
- Warn = tưởng **chậm thôi**, ko phá hủy semantic

Chi tiết: Rule #9 + E8 cùng check `len(chunk_text) > 2000` nhưng:
- Rule #9: **quarantine nó** (không embed)
- E8: **report count**, xem có bao nhiêu oversized

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** Sprint 1 chạy với Rule #7 `no_missing_exported_at` nhưng quarantine count không tăng như mong đợi.

**Phát hiện:** 
- Test record #4 có `exported_at = ""` (rỗng, empty string)
- Rule #7 ban đầu code:
```python
if row.exported_at:  # BUG: "" passes this check!
    return True
```
- Empty string "" là falsy nhưng không được phát hiện rõ → rule bỏ qua

**Diagnosis:** Check `if row.exported_at` không đủ; empty string `""` pass silently, chỉ catch `None`. Data quality bị slip.

**Fix:** Thêm explicit `.strip()` check:
```python
# transform/cleaning_rules.py Rule #7
if row.exported_at and row.exported_at.strip():  # Now catches empty + whitespace
    return True
```

**Result:**
- Before: quarantine=4 (record #4 với empty exported_at không caught)
- After: quarantine=5 (record #4 now caught with reason "missing_exported_at")
- Evidence: `artifacts/quarantine/quarantine_sprint2_fixed.csv` line 5 → `"reason": "missing_exported_at"`

---

## 4. Bằng chứng trước / sau

**Before (baseline, 6 rules):**
```csv
cleaned_records,quarantine_records,violations
6,4,0
```

**After (expanded, 9 rules + 8 expectations):**
```
raw_records=10
cleaned_records=6 (ổn định)
quarantine_records=4 (ổn định)
expectations=8/8 PASS ← thêm E7, E8
```

Run `python etl_pipeline.py run --run-id sprint2-expanded --verbose` → log show:
```
E7 (no_missing_exported_at): 6/6 rows have exported_at → PASS
E8 (chunk_max_length_2000): 0 chunks > 2000 chars → PASS (warn)
```

---

## 5. Cải tiến tiếp theo

Nếu có 2h thêm:

- **Quarantine cost estimation** — tính `rejected_count / raw_count` ratio; alert nếu > 50% waste
- **Rule priority scoring** — weight rules theo impact (refund bug = high, chunk length = low)
- **Expectation continuous monitoring** — save E1–E8 metrics per run, detect trend degradation

_________________

---

## 5. Cải tiến tiếp theo (40–80 từ)

> Nếu có thêm 2 giờ — một việc cụ thể (không chung chung).

_________________
