# Claude Code Prompt — Enrollment & Revenue Pipeline (MMR)

> Usage: paste the Project Brief below as your opening prompt in Claude Code, then work phase by phase. Don't paste all phases at once — direct one phase, review the output, then move on. Keep notes on what you corrected; that's your review log for the README.

---

## Project Brief (opening prompt)

You are helping me build a production-style data pipeline project. I am the architect and reviewer; you write the code. I will review everything and I expect to push back.

**What we're building:** An end-to-end data platform that ingests one healthcare enrollment/payment file format — the CMS MARx Monthly Membership Report (MMR), a fixed-width monthly file — and transforms it through a bronze/silver/gold medallion architecture into analytics-ready models with a semantic layer.

**Stack (non-negotiable):**
- Google Cloud Storage for raw file landing (with a local-filesystem fallback for dev)
- Dagster for orchestration (software-defined assets, not ops/jobs)
- dbt for transformation, targeting BigQuery
- Terraform for provisioning the BigQuery datasets and GCS buckets
- Metabase (or BigQuery views ready for it) for the BI layer
- Python 3.12, managed with uv
- pytest for Python tests, dbt tests for models

**Data rules:**
- ALL data is synthetic. A synthetic-file generator already exists at `code/generate_mmr_files.py`; generated MMR files land in `data/mmr`. No real member data ever enters this repo.
- MMR records are fixed-width, 495 characters, 91 fields. The machine-readable layout (the MMR data dictionary: field name, start position, length, type) is already in the repo — locate it before writing any parser code and treat it as the single source of truth. The parser must be driven by it, not hardcoded offsets.
- This project focuses on Part A/B payment fields; Part D fields may be blank in the synthetic data. Do not treat blank Part D fields as data quality failures.
- ARC (adjustment reason code) definitions are already in the repo as a reference file; they are joined in at the silver layer.
- A mock schema for the silver model exists at `mmr_capitation_month_layout.md` — this is my target design for `silver.mmr_capitation_month`. Read it before Phase 4 and build to it; ask me about any conflicts between it and this brief rather than resolving them silently.
- At the start of Phase 1, inventory these existing assets (generator, data dictionary, ARC reference, silver mock schema), confirm their exact paths back to me, and propose where they should live in the final repo structure — do not move or rename anything without my approval.

**Schema policy (bronze):**
- The bronze BigQuery schema is DERIVED from the MMR data dictionary (names converted to snake_case, types mapped from the layout), not autodetected from data.
- Derive it once, write it to a committed schema artifact (e.g. JSON schema file + dbt source YAML), and load against that pinned schema from then on. The schema does not change unless the CMS layout changes, in which case we re-derive and commit a new version deliberately.
- Include filler fields in bronze — bronze mirrors the file, business filtering happens later.
- One source of truth: the data dictionary drives the parser slicing, the pinned BigQuery schema, and the dbt source definitions.

**Architecture principles I want you to follow:**
- Bronze: raw files parsed to typed tables, one row per source record, with file metadata (filename, file hash, ingest timestamp, row number) preserved. No business logic.
- Layer naming: BigQuery datasets are named `bronze`, `silver`, and `gold` — the layer lives in the dataset name, so table names carry no layer prefix (e.g. `silver.mmr_capitation_month`, not `silver.slv_mmr...`).
- Silver: cleaned, deduplicated records with capitation months exploded — one row per member per capitation month per payment month — with ARC code definitions joined in from the ARC reference file. Model name: `mmr_capitation_month` (snake_case naming throughout; no scratch-style names like `stg1`).
- Capitation proration (core silver transformation): CMS pays adjustment spans as lump sums. Example: adjustment_start_date 2026-01-01 to adjustment_end_date 2026-06-30, paid on payment date 2026-07-01, capitation amount $6,000 — that single record covers 6 capitation months. The transformation must: (1) compute the number of months in the span from adjustment start/end dates (inclusive; the example is 6 months, and start=end month means 1 — never divide by zero), (2) explode the record to one row per capitation month in the span, carrying the payment month on every row, (3) divide each prorated amount field by the month count, so the example yields 6 rows of $1,000 each. The mock schema file (`mmr_capitation_month_layout.md`) is authoritative on which fields prorate: columns annotated "divide by number of capitation months" are prorated (the five Part A/B/C amount fields); ALL other fields — including every risk adjustment factor — carry over unchanged on each exploded row. Do not prorate anything that isn't annotated. The capitation month grain is expressed through the adjustment date columns themselves — no separate grain column: on each exploded row, set payment_adjustment_start_date to the first day of that row's capitation month and payment_adjustment_end_date to the last day of that month. In the example, the six rows carry start/end dates 2026-01-01/2026-01-31 through 2026-06-01/2026-06-30, each with the same payment month. The original full span is preserved in bronze; silver-to-bronze tie-outs reconcile against it there. Handle rounding so the prorated rows sum back exactly to the original amount (allocate any remainder pennies deterministically, e.g. to the final month) — the member-level dollar tie-out test depends on this.
- Negative adjustments: adjustment records can carry NEGATIVE amounts (e.g. retroactive disenrollment clawbacks — CMS recouping capitation already paid). Proration handles them identically: same month-count, same explosion, same division — a -$6,000 six-month span yields 6 rows of -$1,000, and rounding remainders follow the same deterministic allocation with sign preserved (e.g. -$100 over 3 months → two rows of -$33.33 and one of -$33.34). Tie-out tests sum SIGNED amounts, never absolute values; member-level and month-level totals can legitimately be negative. The synthetic data generator must include some negative-amount adjustment records (paired naturally with ARC 03 disenrollments) so this path is actually exercised.
- Gold: business-level marts — `member_months`, `monthly_capitation_revenue`, `raf_distribution`.
- Semantic layer: canonical definitions for member_month, raf_score, pmpm_revenue as documented dbt metrics/models, so every downstream consumer uses one definition.
- Idempotent ingestion: track processed files by filename AND file content hash. If a filename+hash pair has already been ingested, skip it (do not re-ingest). If a known filename arrives with a NEW hash (CMS reissued a corrected file), do not silently ingest or skip — surface it and route it through the explicit reprocess path.
- Reprocess path: provide an explicit mechanism (Dagster op or make target) that, for a given file, deletes its bronze rows and re-ingests it. Reprocessing is always deliberate, never automatic.
- Data quality checks, implemented as dbt data tests so they run on every build (note: ROW counts will not match across layers by design once capitation months are exploded — checks are member-level and dollar-level):
  - Bronze → Silver: every ingested file is represented; distinct member count per file matches; capitation dollars tie out at the member level.
  - Silver → Gold: distinct member count per payment month carries over; pmpm revenue in the pmpm table matches silver; RAF score matches per member.

**How we'll work:** I'll direct one phase at a time. After each phase, stop and summarize what you built and what decisions you made, so I can review before we continue. If a spec detail is ambiguous, ask me — do not invent healthcare domain logic. I know these file formats from production experience and I will catch errors.

Start by proposing the repo directory structure and the Terraform layout. Do not write pipeline code yet.

---

## Phase prompts (use one at a time, in order)

### Phase 1 — Scaffolding & infra
First, inventory the existing assets per the data rules (generator at `code/generate_mmr_files.py`, MMR data dictionary, ARC reference file, silver mock schema at `mmr_capitation_month_layout.md`) and confirm their paths back to me. Then propose the repo structure — `terraform/`, `ingestion/` (Python parsers + Dagster assets), `dbt/` (staging/intermediate/marts layers), `data/`, `tests/` — including where the existing assets should live; get my approval before moving anything. Write the Terraform for three BigQuery datasets (`bronze`, `silver`, `gold`) and a GCS bucket with sensible lifecycle rules. Add a Makefile or justfile with the common commands. Include a `.gitignore` that excludes credentials, `.env`, and any locally generated scratch data. Check that the required software is installed and the configuration matches the brief above.

### Phase 2 — Synthetic data & pinned schema
1. Review the existing synthetic data in `data/mmr` for format validity and completeness (Part A/B focus; Part D may be blank).
2. Review `code/generate_mmr_files.py` and parameterize it (do not rewrite it from scratch) to produce the demo dataset: 6 monthly files, ~100 members each, with 4–5 new members enrolling per month (ARC 02) and 2–3 members disenrolling per month (ARC 03), so the population grows slightly over the 6 months and `make demo` can regenerate the dataset from scratch. Include some negative-amount adjustment records (retro clawbacks tied to disenrollments), including at least one multi-month negative span, so the proration and tie-out logic gets exercised on negatives. I will review generated output against the real layout myself.
3. From the MMR data dictionary, derive the pinned bronze schema per the schema policy: snake_case column names, explicit types, filler columns included. Commit the schema artifact and the dbt source YAML generated from it. Propose the type mapping for my review before committing.

### Phase 3 — Parser & bronze ingestion
Layout-driven MMR parser: reads the data dictionary, processes each file in the landing folder, computes the file hash, checks filename+hash against the processed-files table, skips already-ingested files, surfaces same-name/new-hash files for explicit reprocess, slices fixed-width records, loads to the pinned bronze schema, and flags records that fail validation into a quarantine table rather than crashing. Wrap as a Dagster asset that watches a GCS prefix (or local dir), tracks processed files, and loads bronze tables in BigQuery. Include the explicit reprocess op. Idempotency required.

### Phase 4 — dbt silver & gold
Staging models over bronze (renaming, typing, replay-safe). Silver: `mmr_capitation_month`, built to the mock schema in `mmr_capitation_month_layout.md` — apply the capitation proration logic from the brief (month-count from adjustment start/end dates, explode to one row per capitation month per payment month, divide prorated amount fields by month count with exact-sum rounding), join ARC definitions from the reference file; surface any conflict between the mock schema and this brief to me instead of resolving it silently. Gold: `member_months`, `monthly_capitation_revenue`, `raf_distribution`. Implement the bronze→silver and silver→gold tie-out checks as dbt data tests. Document every model and column in schema.yml — I'll review definitions for domain correctness.

### Phase 5 — Semantic layer & BI
Define dbt metrics (or a metrics mart) for member_month, pmpm_revenue, avg_raf. Create the BigQuery views Metabase will point at. Write a short `docs/semantic_layer.md` explaining each canonical definition and why it's defined that way (I will edit this one heavily — the definitions are mine).

### Phase 6 — Hardening
Dagster asset checks (row-count deltas, freshness), a CI workflow (GitHub Actions: lint, pytest, dbt build against DuckDB or a test dataset), and a `make demo` path that regenerates the synthetic dataset and runs the whole pipeline locally end-to-end.

---

## Review checklist (yours, not Claude's — apply every phase)

- [ ] MMR field offsets: spot-check ~10 fields against the data dictionary by hand
- [ ] Pinned schema: verify type mapping field-by-field before committing; confirm loads fail loudly on schema drift rather than silently coercing
- [ ] ARC codes: enrollment (02) / disenrollment (03) land correctly in generator output and silver joins; xref join doesn't drop members with unmapped codes
- [ ] Capitation-month explosion: member-level dollar totals tie between bronze and silver (row counts will differ by design)
- [ ] Proration: hand-verify one multi-month span end to end (e.g. the 6-month/$6,000 case → 6 rows × $1,000); check an uneven division (e.g. $100 over 3 months) sums back exactly; confirm a single-month record produces exactly 1 row divided by 1
- [ ] Proration scope: ONLY the five annotated amount fields are divided; spot-check that RAF factors and all other fields carry identical values across a span's exploded rows
- [ ] Negative adjustments: hand-verify one multi-month negative span (division, sign-preserving rounding, exploded rows sum back to the original negative amount); confirm tie-outs sum signed values and a clawback month can go negative without failing tests
- [ ] Payment vs capitation month: every exploded row carries the correct payment month; each row's adjustment start/end dates are the first/last day of its own capitation month, and the months collectively cover the original span exactly, inclusive of both endpoints
- [ ] Payment fields: Part A/B amounts land in the right columns; blank Part D is tolerated, not quarantined
- [ ] Idempotency: run the same file twice → skipped; alter one byte of a processed file → surfaced for reprocess, not silently ingested
- [ ] Reprocess path: deletes the file's bronze rows and re-ingests cleanly, no dupes downstream
- [ ] dbt tests actually fail when you inject a bad record (test the tests)
- [ ] Anything the agent invented that "sounds like healthcare" but isn't real — log every catch; these become your README examples
