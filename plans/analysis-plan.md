# Forecast evaluation and model selection: analysis plan

Retrospective analysis of the epiforecasts `covid-us-forecasts` archive.

Status: plan for discussion. Two of four planned steps are in scope now; the other two are described briefly so the wider design is visible.

## 1. Context

Between May 2020 and March 2022 our group produced weekly forecasts of COVID-19 deaths for the 50 US states, DC and the US as a whole, at horizons of one to four weeks, and submitted one forecast each week to the US COVID-19 Forecast Hub run by the Reich Lab. Forecasts were probabilistic, expressed as 23 quantiles of the predictive distribution.

Six models ran each week. Three were base models:

- an Rt model, fitting a renewal process directly to reported deaths
- a case convolution model, forecasting cases and then convolving them to deaths using an estimated delay and scaling
- a time series model, combining ARIMA and exponential smoothing forecasts of cases, with deaths regressed on cases

Three were ensembles of those base models: a per-quantile mean, a per-quantile median, and a quantile regression average (QRA) that weights the base models by recent skill.

Selection of the weekly submission followed a fixed rule with a human override. The QRA (trained on a four week window, four week horizon) was submitted by default, falling back to the median ensemble when the QRA was unavailable. A member of the team then reviewed a standard weekly report showing recent data, each model's forecast per state, and the QRA weights, and could replace the forecast for specific states with a different model. Overrides were recorded and 37 weeks contain at least one.

The archive retains all of this: every model's forecast for every week, the ensemble forecasts, the file actually submitted, and the record of which states were overridden and to what. That completeness is what makes the analysis below possible.

## 2. Rationale

Two gaps motivate the work.

The first is about evaluation itself. Forecast performance is usually reported as a single table of scores, which hides how much the conclusion depends on choices made before scoring: which metric, whether to standardise by population, whether to work on the natural or log scale. With six models, 52 locations, four horizons and around 100 weeks, this archive is large enough to show those dependencies concretely rather than assert them. There is a practical output here, a worked example that can serve as teaching material and as a package vignette for `scoringutils`.

The second is about selection. Operational forecasting teams routinely combine an automated rule with expert judgement, but the value added by the human step is rarely measured, because the counterfactual is usually lost. Here it is not. We know what the automated rule would have submitted, what the human actually submitted, and what happened. That allows a direct comparison rather than a hypothetical one.

## 3. Aim and objectives

Aim: use the archive to characterise how forecast evaluation choices affect conclusions about model performance, and to measure whether expert override of an automated selection rule improved forecast skill.

Objectives:

1. Illustrate the process of evaluating performance across many models and many forecast targets, and show how metric and scale choices change what the evaluation says.
2. Compare human judgement against automated selection, using the recorded overrides.
3. Describe the contribution of individual models within an ensemble, both in our own ensembles and, as an extension, in the Hub ensemble our submission fed into.

## 4. Scope: where this sits in a wider design

The full design has four steps. Steps 3 and 4 are the subject of this plan. Steps 1 and 2 require refitting every model and are deferred.

The wider idea is to separate the reasons our historical forecasts were as good or bad as they were. Three variants of each model can be defined:

```
                  era packages              modern packages
                  (2020 to 2022)            (current)
                 --------------------      -------------------
 real-time       O = original
 implementation  (what we submitted)
                       |
                       |  step 2: effect of real-time
                       |          implementation
                       v
 careful         P = era rebuild   ------>  M = modern rebuild
 implementation                    step 1: effect of improved
                                           methods and packages
```

Comparing O with P isolates the cost of building and running code under time pressure, holding the science fixed. Comparing P with M isolates the gain from improved methods and packages, holding implementation quality fixed. Steps 3 and 4 then score whatever variants exist and ask which model should have been chosen.

For now, only O exists, so steps 3 and 4 run on the historical forecasts. The analysis is built so that adding P and M later requires only an extra column in the scored dataset.

Two design points carry over from that wider plan and are worth stating, because they constrain step 3. Model inputs, when we eventually refit, must use the data as it stood at each forecast date, since data revision would otherwise be confounded with implementation. The outcome used for scoring is instead the final data. Steps 1 and 2 will also need to track which package versions were in use when, because the project followed the EpiNow2 development branch and that moved during the study period.

## 5. Data

| Item | Source | Notes |
|---|---|---|
| Forecasts, base models | `submissions/all-models/` | Hub long format, one file per week |
| Forecasts, ensembles | `ensembles/data/` | unweighted (mean, median) and weighted (QRA), plus QRA weights |
| Submitted forecast | `submissions/submitted/` | the human-in-the-loop stream |
| Override records | `submissions/submission-notes/` | 37 weeks, states swapped and to which model |
| Outcome | Reich Lab Hub incident deaths | fixed at the March 2022 vintage, when JHU reporting ended |
| Population | `data/state_pop.csv` | for per-capita standardisation |
| Reference forecasts | Hub `data-processed/` | COVIDhub-baseline and COVIDhub-ensemble, matched on week, location and horizon |

Forecast files use the Hub format: forecast date, target (for example "1 wk ahead inc death"), target end date, location as a FIPS code, quantile level and value, and model name. The analysis uses incident deaths only. Cumulative death targets are derived arithmetically from incident ones in the submission code, so they carry no independent information.

Fixing the outcome at a single data vintage matters. Reported deaths were revised repeatedly, and scoring against a moving target would make results depend on when the analysis was run.

Two forecasts from outside our own set are needed. The Hub baseline is a naive reference that carries recent counts forward with uncertainty widening by horizon. It does two jobs here: it measures how hard each target was, and it is the anchor against which relative skill is reported. The Hub ensemble is the combined forecast our submission fed into, used as a comparator. Both are retrieved from the Hub repository for the same weeks, locations and horizons, and scored against the same outcome as everything else. The baseline is scored as a model in its own right rather than held separately, since pairwise comparison needs it inside the same scored object.

## 6. Methods

### 6.1 Common scoring framework

All scoring uses `scoringutils` (version 2.x). Forecasts are scored at the level of a single model, forecast date, target end date, location and horizon.

The main metric is the weighted interval score (WIS), a proper score for quantile forecasts that rewards both accuracy and appropriate uncertainty. WIS decomposes into three parts, which we report separately because they answer different questions: overprediction, underprediction, and dispersion (how wide the forecast was). Alongside WIS we report absolute error of the median, bias, and how often the 50% and 90% prediction intervals contained the outcome.

Everything is scored on three scales:

- natural, deaths as reported
- per 100,000 population
- log, after adding one to avoid taking the log of zero

The reason for more than one scale is that deaths are highly skewed across states and over time. On the natural scale, a handful of large states during peak weeks dominate the average score, so a model can look good overall while being poor most of the time. Population standardisation and the log transform change what the average is averaging over. The log scale in particular shifts the question from absolute error to relative error, which is closer to what matters at low counts (Bosse et al. 2023).

Forecasts for the US as a whole are carried through every analysis but always reported separately from the states, since a national aggregate is a different kind of target and would otherwise sit as one very large outlier among 51 smaller ones.

The pipeline is built once and cached, and both analyses read the same scored object:

```
forecasts (6 models) ─┐
outcome (Mar 2022)  ──┼──► forecast object ──► scores on 3 scales ──► cached
population          ─┘                                                  │
                                                          ┌─────────────┴────────────┐
                                                          v                          v
                                            objective 1: evaluation     objective 2: selection
```

### 6.2 Objective 1: predictive quality and the effect of evaluation choices

This is the descriptive half of the paper, and the source the vignette is later cut from. Six parts:

1. Describe the outcome before scoring anything. Weekly deaths over time and their distribution across states, raw and per capita. This makes the skew and the range across states visible, which motivates what follows.
2. Score all six models on the natural scale. Report WIS with its components, absolute error, bias and interval coverage.
3. Repeat on the per-capita and log scales, and compare model rankings across the three. The comparison is the point, not any single ranking.
4. Take one model, the mean ensemble, and look at absolute error alone: overall, by horizon, and by state. This shows how much a single summary number hides.
5. Set out how different metrics serve different needs. Catalogue what each metric answers; sketch three users with different concerns (someone allocating scarce resources who is most exposed to overprediction, someone planning surge capacity who is most exposed to the upper tail being too low, and a methodologist who wants one proper score plus a calibration check); show how model rankings change between metrics; and show which metrics move together, so a reader knows how many they need to look at.
6. Close by noting where the P and M variants will slot in later.

### 6.3 Objective 2: human judgement against automated selection

This is a comparison where the analyst knows the outcome in advance, so it will be pre-registered. The pre-registration will be written and time stamped before any of these scores are computed. Objective 1 does not share this constraint, since it is descriptive and its conclusions do not depend on a hypothesis test.

**Streams to compare.**

```
 three base models
        |
        v
      QRA (weights by recent skill)        <-- automated stream (A)
        |                                      median ensemble if QRA missing
        |
   weekly human review of standard report
        |
   override selected states (37 weeks)
        v
   submitted forecast                      <-- human-in-the-loop stream (H)
```

A third stream, the best model per location and week chosen with hindsight, is computed as an upper bound on what any selection rule could achieve. It is not attainable in real time and will be labelled as such.

Only the QRA is used as the automated comparator, since it already selects on skill; adding a second skill-based rule would not answer a different question.

**Estimand.** The difference in skill between H and A, measured as relative WIS on the log scale, on the location-weeks where an override actually happened. The comparison across all location-weeks is secondary, since most contain no human input and dilute the effect toward zero.

**Reconstruction and its check.** We rebuild A from the ensemble files and apply the recorded overrides to produce H, then verify that the rebuilt H matches the submitted files exactly. Any week that does not match is reported rather than quietly dropped. This check is what licenses the whole comparison, so it comes first.

**Adjustment.** Overrides were not made at random. A person looking at a weekly report intervenes where the automated forecast looks wrong, which will tend to be volatile states, turning points and weeks with reporting artefacts. Comparing H against A without accounting for this compares human judgement on hard problems against automation on all problems. We therefore stratify by horizon, by epidemic phase (growth, peak, decline, low), by target difficulty, and by whether an override occurred, and fit a model with random effects for location and week.

Difficulty is measured as the Hub baseline model's score for the same location, week and horizon, grouped into tiers. This puts difficulty in the same units as the thing being explained, and matches how skill is reported in the Hub literature, so our numbers can be read alongside published evaluations.

The Hub ensemble is deliberately not used for this. Our submission was one of its components, so any change the human override made to our forecast also moved the Hub ensemble by a small amount. Adjusting for a variable that the treatment can affect risks removing part of the effect being measured. The Hub baseline is produced independently of us and carries no such feedback, so it is the safe choice. The Hub ensemble still appears as a comparator.

Relative skill is computed by pairwise comparison against a fixed anchor rather than by averaging raw scores. Base models are missing for some location-weeks, and a model that skips difficult weeks would otherwise appear better than it was. The anchor is the Hub baseline, which is present for every location-week, is external to the set of models being evaluated, and is what published Hub evaluations report against, so our skill numbers can be read directly alongside that literature.

Using the baseline as both the difficulty measure and the anchor needs one check, because a ratio conditioned on its own denominator can produce gradients that are artefacts. It does not affect the primary estimand. That estimand is a paired difference between the two streams on the same location-weeks, so on the log scale the baseline term is common to both and cancels exactly, leaving the ratio of the human to the automated score. The anchor choice therefore matters for reporting model-level skill in objectives 1 and 3, and for comparability with the literature, but not for the human against automated comparison. Where baseline score is genuinely the denominator, in the model-level results, we report a sensitivity analysis using an independent difficulty measure.

**What the overrides did.** The records say which state was overridden and to which model, but not why. Rather than treat every override as the same act, we characterise each one by its measurable effect on the forecast, and then ask which kinds helped.

The distinction that matters is between what is fixed at the moment of the decision and what only becomes known later. Two properties of an override are settled as soon as the forecast is chosen, whatever the outcome turns out to be:

- the change in dispersion, that is whether the person picked a wider or narrower forecast than the automated one
- the change in the central estimate, that is the direction and size of the shift in the median

Both are free of hindsight, so both can be fixed in advance as a way of grouping overrides. Anything else, including whether an override reduced overprediction or improved coverage, depends on the outcome and is a result rather than a grouping. Grouping by outcome-dependent quantities and then testing whether those groups did well would be circular.

That gives a two-way typology, widened against narrowed and shifted up against shifted down, and a test of which types were associated with better skill.

The attribution is then exact, because WIS is the sum of its three parts. The change in score from an override separates additively:

```
change in WIS = change in dispersion
              + change in overprediction
              + change in underprediction
```

We report that split for every override, overall and by type. It recovers what the records do not hold: whether the human step was mainly adjusting uncertainty, mainly adjusting the central estimate, or both, and which of those paid off. Bias and interval coverage are reported as diagnostics, but they do not decompose additively and are secondary.

**Ensemble contribution.** A regression of log-scale relative WIS on whether a model is an ensemble or a base model, with horizon as a covariate and random effects for location and week, quantifies the ensembling gain within our own set. This addresses objective 3.

**Representative example.** To make the results concrete we show one forecast in full. The location-week is chosen by rule, as the one whose relative WIS is closest to that model's median, and the rule is fixed in the pre-registration so the example cannot be chosen to suit the conclusion.

**Confirmatory and exploratory.** Confirmatory: the H against A comparison, and the ensemble regression. Everything else, including the component decomposition by override type, interactions with phase and difficulty, and any per-state pattern, is exploratory and labelled that way. The typology is fixed in advance, but with 37 override weeks the per-type comparisons are descriptive rather than tests.

**Sensitivity.** Natural and per-capita scales alongside log; with and without the US national target; and an alternative outcome vintage.

## 7. Implementation

R throughout, tidyverse style, dependencies pinned with `renv`, both analyses written in Quarto. The repository is around 1.5 GB because it stores every weekly forecast, so a sparse checkout of the forecast, submission and reference data directories is enough.

Build order:

1. `R/build_scores.R`, which assembles forecasts, outcome and population into one scored object on three scales and caches it. Both documents depend on this and nothing else.
2. `prereg.qmd`, written and time stamped before objective 2 scores are computed.
3. `vignette-scoring.qmd`, objective 1.
4. `analysis.qmd`, objective 2.

The scored object carries a column for model variant, unused for now, so that adding the step 1 and 2 rebuilds later does not require reworking either document.

### Outputs

The primary output is a paper covering all three objectives. The line running through it is that how forecasts are evaluated determines which forecast gets chosen: objective 1 shows that evaluation choices change model rankings, objective 2 asks what happened when a person overrode a score-based rule, objective 3 quantifies what the ensembles contributed.

Anticipated main figures:

1. The outcome: weekly deaths over time and across states, raw and per capita.
2. Model rankings under each metric and each scale, showing where they disagree.
3. Score by horizon and by location for a single model, showing what an overall average hides.
4. The two selection streams over time, with the model actually submitted marked each week.
5. Change in score from override, split into dispersion, overprediction and underprediction, by override type.
6. Adjusted comparison of the human and automated streams by stratum, against the hindsight bound.

The `scoringutils` vignette is cut from the objective 1 material once the paper is stable, condensed and rewritten with the package in the foreground rather than the epidemiology. Teaching material comes from the same source.

Sketch of the core of the build, to show the shape:

```r
# forecasts: our six models plus the Hub baseline and Hub ensemble,
#            incident deaths, Hub long format
# outcome:   Reich hub incident deaths, March 2022 vintage
# pop:       state populations

fc <- forecasts |>
  filter(type == "quantile") |>                       # drop the point forecast rows
  mutate(horizon = parse_number(target)) |>           # 1 to 4 weeks
  left_join(outcome, by = c("location", "target_end_date")) |>
  left_join(pop, by = "location") |>
  mutate(geo = if_else(location == "US", "national", "state")) |>
  rename(predicted = value, quantile_level = quantile) |>
  as_forecast_quantile(
    forecast_unit = c("model", "forecast_date", "target_end_date",
                      "location", "horizon", "geo")
  )

scores <- bind_rows(
  score(fc)                                         |> mutate(scale = "natural"),
  score(per_capita(fc))                             |> mutate(scale = "per_100k"),
  score(transform_forecasts(fc, log_shift, offset = 1)) |> mutate(scale = "log")
)
```

## 8. Expected results

Stated in advance so that agreement or disagreement is informative.

For objective 1, we expect the ensembles to score better than the base models on average, consistent with the wider forecast hub literature, and we expect coverage to fall short of nominal at longer horizons. The result we are most interested in is that model rankings change between the natural and the log or per-capita scales, because the natural scale is dominated by large states during peaks. If rankings turn out to be stable across scales, that is a useful finding for the teaching material and worth reporting plainly.

On the components, we expect overrides to have widened forecasts more often than narrowed them, because the failure that stands out in a weekly review is an outcome falling outside the interval rather than an interval being too wide. If that holds, the cost appears as increased dispersion and the benefit as reduced underprediction, and the net effect depends on which dominates. A finding that overrides mostly moved the central estimate instead would point to a different kind of judgement being exercised, and is equally worth reporting.

For objective 2 overall, the direction is open. Two patterns seem plausible. Overrides may be concentrated on the automated forecasts that were about to fail worst, in which case the human stream improves on the automated one, most visibly in the tail of the score distribution. Alternatively, overrides may reflect reaction to recent noise, in which case the human stream is slightly worse on average with wider variance. We expect the unadjusted comparison to be biased against the human stream, because intervention was concentrated on harder targets. The size of the gap between the attainable streams and the hindsight bound tells us how much any selection rule could have gained, which sets the scale for interpreting a small effect either way.

## 9. Pointers for interpretation

- Scores on the log scale behave like ratios. A difference in relative WIS is multiplicative, not a number of deaths, and should be described that way.
- Keep the description of an override separate from its result. How wide or how high someone set the forecast describes the decision they made. Whether that was right is an outcome. The component decomposition is informative precisely because the grouping is fixed before the outcome is looked at.
- Rank changes across scales are a result, not a problem to be resolved by picking a favourite scale. The choice of scale encodes what the evaluator cares about, and the vignette should make that explicit rather than recommending one.
- The adjusted and stratified comparison is the primary read for objective 2. The unadjusted difference is descriptive and will understate the human contribution if overrides went to harder targets.
- The hindsight bound is not a target. It says how much selection could ever have gained, which is the context needed to judge whether a small human effect matters.
- Thirty-seven override weeks is not many, and overrides within a week are correlated across states. The analysis can plausibly detect a moderate effect and will not settle a small one. A null result should be reported as limited power, not as evidence that human input has no value.
- Conclusions apply to this team, this pathogen, this period and this automated rule. They do not generalise to expert judgement in forecasting as a whole.
- Reconstruction fidelity is a precondition. If the rebuilt human stream does not reproduce the submitted files, the comparison is measuring our reconstruction rather than the original decisions.

## 10. Decisions taken, and what is still open

Settled:

- The override records hold no stated reason, so overrides are characterised by their measurable effect on the forecast, split into a hindsight-free grouping and an outcome-dependent result (section 6.3).
- Difficulty is measured by the Hub baseline model's score, not by observed volatility.
- The primary output is a paper. The vignette and teaching material are cut from it afterwards.
- The anchor for pairwise relative skill is the Hub baseline, which is also the difficulty measure. The two roles do not conflict for the primary estimand, where the baseline cancels.

Still open, and worth a view before we start:

1. Definition of epidemic phase. Each location-week classified from the observed trajectory into growth, peak, decline or low, with thresholds fixed in advance. The rule still needs agreeing.
2. Multiplicity. One primary estimand is designated. The correction applied across strata and secondary outcomes needs choosing.
3. Whether the Hub ensemble comparison belongs in this paper or in the extension. It is the natural place to ask whether overrides that helped our submission also helped downstream, but it widens the scope.
4. Which independent difficulty measure to use for the model-level sensitivity analysis, given the baseline now serves as the anchor. Observed volatility is the obvious candidate.

## 11. References

- Bracher J, Ray EL, Gneiting T, Reich NG (2021). Evaluating epidemic forecasts in an interval format. PLoS Computational Biology 17(2): e1008618.
- Bosse NI, Abbott S, Cori A, van Leeuwen E, Bracher J, Funk S (2023). Scoring epidemiological forecasts on transformed scales. PLoS Computational Biology 19(8): e1011393.
- Cramer EY et al. (2022). Evaluation of individual and ensemble probabilistic forecasts of COVID-19 mortality in the United States. PNAS 119(15): e2113561119.
- `scoringutils` documentation, https://epiforecasts.io/scoringutils
- Project code: `epiforecasts/covid-us-forecasts`, in particular `submissions/finalise.R` and `submissions/swap-ensembles.R` for the selection and override logic, and `ensembles/models/` for ensemble construction.
