# us-forecast-eval

context

- evaluating the epiforecasts weekly COVID-19 death forecasts for US states, submitted to the US COVID-19 Forecast Hub May 2020 to March 2022
- source archive: [epiforecasts/covid-us-forecasts](https://github.com/epiforecasts/covid-us-forecasts)
- plan: [`plans/analysis-plan.md`](plans/analysis-plan.md), and the chat it came from, with the options considered: [`plans/2026-11-12-chat-history.md`](plans/2026-11-12-chat-history.md)
- handle: [kath-a-log#183](https://github.com/kathsherratt/kath-a-log/issues/183)

all work is exploratory / experimental until the step 4 pre-registration is time-stamped

## layout

| Path | What is in it |
|---|---|
| `plans/` | The analysis plan and its history. The plan is the source of truth |
| `R/` | Scripts. `R/build_scores.R` comes first; both analyses read its cached output |
| `notebook/pages/` | Site pages, one `.qmd` each |
| `data/` | Cached inputs and the scored object. Not the raw archive |

## publishing

Site publishes to GitHub Pages via `.github/workflows/publish.yml` on push to `main`.

Uses Quarto freeze (`_quarto.yml`: `execute: freeze: true`) so CI never installs R or executes code - a project render always assembles the site from cached results in `_freeze/`, no matter which page changed.

After changing R code or data that feeds a `.qmd` page:

1. Render that page directly to re-execute it and refresh its cache: `quarto render notebook/pages/<page>.qmd` (a project-level `quarto render` won't re-execute under `freeze: true`)
2. Commit the updated `_freeze/` alongside your changes
3. Push

If you forget, CI has no R to fall back on and the render step fails loud rather than publishing stale content.
