# Template — Databricks Asset Bundle

Reference template for the branching workflow. Copy this structure when starting new work.

## Layout

- `databricks.yml` — bundle definition (name, variables, targets).
- `resources/` — job/pipeline definitions, one file per pipeline.
- `targets/` — per-environment overrides (dev / staging / prod).
- `src/common/` — shared helpers (config, schemas, utilities).
- `src/silver/` — common silver transforms (shared across all models).
- `src/gold/` — common gold transforms (shared dims/facts).
- `src/models/ed_model/` — ED Model specific silver → gold → metrics.
- `src/models/surgical_model/` — Surgical Model specific silver → gold → metrics.

## Layering

`common silver/gold` are enterprise-wide transforms every model builds on.
`models/*` hold the domain-specific transforms that apply to only that model.
