# PACE Enrollment & Revenue Data Platform

An end-to-end data pipeline that ingests one of the file formats at the heart of PACE and Medicare Advantage revenue operations — **CMS MARx Monthly Membership Reports (MMR)** and transforms it into analytics-ready models for capitation revenue, risk adjustment, and enrollment analysis.

Built on the data stack: **Dagster · dbt · BigQuery · Terraform · GCS**, with nearly all code written by AI agents (Claude Code) under human architectural direction and review.

> **All data in this repository is synthetic.** The pipeline includes generators that produce format-valid fake MMR files. No real member or beneficiary data was used at any point.

---

## Why these files

If you run a Medicare Advantage plan or PACE organization, this file is essential to your revenue and enrollment reality:

- **MMR** — the monthly fixed-width file from CMS MARx (495-character records, 91 fields) that tells a plan exactly what it was paid for each member: capitation amounts, risk scores (RAF), adjustment reason codes, Part A/B/D components. Reconciling it correctly is the difference between knowing your revenue and guessing.

I've built and operated ingestion of many CMS files in production healthcare settings. This project is a clean-room rebuild on an open-source stack, driven from the public CMS layout specifications.

## Architecture

```
 synthetic          ┌──────────┐      ┌─────────────────────────────┐
 generators ──────► │   GCS    │────► │  Dagster (asset-based)      │
 (MMR / 834)        │  landing │      │  parsers → bronze loading   │
                    └──────────┘      └──────────────┬──────────────┘
                                                     ▼
                    ┌─────────────────────────────────────────────┐
                    │  BigQuery                                   │
                    │  bronze ──► silver ──► gold   (dbt)         │
                    │  raw+meta   conformed   marts + metrics     │
                    └──────────────────────┬──────────────────────┘
                                           ▼
                                   Metabase dashboards
```

- **Bronze:** one row per source record, typed, with file lineage metadata; invalid records quarantined, not dropped; parse new files only, old files skipped.
- **Silver:** conformed entities — `mmr_capitation_months`
- **Gold:** `dim_enrollment`,  `fct_mmr_payment`, `fct_raf`
- **Infra:** BigQuery datasets and GCS buckets provisioned via Terraform

## Highlights

- **Layout-driven MMR parsing.** The parser is driven by a machine-readable layout CSV (field name, position, length, type) rather than hardcoded offsets — the same approach that survives CMS layout revisions in production.
- **Idempotent ingestion.** Reprocessing a file never duplicates data.
- **Tested.** dbt schema and data tests on every model, pytest coverage on parsers, Dagster asset checks for row-count and freshness anomalies, CI via GitHub Actions.
- **Synthetic data generators as spec proof.** Generating a valid 495-character MMR record requires understanding all 91 fields; `generators/` doubles as executable documentation of the layouts.

## Built agent-first

This repo was built with AI agents (Claude Code) writing nearly all of the code, directed by a phased architectural brief ([`docs/build_prompt.md`](docs/build_prompt.md)) with human review at every phase.

What that looked like in practice:

- I set the architecture, stack, layer boundaries, and data rules up front; the agent proposed structure and implementation within them.
- Every phase ended with review against a domain checklist before continuing.
- Domain logic was the main correction surface. Examples of issues caught in review:
  - *[FILL IN during build — e.g., "agent initially treated 834 term dates as exclusive; corrected to match X12 semantics"]*
  - *[FILL IN — e.g., "invented a plausible-sounding but nonexistent adjustment reason code in the generator"]*
  - *[FILL IN — at least 2–3 real examples; these matter more than the rest of this section]*

The takeaway from building this way: agents are excellent at the code and unreliable on the domain. The leverage is in specification and review, and healthcare data punishes anyone who skips the second part.

## Running it locally

```bash
# prerequisites: uv, terraform, gcloud CLI (or use local mode)
git clone <repo-url> && cd <repo>
uv sync

# provision (or skip and use local filesystem + DuckDB dev mode)
cd terraform && terraform apply

# generate synthetic data, run the full pipeline, build dbt models
make demo
```

*(See [`docs/local_dev.md`](docs/local_dev.md) for the no-GCP local path.)*

## Repo structure

```
terraform/       BigQuery datasets, GCS buckets
generators/      synthetic MMR + 834 file generators
ingestion/       Python parsers, Dagster asset definitions
dbt/             staging / intermediate / marts / metrics
docs/            semantic layer definitions, build prompt, local dev
tests/           pytest suites for parsers and generators
```

## About

Built by **Michelle Li** — healthcare analytics leader specializing in PACE, risk adjustment (HCC/RAF), and CMS revenue data. This project is a personal rebuild of production patterns on an open-source stack; all layouts sourced from public CMS documentation, all data synthetic.

[LinkedIn](#) · [Email](mailto:limichelle1122@gmail.com)
