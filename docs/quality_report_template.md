# Quality report — Lab Day 10 (nhóm)

**run_id:** sprint2-expanded (baseline clean) & sprint3-inject-bad (corrupted)  
**Ngày:** 2026-04-15
=======


---

## 1. Tóm tắt số liệu


| Chỉ số | Sprint 2 (baseline) | Sprint 3 (corrupted)| Ghi chú |
|--------|---|---|---|
| raw_records | 10 | 10 | Cùng source CSV |
| cleaned_records | 6 | 6 | Số lượng tương tự (rules không khác) |
| quarantine_records | 4 | 4 | Same failure reasons |
| Expectation halt? | NO ✓ | YES ❌ | refund_no_stale_14d_window FAIL |
| expectations_ok | 8/8 PASS | 7/8 PASS | E3 failure in Sprint 3 |
=======


---

## 2. Before / after retrieval (bắt buộc)


**Dataset source:** `data/test_questions.json` (4 golden queries)  
**Evaluator:** `eval_retrieval.py` (keyword-based top-k matching)

### 2a. Critical metric: q_refund_window (MUST FIX)

**Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền?**

**Sprint 2 Baseline (with refund fix):**
```csv
question_id,contains_expected,hits_forbidden,top1_doc_id
q_refund_window,yes,no,policy_refund_v4
```
✅ PASS: Contains "7 ngày", avoids "14 ngày"

**Sprint 3 Corrupted (--no-refund-fix):**
```csv
question_id,contains_expected,hits_forbidden,top1_doc_id
q_refund_window,yes,yes,policy_refund_v4
```
❌ FAIL: Contains "7 ngày" but ALSO returns "14 ngày" chunk (hits_forbidden=yes)

**Impact:** Agent would give WRONG answer if top-k includes stale refund chunk  
**Root cause:** Expectation E3 failed → "14 ngày" not removed during clean  
**Fix applied:** `cleaning_rules.py` line ~95, `apply_refund_window_fix=True` (used in Sprint 2)

---

### 2b. Merit metric: q_leave_version (HR versioning)

**Theo chính sách nghỉ phép hiện hành (2026), nhân viên được bao nhiêu ngày?**

**Sprint 2 Baseline:**
```csv
question_id,contains_expected,top1_doc_expected
q_leave_version,yes,yes
```
✅ PASS: Returns "12 ngày" (2026 version), top1_doc=hr_leave_policy

**Sprint 3 Corrupted:**  
(Same result, since no --no-leave-version flag used)
```csv
question_id,contains_expected,top1_doc_expected
q_leave_version,yes,yes
```
✅ PASS: Baseline rule (stale HR <2026-01-01) still works
=======


---

## 3. Freshness & monitor

**manifest_sprint2-expanded.json:**
```json
{
  "latest_exported_at": "2026-04-10T08:00:00",
  "run_timestamp": "2026-04-15T03:06:44.088250+00:00",
  "sla_hours": 24.0
}
```

**freshness_check result:**
```
FAIL — age_hours=115.187 > 24.0
```

**Explanation:** Source CSV has very old export (2026-04-10); expected for lab demo (represents stale data scenario). In production, would trigger alarm at 24h threshold.

**Mitigation:** Upstream (HR DB) must export fresh policy within 24h of changes.
=======


---

## 4. Corruption inject (Sprint 3)

**Kịch bản:** `python etl_pipeline.py run --run-id sprint3-inject-bad --no-refund-fix --skip-validate`

**Cố ý làm sai gì:**
- `--no-refund-fix`: Skip the critical bug fix (14d → 7d)
- `--skip-validate`: Allow expectation halt to continue (for analysis)

**Cách phát hiện:**
- Expectation log: `expectation[refund_no_stale_14d_window] FAIL (violations=1)` ← E3 caught it!
- Eval metric: `q_refund_window hits_forbidden=yes` ← Retrieval eval shows regression
- Quarantine: No change (same records); difference is in cleaned data content

**Result:** All expectations in Sprint 2 → 8/8 PASS; in Sprint 3 → 7/8 FAIL (E3 halt blocked by --skip-validate)
=======


---

## 5. Hạn chế & việc chưa làm

1. **No LLM eval:** Baseline uses keyword matching only. Could add GPT-judge for semantic correctness (Distinction+).
2. **No streaming checks:** Freshness SLA only checked end-of-pipeline. Could add mid-pipeline watermark checks.
3. **No A/B test framework:** Only compare 2 snapshots. Could build continuous experiment tracking.
4. **Hardcoded cutoff dates:** HR version cutoff "2026-01-01" in code (Rule #3). Should read from `data_contract.yaml` for flexibility.
5. **No automated roll-forward:** After --skip-validate, would need manual deploy approval to publish corrupted collection.

=======

