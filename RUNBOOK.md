# RUNBOOK — Nadex-results (LIVEWELL)

**Scope:** Manual/adhoc runs that extract & normalize Nadex Daily Results PDFs and write cleaned CSVs + logs to S3.  
**Owner/Role:** PIC / Operator  
**Configs:** `configs/s3.yaml` (buckets/prefixes), `configs/ticker_from_description.csv` (mapping)  
**Artifacts (S3):** `historical/`, `manifests/processed_files.json`, `logs/run_log.csv`

---

## 1) Preconditions (T-10 min)
- ✅ AWS credentials configured (SSO/role) and S3 access verified.
- ✅ `configs/s3.yaml` present and correct (bucket, region, prefixes: `historical`, `manifests`, `logs`).
- ✅ Notebook: `notebooks/nadex-historical.ipynb` opens and runs.
- ✅ No **hard-coded** buckets in notebook (all from config).
- ✅ Mapping file reachable: `configs/ticker_from_description.csv`.

> Quick sanity:
> - Open `configs/s3.yaml` and visually confirm values.
> - In the notebook, run the config loader cell and print `BUCKET`, `RUNLOG_KEY`.

---

## 2) Daily Run (T-0)
1. **Select window:** set `start` / `end` dates in the final run cell (use a bounded window; historical re-runs are ok—manifest skips duplicates).
2. **Run pipeline:** execute the notebook’s main cell (`run_nadex_pipeline(...)`).
3. **Write run log:** ensure the final cell calls `append_runlog_s3(...)` (with `run_id` timestamp and counters).

**Expected outputs**
- Clean CSVs uploaded under **`historical/`** (date-partitioned by your logic).
- **`manifests/processed_files.json`** updated.
- **`logs/run_log.csv`** appended with a new row:
  ```
  date,start_time,end_time,status,files_processed,files_skipped,files_error,run_id,notes
  ```

**Post-run verification (T+10)**
- S3 lists show new/updated keys in `historical/` for your window.
- `logs/run_log.csv` contains today’s row with non-zero counters (unless truly no new files).
- Spot-check 1 CSV (headers, row count, Ticker mapping, no obviously bad values).

---

## 3) Abort Criteria (stop & triage)
Stop the run and file a note if any of the following occur:
- **Zero tables extracted** from expected PDFs for the window.
- **S3 failures** (permission denied, missing bucket, write error).
- **Run log not updated** or counters look wrong (e.g., processed=0 but you expected data).
- **Parsing anomalies** (columns missing, all values NA, mapping yields `"UNKNOWN"` tickers at high rate).

**Immediate actions**
- Capture the error message and the **`run_id`**.
- Do **not** re-run over the same window until fixed (prevents noisy logs).
- Note suspected root cause and next step in `notes` for the next attempt.

---

## 4) Return to Service
1. Fix root cause (config, mapping rule, small code patch).
2. Re-run the **same** window to confirm behavior (small range).
3. Verify new entries in `historical/` and a correct `run_log.csv` row.
4. Resume normal runs.

---

## 5) Operational Notes
- **Idempotency:** Already-processed files are skipped via `manifests/processed_files.json`.
- **Mapping:** Update `configs/ticker_from_description.csv` to improve coverage (lower `priority` wins).
- **No data in repo:** Do not commit CSV outputs; everything belongs in S3.
- **Provenance:** `run_id` is a timestamp, e.g., `20250915T174200`.

**Example run log row**
```
2025-09-15,2025-09-15T17:42:00,2025-09-15T17:43:20,success,18,4,0,20250915T174200,Historical batch run
```

---

## 6) Change History
- 2025-09-15: v0.1 — initial minimal runbook (manual run, S3 artifacts, manifest, run log).
