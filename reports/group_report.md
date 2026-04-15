# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability


=======
**Tên nhóm:** 63  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Thái | Ingestion / Raw Owner | 26ai.thaitd@vinuni.edu.vn |
| Ánh | Cleaning & Quality Owner | 26ai.anhdt4@vinuni.edu.vn |
| Huy | Embed & Idempotency Owner | 26ai.huydpb@vinuni.edu.vn |
| Quân | Monitoring / Docs Owner | 26ai.quanmv@vinuni.edu.vn |

=======

**Ngày nộp:** 2026-04-15  
**Repo:** https://github.com/Thaitd1102/Lab10-E403-Team63  
**Độ dài báo cáo:** 850 từ

---

## 1. Pipeline tổng quan (150–200 từ)

### Nguồn dữ liệu:
- **CSV mẫu:** `data/raw/policy_export_dirty.csv` (10 records)
  - Simulates HR/Policy export từ legacy systems
  - Contains typical issues: duplicates, date format inconsistency, version conflicts, stale chunks

### Luồng xử lý chuẩn:
```bash
python etl_pipeline.py run --run-id sprint2-expanded
```

**Kết quả thực tế (Sprint 2):**
- **raw_records** = 10 (từ CSV)
- **cleaned_records** = 6 (sau 9 cleaning rules)
- **quarantine_records** = 4 (failures: unknown_doc_id, stale_hr_date, duplicate_text, missing_effective_date)
- **expectations_ok** = 8/8 PASS (6 baseline + 2 Sprint 2 mới)
- **embed_upsert** = 6 chunks vào Chroma collection `day10_kb`
- **freshness_check** = FAIL (expected: data 5 ngày cũ > SLA 24h)
- **Exit code** = 0 (pipeline OK)

**Lệnh chạy một dòng:**  
```bash
cd lab && python etl_pipeline.py run && python eval_retrieval.py --out artifacts/eval/eval.csv
```

---

## 2. Cleaning & expectation (150–200 từ)

### Baseline (received):
Rules 1–6: allowlist, date normalize, HR stale check, dedupe, missing_text, critical refund bug fix

### Sprint 2 mở rộng:
**3 rule mới:**
1. **Rule #7 (missing_exported_at)**: Exported-at không được rỗng — ensure freshness SLA can be tracked. **Impact:** 0 quarantine (all records have exported_at in mẫu)
2. **Rule #8 (exported_date_in_future)**: Exported-at không vượt quá hôm nay (logic sanity). **Impact:** 0 quarantine (mẫu dates reasonable)
3. **Rule #9 (chunk_length_limit)**: Chunk không vượt 2000 ký tự (embedding cost). **Impact:** 0 quarantine (mẫu chunks all <2000 chars)

**2 expectation mới:**
- **E7 (no_missing_exported_at, halt)**: Tất cả cleaned rows phải có exported_at. **Result:** 0 violations, PASS
- **E8 (chunk_max_length_2000, warn)**: Phát hiện chunks > 2000 chars. **Result:** 0 oversized, PASS

### Bảng metric_impact (chứng minh rule/expectation hoạt động):

| Rule / Expectation mới | Trước (baseline) | Sau (mở rộng) | Chứng cứ |
|---|---|---|---|
| Rule#7 missing_exported_at | N/A | 0 quarantine (all have exported_at) | artifacts/quarantine/quarantine_sprint2.csv → no "missing_exported_at" reason |
| Rule#8 exported_date_in_future | N/A | 0 quarantine (all dates ≤ 2026-04-15) | Log: no entries with "exported_date_in_future" |
| Rule#9 chunk_length_limit | N/A | 0 quarantine (max chunk=~180 chars) | artifacts/cleaned shows longest chunk <200 |
| E7 no_missing_exported_at (halt) | N/A | PASS (0 violations) | expectation log: rows_missing_exported_at=0 |
| E8 chunk_max_length_2000 (warn) | N/A | PASS (0 violations) | expectation log: oversized_chunks=0 |

---

## 3. Before / after ảnh hưởng retrieval (200–250 từ)

### Kịch bản inject (Sprint 3):
```bash
python etl_pipeline.py run --run-id sprint3-inject-bad --no-refund-fix --skip-validate
```

Cố ý bỏ qua `apply_refund_window_fix=True` → cleaned CSV sẽ chứa "14 ngày làm việc" thay vì "7 ngày"

**Expectation fail observed:**
```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1
```

### Kết quả định lượng (eval before/after):

**Sprint 2 Baseline (with fix):**
```csv
q_refund_window,yes,no    ← GOOD: has "7 ngày", avoids "14 ngày"
q_p1_sla,yes,no           ← GOOD
q_lockout,yes,no          ← GOOD
q_leave_version,yes,no    ← GOOD: "12 ngày 2026", top1_expected=yes
```
**Score: 4/4 pass** ✅

**Sprint 3 Corrupted (no-refund-fix):**
```csv
q_refund_window,yes,yes   ← BAD: hits forbidden "14 ngày" chunk!
q_p1_sla,yes,no           ← same
q_lockout,yes,no          ← same
q_leave_version,yes,no    ← same
```
**Score: 3/4 pass** ❌ (q_refund_window fails)

**Critical finding:** Query still returns correct answer in top results (contains_expected=yes) BUT also includes stale chunk (hits_forbidden=yes). Agent would give MIXED signal → user confusion.

**Evidence files:**
- `artifacts/eval/eval_sprint2_baseline.csv`
- `artifacts/eval/eval_sprint3_corrupted.csv`

---

## 4. Freshness & monitoring (100–150 từ)

### SLA lựa chọn:
- **FRESHNESS_SLA_HOURS = 24** (from .env.example)
- Meaning: Data export không được cũ hơn 24 giờ

### Manifest check (Sprint 2):
```json
{
  "latest_exported_at": "2026-04-10T08:00:00",
  "run_timestamp": "2026-04-15T03:06:44+00:00",
  "sla_hours": 24.0,
  "freshness_check": "FAIL",
  "age_hours": 115.112
}
```

**Kết luận:** 
- Data age = 115 hours > 24h SLA → **FAIL** (expected for demo using old mẫu)
- In production: Would need fresh export from HR/Policy DB
- Mitigation: Either adjust SLA window or trigger fresh export upstream

---

## 5. Liên hệ Day 09 (50–100 từ)

**Data sau embed** có phục vụ lại multi-agent Day 09?

**ANSWER: YES, tách collection**

- Day 09 agent initially used `day09_kb` collection
- Day 10 pipeline builds `day10_kb` (same documents, but **cleaned + deduplicated + versioned**)
- Benefit: Agent now queries clean data
  - No stale HR policy (10d vs 12d resolved)
  - No critical refund bug (14d → 7d fixed)
  - All chunks traced to run_id for audit

**Integration ready:** Agent can switch to `day10_kb` collection after deploying this pipeline.

---

## 6. Rủi ro còn lại & việc chưa làm

1. **Hardcoded thresholds:** HR cutoff date "2026-01-01", chunk length 2000 → Should move to `data_contract.yaml` config
2. **Manual quarantine approval:** Quarantine CSV ghi log nhưng không có workflow tự động → Need JIRA integration
3. **No LLM eval:** Chỉ keyword matching → Could add GPT semantic judge (Distinction tier)
4. **Batch freshness only:** Check ở cuối pipeline → Could add streaming watermark checks
5. **No rollback strategy:** If bad embed published → Manual collection rebuild (chậm) → Need versioned Chroma snapshots

