# ETL Pipeline — System Context

## Pipeline Overview

```
_raw_data/           ← raw CSVs (and/or safe XLSX sheets)
_clean_data/         ← outputs of gen_clean.ipynb
_merge_output/       ← outputs of gen_merge.ipynb (single merged CSV)
_analysis_output/    ← outputs of gen_analysis.ipynb (XLSX + any charts)
```

Each stage is a separate notebook. Run them in order. Each stage writes auditable
outputs before moving to the next. Never skip the human review pauses — that's the point.

---

## gen_clean.ipynb

### What it does
- Loads all CSVs from `_raw_data/` (and safely ingests simple XLSX sheets)
- Normalizes column names (lowercase, underscores, collision-safe)
- Detects and logs potential issues without automatically fixing them
- Gives you human-in-the-loop gates before writing anything

### Audit outputs (written to `_clean_data/_clean_audit.xlsx`)

| Sheet | What to look for |
|---|---|
| `clean_audit` | Top block: table shapes, column rename summary, candidate keys, dup signatures. Lower blocks: ingestion exceptions, dedup summary, delimited candidates, column map, dropped columns, column profiles |
| `clean_plan` | Steps applied and their status (applied / applied_if_needed / pending) |
| `data_dictionary` | Per-column profile with sample values. Check for unexpected nulls or dtypes. |
| `dedup_audit` | Exact row dup summary + key-based dup signatures per candidate key column |
| `dedup_plan` | What dedup steps are proposed and their status |
| `auto_audit` | Proof that automated steps were applied: row/col counts before vs. after |
| `dedup_apply_audit` | What dedup was actually applied (if anything) |

### Human decision points

**Stage A — after audit workbook is built:**
Review `clean_audit`. Key questions:
- Are column names sensible after normalization? (column_map block)
- Are there any delimited columns that should be split? (delimited_candidates block)
- Are there obvious duplicate rows or key violations? (dedup_summary, dedup_audit)

**Stage B1a — baseline auto-clean:**
Type `YES` to apply schema hygiene (name normalization, drop Unnamed columns).
This is safe — it only normalizes structure, it does not drop rows or change values.

**Stage B1b — dedup audit:**
Runs automatically after B1a. Writes `dedup_audit` and `dedup_plan` to the workbook.
Review before populating allowlists in the next cell.

**Post-clean allowlists cell:**
This is where you optionally apply:
- `split_allowlist` — to split a delimited column into parts (e.g. a semicolon-separated list)
- `exact_row_tables` — table names where exact duplicate rows should be dropped
- `key_rules` — key-based dedup with a keep strategy (`first`, `last`, `max_nonnull`)

If you don't need any of these, leave them as empty lists and proceed.

**Dataset-specific manual cleaning cell:**
This is your escape hatch. Add any transforms that the automated steps couldn't handle:
- Row filtering (e.g. remove rows where a column matches a bad-value pattern)
- Column renames (e.g. standardize a join key name across tables)
- Value replacements (e.g. fix known bad categorical values)
- Any explicit dedup not covered by the allowlists

Document what you changed and why. These are the decisions you'll need to explain.

**Stage B2 — final CSV export:**
Type `YES` to write cleaned CSVs to `_clean_data/`. Do not proceed until you're
satisfied with the audit and manual cleaning cells above.

---

## gen_merge.ipynb

### What it does
- Loads clean CSVs from `_clean_data/`
- Runs generic join reconnaissance across all table pairs
- Builds a conservative join plan with risk flags
- Executes the plan (left joins only) with row-explosion guardrails
- Produces an audited single merged CSV

### Audit outputs (written to `_merge_output/_merge_audit.xlsx`)

| Sheet | What to look for |
|---|---|
| `merge_audit` | Top block: column profiles, redundant pairs, overlap clusters, any post-join audit blocks. Main table: all candidate join pairs ranked by composite score with evidence metrics, reason flags, and cardinality labels |
| `merge_plan` | The proposed joins with risk assessment. Also shows `plan_recon_review` — a filtered recon view for only the planned joins |

### Reading the merge_audit table

Key columns to focus on:

- `composite_score` — overall ranking score (higher is better). Top candidates for each table pair appear first.
- `evidence_max` — max of unique_coverage and row_match_rate across raw/str modes. Values near 1.0 mean strong key overlap.
- `name_similarity` — token + sequence similarity of column names. Values above 0.70 are solid.
- `join_cardinality_label` — 1:1, M:1, 1:M, or M:M. For dimension joins you want M:1 (many fact rows to one dim row). M:M is a red flag.
- `inflation_risk_bucket` — low / medium / high. Based on sampled merge inflation. High means the join would multiply rows significantly.
- `right_key_quality_bucket` — ok or risky. Risky means the right-side join key is not unique enough for safe auto-join.
- `reason_flags` — semicolon-separated list of detected issues. Common ones: `right_join_key_not_unique`, `sample_merge_inflation_high`, `sentinel_date_like_column`, `surrogate_index_like_column`.

### Reading the merge_plan table

- `status: ok` — the plan algorithm is confident this join is safe to auto-execute.
- `status: flag` — something triggered a caution. Read `flag_reason` to understand why.
- `status: no_candidate` — no join pair was found for this table. You'll need to handle it manually.
- `explode_risk` / `coverage_risk` — low / medium / high. Explode = row multiplication risk. Coverage = expected null rate after left join.

### Human decision points

**After recon + plan cells:**
Review `_merge_audit.xlsx` before executing. Key questions:
- Does the plan pick the right join keys for each table pair?
- Are any flagged joins ones you actually want to force through?
- Are there `no_candidate` tables that need a manual join approach?

**Execute joins gate:**
Type `YES` to run the automated join plan. Only `status: ok` joins are executed by default.

**Auto audit cell:**
Runs immediately after execution. Check `auto_row_integrity` — if `delta_rows` is non-zero,
the join multiplied rows. Investigate before proceeding.

**Coverage check + force flagged cell:**
For any join the plan flagged, use `left_join_coverage()` to check actual key overlap.
- `match_rate` near 1.0 → safe to override
- Low `match_rate` → genuine key mismatch — do not override without understanding why

To override: uncomment the `plan.loc[...]` lines, enter the table names, and type `YES`.

**Dataset-specific derived fields cell:**
Add any columns that need to be computed from the joined data before export.
Duration columns, flag columns, composite fields. Document what you add.

**Final export gate:**
Type `YES` to write `_merged_final.csv` to `_merge_output/`.

---

## gen_analysis.ipynb

### What it does
- Loads the single merged CSV from `_merge_output/`
- Provides a column resolution pattern so schema changes only need one edit
- Provides a derived fields cell for pre-analysis computed columns
- Gives section scaffolding for baseline distributions, cross-variable relationships, and temporal trends
- Writes a data dictionary and summary tables to Excel

### This notebook is intentionally incomplete by design

Sections 1–3 are generic starters. Section 4 is a placeholder for dataset-specific analysis.
You will always need to edit this notebook for your specific questions.

### Column resolution cell

Edit this cell first. Define a resolved constant for each semantic role:

```python
ENTITY_ID_COL = _resolve(["customer_id", "student_id", "user_id"], df_raw)
DATE_COL      = _resolve(["date", "transaction_date", "dateid"], df_raw)
VALUE_COL     = _resolve(["amount", "spend", "revenue"], df_raw)
SEGMENT_COL   = _resolve(["store", "region", "category"], df_raw)
```

All downstream cells reference these constants. If the merged schema changes, update only here.

### Derived fields cell

Rename merge-produced columns to `derived_` prefix. Add computed fields with the same prefix.
Rename join key columns to `joinKeyCol_` prefix.

Document column conventions in this file so future sessions don't need to reverse-engineer them.

### Analysis sections

Each section should have a clear stakeholder question it answers. Before writing code, write the question as a comment. Frame outputs as observational unless the analysis explicitly supports a stronger claim.

### Outputs

- `_analysis_output/<base_table_name>__analyses.xlsx` — summary tables + data dictionary
- `_analysis_output/__*.png` — any charts produced (prefix with section number)

---

## Column Naming Conventions

These apply across all ETL projects using this pipeline:

- `derived_*` — columns computed during cleaning or analysis (not raw input)
- `joinKeyCol_*` — columns used as merge keys (renamed for clarity)

Document project-specific column decisions in this file under a "Project Notes" section below.

---

## Common Issues

| Symptom | Likely cause | Where to look |
|---|---|---|
| `delta_rows` non-zero after join | Right-side key is not unique | `merge_audit`: `right_key_quality_bucket = risky`, `join_cardinality_label = 1:M or M:M` |
| Many nulls in joined columns | Low key overlap / wrong join key | `merge_audit`: `evidence_max` low, `coverage_risk = high` |
| `no_candidate` in plan | No name-similar columns found above threshold | Lower `name_sim_min` in `merge_audit()` call, or handle manually |
| Unexpected column name collisions after join | Tables share non-key column names | `prefix` rename strategy handles this; check `projected_new_col_examples` in plan |
| Clean CSVs look wrong after export | Manual cleaning cell ran before allowlists | Check execution order; allowlists cell must run before manual cleaning cell |
| Analysis column not found | Schema changed after clean or merge | Update column resolution cell in gen_analysis |

---

## Project Notes

<!-- Document project-specific decisions here as you work through each dataset.
     Example sections:
     ### [Project Name] — [Date]
     - Join key: customer_id (renamed from cust_id in raw_orders.csv)
     - Dedup rule: keep first on (customer_id, date) — duplicates are re-transmissions
     - Derived columns: derived_days_since_first_order, derived_ltv_bucket
-->
