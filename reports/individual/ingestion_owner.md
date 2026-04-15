# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Member 1 (Ingestion Owner)  
**Vai trò:** Ingestion / Raw Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài báo cáo:** 520 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**
- `data/raw/policy_export_dirty.csv` — ingest source
- `etl_pipeline.py` lines 44–75 — `cmd_run()` load_raw_csv logic
- `transform/cleaning_rules.py` lines 20–50 — `load_raw_csv()` và `_norm_text()`
- `artifacts/logs/run_sprint2-expanded.log` — tất cả logs

**Kết nối với thành viên khác:**
- Đưa `raw_records=10` cho Cleaning Owner để clean
- Ghi ra `run_id=sprint2-expanded` cho Embed Owner trace
- Log tất cả metrics cho Monitoring Owner check freshness

**Bằng chứng:**
- Commit history: `transform/cleaning_rules.py` định nghĩa `load_raw_csv()` 
- Log artifact: `run_sprint2-expanded.log` ghi `raw_records=10` line 1

---

## 2. Một quyết định kỹ thuật

Tôi quyết định để **raw CSV tự định dạng** (không chuẩn hoá sớm) vì:

1. **Separation of concerns**: Ingest chỉ load → log, không xử lý
2. **Debug dễ**: Xem raw dữ liệu original vào quarantine log sau cleaning
3. **Audit trail**: Giữ exact format export từ source để trace lỗi upstream

Chi tiết: `load_raw_csv()` chỉ strip whitespace (`(v or "").strip()`), không normalize ngày hay remove BOM. Tất cả logic cleaning để `clean_rows()` xử lý ở tầng tiếp.

---

## 3. Một lỗi đã xử lý

**Triệu chứng:** Lần chạy đầu `etl_pipeline.py run` bị quit với `FileNotFoundError: policy_export_dirty.csv`.

**Phát hiện:** Log show `raw_path="data/raw/policy_export_dirty.csv"` nhưng Windows path separator `\` không match. 

**Fix:** Dùng `Path(__file__).resolve().parent / "data" / "raw"` để cross-platform path resolution (PureWindowsPath tự xử lý).

**Result:** Chạy lại → `raw_records=10` ✅

---

## 4. Bằng chứng before/after

**Before (lỗi path):**
```
ERROR: raw file not found: E:\assignments\...\policy_export_dirty.csv
Exit code: 1
```

**After (fix):**
```log
run_id=sprint2-expanded
raw_records=10
cleaned_records=6
quarantine_records=4
```

Run artifact: `artifacts/logs/run_sprint2-expanded.log` (full trace)

---

## 5. Cải tiến tiếp theo

Nếu có 2h thêm:

- **Add schema validation** trước load (kiểm `chunk_id, doc_id, chunk_text` column names)
- **Streaming ingest** cho CSV lớn (thay vì load toàn bộ vào memory)
- **Multi-source ingestion** — support API fetch + S3 download (không chỉ local file)
