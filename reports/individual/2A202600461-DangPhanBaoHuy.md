# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đặng Phan Bảo Huy (Embed & Idempotency Owner)  
**Vai trò:** Embed & Idempotency Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài báo cáo:** 545 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**
- `etl_pipeline.py` lines 101–144 — `cmd_embed_internal()` containing both upsert + prune logic
  - Line 114: `collection.upsert(ids=ids, documents=documents, embeddings=embeddings)`
  - Line 107: `stale_ids = set(collection.get()['ids']) - set(cleaned_ids)`
  - Line 108: `collection.delete(ids=list(stale_ids))`
- `chroma_db/` — persistent PersistentClient pointing to `day10_kb` collection
- `artifacts/manifest/manifest_sprint2-expanded.json` — payload docs count

**Kết nối với thành viên khác:**
- Nhận 6 cleaned rows từ Cleaning Owner
- Truyền `chroma_collection=day10_kb` + `chunk_count=6` cho Monitoring Owner
- Ensure `chunk_id` stable để Cleaning Owner trace được

**Bằng chứng:**
- `etl_pipeline.py` line 182: `client = chromadb.PersistentClient(path="chromadb")`
- `etl_pipeline.py` line 195: `collection.upsert(ids=ids, documents=docs, embeddings=embeddings)`
- Manifest: `"collection": "day10_kb", "doc_count": 6`

---

## 2. Một quyết định kỹ thuật

**Idempotent upsert + prune strategy:**

Tôi chọn:
1. **Upsert by chunk_id** — nếu same chunk_id seen trước, update embedding (không duplicate)
2. **Prune after publish** — xoá tất cả IDs không trong latest cleaned set (xóa stale vectors)

**Chi tiết:**
```python
# Upsert: nếu chunk_id='policy_refund_v4_c0' exists, thay thế
collection.upsert(ids=['policy_refund_v4_c0'], documents=['...'], embeddings=[...])

# Prune: sau cleaned, lấy all_ids_in_chroma, remove những ko trong cleaned_ids
stale_ids = set(collection.get()['ids']) - set(cleaned_ids)
collection.delete(ids=list(stale_ids))
```

**Lý do:** Nếu không prune, top-k search sẽ mix old embedding ("14 ngày refund") + new ("7 ngày refund") → confusing agent output. Prune ensures **single source of truth**.

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** Sau Sprint 2 run, `grading_run.py` query q_refund_window trả lại mixed results (khi chạy lại Sprint 1 dataset).

**Phát hiện:** Collection vẫn có 10 chunks (6 mới + 4 stale từ run trước). `top_k=5` trả về `policy_refund_v4` (mới, 7d) + `policy_refund_v3` (cũ, 14d). Agent bối rối.

**Fix:** Thêm prune step sau `upsert()`. Chạy `python etl_pipeline.py run --run-id sprint2-expanded` → xoá tất cả non-latest chunk_ids.

**Result:** Collection stable ở 6 chunks; `q_refund_window` query chỉ thấy 7d version.

---

## 4. Bằng chứng trước / sau

**Before (no prune):**
```json
{
  "run_id": "sprint1-test",
  "collection": "day10_kb",
  "doc_count": 6,
  "chroma_collection_size": 10  // 6 cleaned + 4 stale từ manual embed prior
}
```

**After (with prune):**
```json
{
  "run_id": "sprint2-expanded",
  "collection": "day10_kb",
  "doc_count": 6,
  "chroma_collection_size": 6  // cleaned set = collection size
}
```

Verify: `collection.get()['ids']` line count stable = 6

---

## 5. Cải tiến tiếp theo

Nếu có 2h thêm:

- **Embedding versioning** — tag mỗi chunk với `{chunk_id, embedding_model_version}` để track model upgrades
- **Batch optimize** — streaming embedding thay vì load toàn bộ 6 chunks cùng lúc
- **Collection checksum** — hash tất cả cleaned_ids, compare với prior; alert nếu unexpected change
