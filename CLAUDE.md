# us-forecast-eval

Retrospective evaluation of the epiforecasts US COVID-19 death forecasts, 2020 to 2022. Kath's project. Handle: kathsherratt/kath-a-log#183.

## Source of truth

`plans/analysis-plan.md`. Read the section a task names, not the whole file. Do not change a settled decision in section 10 without asking. `plans/2026-11-12-chat-history.md` is background only; open it only when a task asks why a choice was made.

Steps 3 and 4 are in scope. Steps 1 and 2 are parked.

## Build order (plan section 7)

1. `R/build_scores.R`: one cached scored object, three scales, a `variant` column
2. `notebook/pages/prereg.qmd`: written, committed, pushed and tagged `prereg-v1` before any step 4 score is computed
3. `notebook/pages/vignette-scoring.qmd`: objective 1
4. `notebook/pages/analysis.qmd`: objective 2

The pre-registration rule is the one hard constraint. Nothing that compares the human and automated streams runs before the `prereg-v1` tag exists.

## Data

The archive repo is about 1.5 GB. Never clone it in full. Use a sparse checkout of `submissions/`, `ensembles/data/` and `data/` only. Outcome: Reich Lab Hub incident deaths at the March 2022 vintage.

## Code

- R, tidyverse, native pipe `|>`
- `scoringutils` 2.x API (`as_forecast_quantile()`, `score()`, `summarise_scores()`); check helper names against the installed version
- Small chunks, a comment above each saying what it is for
- Pin dependencies with `renv`
- Render pages locally and commit `_freeze/` (see README)

## Writing

Sentence case headings. No em dashes. Plain, factual, no hyperbole.

## Issues

One step label each: `step:model`, `step:implementation`, `step:prediction`, `step:selection`. `decision` for an open choice from plan section 10. Close a decision issue by writing the answer into the plan.
