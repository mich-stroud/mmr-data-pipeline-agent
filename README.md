# mmr-data-agent

**An agent-first data pipeline for CMS Monthly Membership Reports (MMR)** — built with Claude Code on dbt + BigQuery, by a healthcare data leader who has run this exact workload in production on an enterprise stack.

> **Status: in active development.** This repo ships incrementally — see the [Roadmap](#roadmap) for what's done and what's next. Built in public, on purpose.

---

## Why this exists

The MMR is the monthly file CMS sends Medicare Advantage and PACE organizations to explain their capitation payment: member-by-member, with risk scores (RAF), adjustment reason codes (ARCs), and payment amounts. Reconciling it against enrollment is how a plan knows it's being paid correctly — and where missed revenue hides.

I've built and operated MMR ingestion in production at a PACE organization (Python + Microsoft Fabric + T-SQL). This project rebuilds that pipeline on the open-source composable stack, agent-first: Claude Code writes nearly all of the code; I set the architecture, review every diff, and verify outputs against the MMR data dictionary.

**No PHI anywhere in this repo.** All member data is synthetic, generated to match the CMS MMR fixed-width layout.

## Architecture

```
synthetic MMR files (fixed-width, per CMS data dictionary)
        │
        ▼
   GCS landing bucket
        │   idempotent loader (filename + file-hash dedup)
        ▼
   BigQuery `bronze`   ── raw records, lineage columns (source file, hash, load_ts)
        │   dbt
        ▼
   BigQuery `silver`   ── typed, parsed, ARC codes decoded, one row per member-month
        │   dbt
        ▼
   BigQuery `gold`     ── capitation revenue by member-month; RAF trend marts
```

Design choices worth noting:

- **Idempotency by filename + file hash.** CMS files get re-sent, renamed, and corrected. The loader records both, so a re-dropped file no-ops and a changed file with a reused name is caught rather than silently double-loaded.
- **Bronze preserves the raw record.** The fixed-width line is stored intact alongside parsed fields, so parsing bugs are re-runnable history, not data loss.
- **Schema derived from the CMS MMR data dictionary** (included in `/reference`), not hand-guessed — the same discipline a real plan's pipeline requires.
- **ARC reason codes as a seeded reference table**, joined in silver so every payment adjustment is explainable.

## Agent-first workflow

This repo is also an artifact of *how* it was built:

- **Claude Code writes ~95% of the code.** My role is architect and reviewer: I specify models and contracts, review diffs, and verify outputs against the data dictionary and hand-checked fixtures.
- **`CLAUDE.md`** documents the conventions the agent follows — naming, layer boundaries, testing expectations — so agent output stays consistent across sessions.
- **Commit history is the receipt.** Commits are small and sequential; where the agent got something wrong (fixed-width parsing off-by-ones are a classic), the fix is visible in history rather than squashed away.

## Repo layout

```
/ingestion      Python loader: GCS → BigQuery bronze, idempotency logic
/dbt            dbt project: staging (silver) and marts (gold) models + tests
/reference      MMR data dictionary, ARC code list (public CMS documentation)
/synthetic      Synthetic MMR file generator + sample files
CLAUDE.md       Agent conventions and guardrails
```

## Roadmap

**Done**
- [x] Synthetic MMR generator matching the CMS fixed-width layout
- [x] Reference data: MMR data dictionary, ARC reason codes
- [x] Bronze/silver/gold dataset design; silver schema drafted

**This week**
- [ ] Idempotent GCS → bronze loader (filename + hash)
- [ ] dbt silver models: typed parsing, ARC decode, member-month grain
- [ ] dbt tests: uniqueness, accepted ARC values, payment reconciliation checks
- [ ] First gold mart: capitation revenue by member-month

**Next**
- [ ] Dagster orchestration (asset-based, schedule on file arrival)
- [ ] Terraform for BigQuery datasets + GCS buckets
- [ ] Metabase dashboard: revenue trend, RAF distribution, ARC breakdown
- [ ] DTRR cross-check: enrollment vs. payment reconciliation

**Later**
- [ ] Semantic layer: canonical definitions (member, RAF, revenue) consumable by humans and agents
- [ ] Anomaly flags: month-over-month RAF and payment deltas

## About

Built by [Michelle Li](https://www.linkedin.com/in/michelle-m-li) — Senior Director of Enterprise Analytics at a PACE organization, where I run CMS data ingestion (MMR, DTRR, 834/820) in production.
