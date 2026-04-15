# Data contract — Lab Day 10

> Kiến thức từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.
=======


---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |

|-------|-------|-----------|---|
| **HR Policy (Leave)** | CSV export từ HR module | 1. Duplicate rows<br>2. Old version (2025) vs 2026<br>3. Missing effective_date | Quarantine count, expectation E6 |
| **IT Helpdesk FAQ** | CSV extract + portal | 1. Date format (D/M/Y vs ISO)<br>2. Missing chunk_text | Quarantine + E5 ISO check |
| **Policy: Refund** | Core business table | **1. CRITICAL: 14d vs 7d bug**<br>2. Migration error<br>3. Duplicates | **E3 HALT if fail** |
| **SLA P1 Support** | SLA doc | 1. Unit typo<br>2. Version conflict | Generic E1 safeguard |
=======


---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |

|-----|------|------|---|
| **chunk_id** | string | ✓ | `{doc_id}_{seq}_{hash[:16]}` |
| **doc_id** | string | ✓ | Allowlist: policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy |
| **chunk_text** | text | ✓ | Cleaned; refund fix marked `[cleaned: stale_refund_window]` |
| **effective_date** | YYYY-MM-DD | ✓ | ISO 8601; HR <2026-01-01 → quarantine |
| **exported_at** | ISO datetime | ✓ | Freshness SLA tracking |
=======

---

## 3. Quy tắc quarantine vs drop


**Quarantine (lưu giữ):** Analyzable failures  
- Date format lạ, HR version cũ, unknown doc_id  
- File: `artifacts/quarantine/quarantine_{run_id}.csv`  
- Approval: Manual review + fix source if salvageable  

**Drop (xóa vĩnh viễn):** Unrecoverable  
- Empty chunk_text, duplicate text  
- Not written, counted in log only  

---

## 4. Source of truth & versioning

**Refund policy canonical:**
- File: `data/raw/policy_export_dirty.csv` (lab) / HR DB (prod)
- Bug fix: "14 ngày" → "7 ngày" (applied in cleaning_rules.py line ~95)
- Marker: `[cleaned: stale_refund_window]` appended for traceability

**HR leave versioning:**
- 2025: "10 ngày" (quarantine if found)  
- 2026: "12 ngày" (canonical; cutoff: 2026-01-01)  

**All chunks tagged with run_id for rollback capability.**
=======
