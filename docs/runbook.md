# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

**Agent trả lời** "Khách hàng có **14 ngày làm việc** để yêu cầu hoàn tiền" (sai)  
**Expected answer:** "Khách hàng có **7 ngày làm việc** để yêu cầu hoàn tiền" (đúng)

Or:  
**Agent retrieves** policy chunk mentioning "10 ngày phép năm" (old HR 2025)  
**Expected:** "12 ngày phép năm" (current 2026 policy)
=======


---

## Detection

**Metric 1 — Expectation fail log:**
```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1
```
This indicates cleaned data still contains "14 ngày làm việc" chunk for policy_refund_v4.

**Metric 2 — Eval metric degrade:**
```csv
question_id,contains_expected,hits_forbidden
q_refund_window,yes,yes  ← FAIL (should be hits_forbidden=no)
```
Top-k results contain "14 ngày" (forbidden keyword).

**Metric 3 — Freshness stale:**
```
freshness_check=FAIL ... reason=freshness_sla_exceeded (age_hours=115 > 24)
```
Source data export is older than SLA → ingest data may be outdated.
=======


---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |

|------|------|---|
| 1 | `cat artifacts/manifests/manifest_<run-id>.json` | latest_exported_at < 24h; no_refund_fix=false |
| 2 | `cat artifacts/quarantine/quarantine_<run-id>.csv \| grep refund` | Stale chunks should NOT be here (shouldn't be quarantined) |
| 3 | `cat artifacts/cleaned/cleaned_<run-id>.csv \| grep "14 ngày"` | Should return empty (fix applied) |
| 4 | `python eval_retrieval.py --out eval_debug.csv && grep q_refund eval_debug.csv` | Real retrieval: contains_expected=yes AND hits_forbidden=no |
=======
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | … |
| 2 | Mở `artifacts/quarantine/*.csv` | … |
| 3 | Chạy `python eval_retrieval.py` | … |


---

## Mitigation

**If expectation halt (refund rule fail):**

1. Check rule application:
   ```bash
   python etl_pipeline.py run --run-id debug_check
   # Log will show: "expectation[refund_no_stale_14d_window] FAIL"
   ```

2. Fix root cause:
   - If CSV still has "14 ngày": Fix source data export
   - If code didn't apply fix: Check `cleaning_rules.py` line ~95

3. Rerun with fix:
   ```bash
   python etl_pipeline.py run --run-id fixed_recovery
   # Expect: expectation[refund_no_stale_14d_window] OK
   ```

4. Validate via eval:
   ```bash
   python eval_retrieval.py --out eval_fixed.csv
   # q_refund_window should now have hits_forbidden=no
   ```

**If freshness stale:**
- Re-trigger source export (upstream task)
- Adjust FRESHNESS_SLA_HOURS in .env if changed
- Post banner: "Policy data last updated X hours ago"
=======


---

## Prevention

### Add to expectation suite:
- E3: refund_no_stale_14d_window (halt) ✓ present
- E7: no_missing_exported_at (halt) ✓ present  
- Add: exported_at freshness check (warn)

### Add to monitoring:
- Alert: freshness_check=FAIL for 2+ runs → page oncall
- Alert: expectation fail → blocker, halt deploy

### Add to escalation:
1. Expectation fail 1× → team fix in 2h
2. Expectation fail 2× → escalate to policy owner
3. Expectation fail 3× → block agent deployment

### Add to Day 09 handoff:
- Agent logs top-k chunk metadata (doc_id, run_id, effective_date)
- If agent.response ≠ expected → trace run_id for root cause audit
=======

