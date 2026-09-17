# Scenario: Concurrent Feature Modeling with a DAB Template

A walkthrough of setting up a Databricks Asset Bundle (DAB) template repo and
practicing an isolated, concurrent multi-feature branching workflow through
`validation` and `production` environments.

Repo: `jojohughes/dbx-bundle-branching-workflow`
Default branch: `main` (holds the pristine template baseline)

## Branch model

```
main (template baseline, never modified after setup)
 ├── validation   (integration branch — features merge here first)
 ├── production   (release branch — features promoted here)
 ├── feature/emergency  (ED Model work)
 └── feature/surgery    (Surgical Model work)
```

Intended promotion flow: `feature/*` → `validation` → `production`.
`main` is kept clean as the canonical template.

## Steps performed

### 1. Template baseline on `main`
Created a `template/` folder containing a DAB structure and pushed it to `main`:

```
template/
├── databricks.yml, README.md
├── resources/   (silver, gold, ed_model, surgical_model job definitions)
├── targets/     (dev, staging, prod)
└── src/
    ├── common/  (config, transform_utils)
    ├── silver/  (patients, encounters, departments)
    ├── gold/    (dim_patient, dim_department, fact_encounter)
    └── models/
        ├── ed_model/       (silver, gold, metrics)
        └── surgical_model/ (silver, gold, metrics)
```
Files are lightweight stubs (Databricks notebook header + `NotImplementedError`).
Commit: `2614249`.

### 2. Created branches from `main`
`validation`, `feature/emergency`, and `feature/surgery` all branched from `main`.

### 3. Feature edits (kept isolated per feature)
- **feature/emergency** — modified an existing file
  `template/src/models/ed_model/ed_metrics.py`, adding a `door_to_provider_minutes`
  metric stub (commit `c1c97c3`).
- **feature/surgery** — added a new, surgery-specific file
  `template/src/models/surgical_model/surgical_case_duration.py` (commit `33e782e`).

### 4. Merges into `validation` (using `--no-ff`)
- Merged `feature/emergency` → `validation` (`7f7d8b8`). Diff vs `main`: only the ED file.
- Merged `feature/surgery` → `validation` (`011892f`). Diff vs `main`: ED file + surgical file.
- Added a **second** edit to `feature/emergency`
  (`left_without_being_seen_rate` stub, commit `ff979f9`) and re-merged
  `feature/emergency` → `validation` (`e1b3f36`). Only the new commit came across.

### 5. `production` branch (Option 2 — keep `main` pristine)
- Created `production` from `main`.
- Merged `feature/emergency` → `production` (`d65f420`).
- Merged `feature/surgery` → `production` (`78f23f5`).
- Diff `main..production`: exactly the two expected files, nothing else.

## Result — branch tips at time of writing

| Branch            | Tip       | Contents relative to `main` |
|-------------------|-----------|-----------------------------|
| `main`            | `2614249` | template baseline only |
| `validation`      | `e1b3f36` | ED metrics (both edits) + surgical case duration |
| `production`      | `78f23f5` | ED metrics (both edits) + surgical case duration |
| `feature/emergency` | `ff979f9` | ED metrics only |
| `feature/surgery`   | `33e782e` | surgical case duration only |

Isolation held throughout: every merge diff touched only the file(s) owned by
that feature, and no merge produced a conflict.

## Issues & things to watch when doing concurrent feature modeling

1. **CRLF/LF line endings (Windows).** Every commit warned
   `LF will be replaced by CRLF`. Cosmetic here, but on a mixed Win/Mac/Linux
   team it can create spurious diffs. Mitigation: add a `.gitattributes`
   (e.g. `* text=auto eol=lf`) to normalize.

2. **Auto-detected commit identity.** Git guessed the author as
   `...@prominenceadvisors.onmicrosoft.com` because `user.name`/`user.email`
   were unset. Set them explicitly so history has consistent attribution.

3. **Branch naming — slashes not backslashes.** Requested as `feature\emergency`
   (Windows path style); Git requires forward slashes → `feature/emergency`.

4. **The real conflict risk is shared files, not model-specific ones.** In this
   scenario each feature owned a distinct file, so merges were trivial. Two
   features that both edit `src/common/` or the same `resources/*.job.yml` (or
   `databricks.yml`) **will** collide. Keep shared/common changes in their own
   small PRs merged first, then rebase feature branches on top.

5. **Divergence between `validation` and `production`.** They are merged
   independently from the features, so they can drift (e.g. if a feature is
   merged to validation but not yet promoted). Decide the source of truth for
   promotion — promoting `validation` → `production` as a unit is usually safer
   than cherry-picking individual features into both.

6. **`--no-ff` vs fast-forward.** `--no-ff` was used so every integration is a
   visible merge commit (easy to audit/revert a whole feature). Fast-forward
   merges give a linear history but lose that grouping. Pick one convention and
   stick to it.

7. **Re-merging an updated feature.** Merging `feature/emergency` a second time
   after a new commit worked cleanly and brought only the new change — but this
   depends on not rewriting history (no rebase/force-push) on a branch others
   have already merged from.

8. **No branch protection yet.** Nothing prevents a direct push to `main`,
   `validation`, or `production`. For a real workflow, protect these branches and
   require PRs so the template baseline and release branch stay controlled.
