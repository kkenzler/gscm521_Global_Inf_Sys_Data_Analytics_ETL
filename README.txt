ETL Pipeline
============
Generic three-stage data cleaning, merging, and analysis pipeline implemented
as Jupyter notebooks. Each stage produces auditable Excel workbooks with
human-reviewable records of every decision made. Designed for datasets that
need traceable, documented transformation before analysis.


HOW TO START
------------
Windows/Mac/Linux: Open each notebook in JupyterLab or VS Code and run cells
in order. Run the three notebooks sequentially:

  1. gen_clean.ipynb    -- normalize, deduplicate, audit
  2. gen_merge.ipynb    -- join reconnaissance, merge plan, left-joins
  3. gen_analysis.ipynb -- statistical analysis (customize for your dataset)

Place raw input files (CSV or simple XLSX sheets) in the _raw_data/ folder
before starting.


PREREQUISITES
-------------
Python packages:

  pip install pandas openpyxl jupyter

Jupyter environment (any of):
  JupyterLab: pip install jupyterlab  then  jupyter lab
  VS Code:    install the Jupyter extension


USAGE
-----
Stage 1 -- gen_clean.ipynb:
  - Loads all CSVs from _raw_data/.
  - Normalizes column names and detects issues without auto-fixing them.
  - Writes _clean_data/_clean_audit.xlsx for your review.
  - Type YES at each gate to confirm you have reviewed the audit before
    the notebook writes any cleaned output.
  - Optionally configure split_allowlist, exact_row_tables, and key_rules
    in the allowlists cell before proceeding.
  - Output: cleaned CSVs in _clean_data/.

Stage 2 -- gen_merge.ipynb:
  - Loads clean CSVs from _clean_data/.
  - Runs join reconnaissance across all table pairs and builds a ranked
    merge plan with risk flags.
  - Writes _merge_output/_merge_audit.xlsx for your review.
  - Type YES at the execute gate to run auto-approved joins.
  - Manually force-through any flagged joins you have reviewed and approved.
  - Output: _merge_output/_merged_final.csv

Stage 3 -- gen_analysis.ipynb:
  - Loads _merged_final.csv.
  - Requires manual editing: update the column resolution cell at the top
    to map semantic roles (ENTITY_ID_COL, DATE_COL, VALUE_COL, etc.) to
    the actual column names in your merged dataset.
  - Sections 1-3 are generic starters; Section 4 is a placeholder for
    dataset-specific analysis. You will always need to customize this notebook.
  - Output: _analysis_output/<base_table>__analyses.xlsx and PNG charts.


OUTPUT
------
_clean_data/
  _clean_audit.xlsx       -- audit workbook: column map, dedup summary,
                             data dictionary, applied steps
  <table>_clean.csv       -- one cleaned CSV per input table

_merge_output/
  _merge_audit.xlsx       -- join candidate rankings, merge plan, coverage
  _merged_final.csv       -- single merged dataset ready for analysis

_analysis_output/
  <base>__analyses.xlsx   -- summary tables and data dictionary
  __<N>_*.png             -- charts produced per analysis section


TROUBLESHOOTING
---------------
delta_rows non-zero after join     Right-side join key is not unique. Check
                                   merge_audit: join_cardinality_label and
                                   right_key_quality_bucket columns.

Many nulls in joined columns       Low key overlap. Check merge_audit:
                                   evidence_max is low or coverage_risk=high.

no_candidate in merge plan         No name-similar column pair found. Lower
                                   name_sim_min in the merge_audit() call or
                                   handle the join manually.

Analysis column not found          Schema changed after clean/merge. Update
                                   the column resolution cell in gen_analysis.
