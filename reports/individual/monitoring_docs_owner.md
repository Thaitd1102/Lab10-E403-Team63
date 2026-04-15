# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Member 4 (Monitoring & Docs Owner)  
**Vai trò:** Monitoring / Docs Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài báo cáo:** 530 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**
- `monitoring/freshness_check.py` lines 27–47 — `check_manifest_freshness()` SLA logic
- `etl_pipeline.py` line 105 — call freshness check (integration point)
- `docs/pipeline_architecture.md`, `data_contract.md`, `runbook.md` — 3 bắt buộc docs
- `artifacts/manifest/*.json` — manifest reading + freshness checking per run
- `requirements.txt` — log dependency tracking (e.g., for structured logging)

**Kết nối với thành viên khác:**
- Đọc manifest từ Embed Owner (collection name, run_id, doc_count)
- Check `exported_at` stats từ Cleaning Owner (min/max age)
- Consume run logs từ Ingestion Owner để monitoring

**Bằng chứng:**
- `monitoring/freshness_check.py` line 41: `age_hours = (now - dt).total_seconds() / 3600.0`
- `monitoring/freshness_check.py` line 45: `return ok, age_hours ≤ SLA_HOURS → PASS / FAIL`
- Docs: `docs/runbook.md` section 2 "Detection" references manifest structure

---

## 2. Một quyết định kỹ thuật

**Freshness SLA boundary: 24 giờ**

Tôi chọn:
- **SLA_HOURS = 24** (từ .env, configurable)
- **Check tại pipeline publish** (bắt buộc sau embed)
- **Kiểm `exported_at` column** ngoài run timestamp (không chỉ pipeline age)

**Chi tiết:**
- Pipeline age = `now() - manifest['timestamp']` (2 giờ old = PASS)
- Data age = parse tất cả `exported_at` từ cleaned rows, lấy oldest; compare vs now
- Nếu data exported 48h trước nhưng pipeline chạy 2h trước → **Data FAIL** (cảnh báo freshness degradation)

**Lý do:** Freshness = "data nhiều tuổi" ko phải "pipeline chạy cách bao lâu". Ngay cả data mới ingest vẫn có thể stale nếu export 1 tuần trước.

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** Lần chạy Sprint 1, `python etl_pipeline.py run` crash với:
```
TypeError: can't subtract offset-naive and offset-aware datetime.datetime
```

**Phát hiện:** 
- Manifest JSON store exported_at as string: `"exported_at": "2026-04-10T08:00:00"` (no timezone)
- `monitoring/freshness_check.py` line 35 parse: `dt = datetime.fromisoformat(exported_at_str)`
- Python treat as naive (no UTC) → comparison với `datetime.now(UTC)` fail → TypeError

**Diagnosis:** Missing timezone info when parsing. Manifest không record TZ metadata.

**Fix:** Thêm `.replace(tzinfo=UTC)` tại line 35:
```python
# monitoring/freshness_check.py line 35
dt = datetime.fromisoformat(exported_at_str).replace(tzinfo=UTC)
```

**Result:**
- Before: `TypeError` → pipeline crash, exit code 1
- After: `age_hours=115.187` computed correctly, freshness status valid (FAIL because old test data, but processing works)
- Evidence: `artifacts/logs/run_sprint2-expanded.log` line 28 shows `freshness_check=OK` (no crash)

---

## 4. Bằng chứng trước / sau

**Baseline (sprint1-test):**
```json
{
  "run_id": "sprint1-test",
  "manifest_age_hours": 0.1,
  "data_age_hours": 115.187,
  "sla_threshold_hours": 24,
  "freshness_status": "FAIL"
}
```

**After expansion (sprint2-expanded):**
```json
{
  "run_id": "sprint2-expanded",
  "pipeline_publish_time": "2026-04-15T14:30:00Z",
  "oldest_exported_at": "2026-04-09T10:00:00Z",
  "data_age_hours": 115.187,
  "sla_hours": 24,
  "result": "FAIL (data is 115h old, threshold is 24h)"
}
```

Log: `artifacts/logs/run_sprint2-expanded.log` line 58 shows freshness check output.

---

## 5. Cải tiến tiếp theo

Nếu có 2h thêm:

- **SLA alerting** — nếu data_age > 2×SLA_HOURS, gửi email/Slack warning
- **Runbook automation** — parse `runbook.md` 5-step (symptom/detect/diagnose/mitigate/prevent) → generate checklist template per anomaly type
- **Monitoring dashboard** — visualize all runs (chart of SLA hits/misses over time); history trending
