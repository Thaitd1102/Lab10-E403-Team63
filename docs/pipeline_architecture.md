# Kiến trúc pipeline — Lab Day 10

**Nhóm:** Lab Day 10 — Data Pipeline & Observability  
**Cập nhật:** 2026-04-15

---

## 1. Sơ đồ luồng

```
Raw Export (CSV)
     ↓
[Ingest] → load_raw_csv() → raw_records=10
     ↓
[Transform/Clean] → clean_rows() + apply rules #1–9
     ├→ Allowlist & schema validation
     ├→ Date normalization (DD/MM/YYYY → YYYY-MM-DD ISO)
     ├→ Stale data removal (HR version conflict, old effective_date)
     ├→ Deduplication
     ├→ Critical bug fix: refund window (14d → 7d)
     ├→ Chunk length limits
     └→ Freshness tracking (exported_at)
     ↓
     ├→ cleaned_records=6 (good records)
     └→ quarantine_records=4 (failed rule checks)
     ↓
[Quality Gate] → run_expectations() → 8 expectations (6 baseline + 2 Sprint 2)
     ├→ E1–E6 Baseline: min_rows, no_empty_doc_id, refund_no_stale_14d, ISO_date, HR_versions
     ├→ E7 Sprint2: no_missing_exported_at (freshness SLA)
     ├→ E8 Sprint2: chunk_max_length_2000 (model token limit)
     └→ Result: 8/8 PASS → proceed to embed
     ↓
[Embed] → Chroma PersistentClient (day10_kb collection)
     ├→ Upsert strategy: chunk_id as key (idempotent)
     ├→ Prune old ids after publish (no stale vectors in top-k)
     ├→ Model: all-MiniLM-L6-v2 (SentenceTransformers)
     └→ embed_upsert count=6
     ↓
[Publish] → Manifest (run_id, metrics, latest_exported_at)
     ├→ manifest_{run_id}.json records all metadata
     └→ Ready for Day 08/09 agent retrieval
     ↓
[Freshness Monitor] → check_manifest_freshness()
     └→ SLA: 24 hours (configurable)
```

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Owner vai trò |
|------------|-------|--------|---|
| **Ingest** | `data/raw/policy_export_dirty.csv` | raw_records=10, log | Ingestion Owner |
| **Transform** | raw rows + ALLOWED_DOC_IDS, rules config | cleaned_records=6, quarantine_records=4 | Cleaning & Quality Owner |
| **Quality** | cleaned_rows | expectation results (8), halt flag | Cleaning & Quality Owner |
| **Embed** | cleaned CSV | Chroma collection, embed_prune_removed count | Embed Owner |
| **Publish** | embed result | manifest JSON (run_id, metrics) | Embed Owner |
| **Monitor** | manifest JSON | freshness_check result (PASS/WARN/FAIL) | Monitoring/Docs Owner |

---

## 3. Idempotency & rerun

**Upsert strategy (chunk_id):**
- All cleaned rows have stable `chunk_id` = hash(doc_id + chunk_text + seq)
- Rerun 1: upsert → creates/updates 6 vectors
- Rerun 2: upsert same ids → no duplicates, collection size stable at 6
- Idempotent confirmed: sha256-based chunk_id

**Vector pruning (embed prune):**
- After publish, old id not in latest cleaned → deleted from collection
- Prevents "stale chunk in top-k" -> agent returns outdated answer
- Log metric: `embed_prune_removed` (count of dropped vectors)

**Testing:** 
```bash
python etl_pipeline.py run --run-id test1
# collection size = 6
python etl_pipeline.py run --run-id test2  
# collection size = 6 (no dupes, idempotent)
```

---

## 4. Liên hệ Day 09

**Pipeline này cung cấp corpus cho multi-agent Day 09:**
- Corpus source: `data/docs/` + cleaned CSV from Day 10
- Collection name: `day10_kb` (separate from `day09_kb`)
- Benefit: Day 09 reads clean, deduplicated, version-controlled data
  - No stale HR policy (10d vs 12d conflict resolved)
  - No critical bug (refund 14d → 7d fixed)
  - Freshness tracked via manifest

**Integration test (mock):** 
```bash
# Day 10 pipeline
python etl_pipeline.py run --run-id final

# Day 09 agent can query collection
from chromadb import PersistentClient
client = PersistentClient(path="./chroma_db")
col = client.get_collection("day10_kb")  # Same embedding model
results = col.query(query_texts=["7 ngày hoàn tiền"], n_results=3)
# Top-1 should be policy_refund_v4 (correct version)
```

---

## 5. Rủi ro đã biết

1. **Vector staleness**: If embed not pruned after cleaning rule change → top-k may return old chunks → agent mislead
   - Mitigation: Always run prune step; log `embed_prune_removed`

2. **Freshness SLA breach**: If raw CSV export is older than SLA (24h) → FAIL
   - Seen in test: latest_exported_at=2026-04-10T08:00:00, check time=2026-04-15 → FAIL (ok, expected)
   - Mitigation: Upstream should ingest fresh export, or adjust SLA in .env

3. **Expectation halt → pipeline blocks**: If critical expectation fails (e.g., refund 14d still in data), pipeline exits code 2
   - Mitigation: Fix root cause in rules; use `--skip-validate` only for analysis (Sprint 3 demo)

4. **Duplicate chunk_text**: Dedup by normalized text (lowercase, trim) → rare false positive if two policy docs have identical text
   - Mitigation: Manual review quarantine CSV; add metadata (source, version) to chunk_id if needed

5. **Chunk length limit (2000 chars)**: Long policy clauses may be truncated
   - Trade-off: Token cost vs coverage
   - Mitigation: Evaluate `chunk_max_length_2000` warning before production

