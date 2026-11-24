<p align="center"><img src="assets/banner.svg" alt="Construction Project Controls Platform" width="100%"></p>

# Construction Project Controls Platform: Research Report

> **Where the code is.** The experiment scripts that produced these results are part of the source package, which is distributed separately (see [SOURCE-ACCESS.md](../SOURCE-ACCESS.md)). The raw results they wrote are included in this repository under [`results/`](../results/).

> **Status: protocol v1.0 with results on SYNTHETIC data only.** The protocol (hypotheses, methods, statistical plan) was fixed before the reported run; where it was made more specific or departed from, the change log in section 12 says so. Every number below comes from files in [results/](../results/) written by `npm run measure`. The simulator that generated the projects is the author's own model, not calibrated to any real project, so **these results say how the forecasting methods behave on that model and nothing about real construction outcomes.** Outputs of the product are planning aids, not engineering, legal or safety advice.

## Abstract

How accurate are cost-at-completion and finish-date forecasts from standard earned-value methods compared with a ridge-regression model and a bottom-up approach, and how early can they be trusted? On 3,000 held-out synthetic projects per checkpoint (5 independent replicates of 200 test projects at each of three change-order volumes, with models trained only on separate training projects), a ridge regression on earned-value, pending-change-order and progress features had a mean absolute percentage error 40% to 48% lower than the CPI-based formula at every one of ten completion checkpoints. Earned schedule had a lower mean finish-date error than the traditional cost-based SPI formula at every checkpoint, and the gap was widest late in the project (5.2 days against 16.2 days at 90% complete). Accuracy did **not** degrade gracefully for the regression model when reporting was delayed: its error at a four-week delay was about twice its error with no delay, while the CPI-based formula's error grew by 17%. All of this is conditional on a generator in which change orders and performance drift were built in by the author.

## 1. Research question

How accurate are cost-at-completion and schedule forecasts from standard earned-value methods compared with regression and bottom-up approaches, and how early in a project can they be trusted?

The **bottom-up** approach (committed cost, budget hold and weighted change-order exposure by cost code) is implemented in the product and shown next to the other methods, but it is **not evaluated in these experiments** because the simulator has no cost-code structure or commitments. This is a gap against the original research question.

## 2. Hypotheses

- H1: Regression models using earned-value, change-order and commitment features reduce estimate-at-completion error by at least 20% versus the CPI-based formula at 30-70% project completion. (Commitment features are not available in the simulator; the regression uses earned-value, progress and pending change-order features only.)
- H2: Including pending change-order exposure cuts forecast error materially in projects with high change volume.
- H3: Earned-schedule forecasts predict finish dates better than the traditional schedule-performance-index formula in the later half of a project.
- H4: Forecast accuracy degrades gracefully when data is entered with a one-to-four-week lag.

Decision criteria, written down in `experiments/summarize.ts` before the reported run and copied into `results/summary.json`:

- **H1** supported if the regression's mean absolute percentage error is at least 20% lower than the CPI-based formula's at every checkpoint of 30%, 40%, 50%, 60% and 70%, with Holm-adjusted paired Wilcoxon p < 0.05 at each.
- **H2** ("materially" operationalised as at least 10%) supported if, in high-change-volume projects, adding pending change-order exposure to the regression reduces per-project mean absolute percentage error (averaged over the 30-70% checkpoints) by at least 10%, the bootstrap 95% interval is above zero and the Holm-adjusted p (across the three volumes) is below 0.05.
- **H3** supported if the earned-schedule finish forecast has a lower mean absolute finish-date error than the traditional SPI formula at every checkpoint of 50% to 90% (50, 60, 70, 80, 90), with Holm-adjusted p < 0.05 at each.
- **H4** ("gracefully" operationalised as at most 1.5 times) supported if the error with a four-week reporting delay is at most 1.5 times the error with no delay, for both the CPI-based and the regression estimates (point estimates).

## 3. Background

**Not written.** The planned 5 to 8 source related-work section was not done. The two starting references in section 11 have not been verified against the originals.

## 4. Method

### 4.1 System under test

The forecasting code of the platform: `src/domain/evm.ts` (metrics, EAC formulas, earned schedule) and `src/domain/regression.ts` (ridge regression). The experiment harness calls the same functions the product uses (`eacMethods`, `eacFeatures`, `earnedSchedule`, `fitRidge`).

### 4.2 Environment: the synthetic project simulator

Real project histories were not available, so every result uses `src/sim/simulator.ts`. Time is in whole weeks. Per project:

- Base scope with budget `bac0` (log-uniform between 2 million and 200 million) and planned duration D of 30 to 90 weeks (uniform, whole weeks); planned cumulative value follows the S-curve 3x^2 - 2x^3 of elapsed fraction.
- Schedule performance: earned schedule advances each week by `rate x exp(eta)` where `rate` is lognormal with median 0.95 (sigma 0.10, so projects are on average a little late) and `eta` is an AR(1) noise (phi 0.7, sigma 0.06). The base scope finishes when earned schedule reaches D, at fractional week T.
- Cost performance: each week's earned value costs `q x exp(kappa)` where `q` is lognormal with median 1.05 (sigma 0.10, a small average overrun) and `kappa` is AR(1) noise (phi 0.8, sigma 0.03).
- Change orders arrive as a Poisson process at 3, 10 or 25 per year (low, medium, high volume). Each has an amount of 0.6% of the base budget times a lognormal factor (sigma 0.8); it is "potential" from its arrival week until a decision 2 weeks plus an exponential delay (mean 6 weeks) later; with probability 0.7 it is approved (the budget rises by its amount and the work is performed over 8 weeks at a cost factor close to q), otherwise it is rejected and costs nothing.
- A controls team sees, at any week, the revised budget, earned value, actual cost, planned value (revised budget times the planned fraction) and the **unweighted** total of change orders that are still undecided.

All parameters are the author's assumptions (`DEFAULT_CONFIG`, recorded in `results/run_metadata.json`). Two properties matter for reading the results: the generator makes pending change orders informative about the final cost by construction (a fraction of them is approved and then costs money), and it contains a systematic average cost overrun and schedule slip that a trained model can learn.

### 4.3 Baselines

- Budget at completion (no forecast): EAC = revised BAC
- CPI-based estimate at completion: EAC = BAC / CPI
- Traditional schedule forecast: planned duration / (EV / PV)
- Earned-schedule method: planned duration x AT / ES (that is, PD / SPI(t))
- Planned finish (no forecast)

### 4.4 Independent variables

Forecast method; percent complete at forecast time (ten checkpoints: 10%, 20%, ... 90%, 95% of the revised budget earned, taken at the first week at which the checkpoint is reached); change-order volume; data-entry lag (0 to 4 weeks, at the 50% checkpoint).

### 4.5 Metrics

- Mean absolute percentage error (MAPE) of the estimate at completion against the simulated final cost
- Absolute finish-date error in days (weeks x 7) against the simulated finish of the base scope
- Forecast error against percent complete (the checkpoint tables)

The protocol also listed a rank correlation between predicted and actual overrun; it was **not computed**.

### 4.6 Models and training

Two ridge regressions per checkpoint, trained on training projects only: target = final cost / revised BAC at that week, features = 1, 1/CPI, (1/CPI)(1 - fraction complete), pending exposure / BAC, SPI (cost-based), fraction complete; the "without exposure" variant drops the exposure column. A third model per checkpoint predicts finish duration / planned duration from 1, AT/ES, (AT/ES)(1 - fraction complete), PV/EV, fraction complete and exposure / BAC. Ridge penalty 0.001 times the number of training rows, applied to standardised features. Training and test projects use disjoint seed ranges (a test in `tests/experiments.test.ts` checks this for the full sizes). The training set mixes the three change volumes.

## 5. Experiment matrix

| ID | Experiment | Manipulation | Measured |
|---|---|---|---|
| E1 | Cost forecast accuracy | Baselines vs. regression at 10 completion checkpoints | MAPE by checkpoint |
| E2 | Value of change-order data | With and without pending change-order features | Error reduction by change volume |
| E3 | Schedule forecasting | SPI-based vs. earned-schedule vs. regression | Finish-date error |
| E4 | Data-lag robustness | Inject reporting delays of 1 to 4 weeks | Error growth |

## 6. Statistical plan

- **Replication:** 5 replicates; each replicate generates 300 training and 200 test projects at each of three change volumes (900 training, 600 test); 3,000 test projects per checkpoint in total. Fixed before the reported run (`FULL_SIZES` in `experiments/lib.ts`).
- **Summaries:** means with percentile-bootstrap 95% intervals (1,000 resamples of projects, seeded), plus medians and IQR in `results/summary.json`.
- **Comparisons:** paired Wilcoxon signed-rank tests (normal approximation with tie and continuity corrections, `src/stats/index.ts`, verified against a hand-computed value in `tests/stats.test.ts`).
- **Effect sizes:** the relative error reduction with its bootstrap interval is reported in the tables; the paired dominance statistic (a paired form of Cliff's delta) is in `results/summary.json`.
- **Multiple comparisons:** Holm-Bonferroni across the five checkpoints of each hypothesis window (H1, H3) and across the three volumes (H2).
- **No silent exclusions:** projects for which a method is undefined at a checkpoint are skipped and counted; the count was 0 in all three experiments (`skips` in `results/summary.json`).
- **Negative results are results:** H4 was not supported.

## 7. Results

All data are synthetic. "MAPE" is a mean absolute percentage error of the estimate at completion against the final simulated cost.

### E1. Cost forecast accuracy

**Manipulation:** Baselines vs. regression at 10 completion checkpoints
**Measured:** MAPE by checkpoint

<!-- table:e1 -->
| Checkpoint (% complete) | Projects | MAPE: budget at completion | MAPE: CPI-based [95% CI] | MAPE: regression without exposure | MAPE: regression [95% CI] | Reduction vs CPI [95% CI] | Holm p (30-70% only) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 10% | 3000 | 12.83% | 7.47% [7.26%, 7.68%] | 5.62% | 4.44% [4.32%, 4.57%] | 40.5% [38.46%, 42.44%] | - |
| 20% | 3000 | 12.25% | 6.61% [6.43%, 6.80%] | 4.89% | 3.75% [3.64%, 3.87%] | 43.3% [41.28%, 45.05%] | - |
| 30% | 3000 | 11.72% | 5.82% [5.66%, 5.98%] | 4.32% | 3.29% [3.19%, 3.40%] | 43.5% [41.56%, 45.38%] | 6.9e-164 |
| 40% | 3000 | 11.24% | 5.06% [4.92%, 5.22%] | 3.77% | 2.85% [2.77%, 2.93%] | 43.7% [41.97%, 45.54%] | 1.7e-162 |
| 50% | 3000 | 10.74% | 4.30% [4.18%, 4.43%] | 3.22% | 2.37% [2.29%, 2.44%] | 45.0% [43.18%, 46.76%] | 4.4e-172 |
| 60% | 3000 | 10.28% | 3.54% [3.42%, 3.65%] | 2.72% | 1.95% [1.89%, 2.01%] | 44.8% [42.98%, 46.48%] | 8.7e-169 |
| 70% | 3000 | 9.86% | 2.76% [2.66%, 2.85%] | 2.19% | 1.50% [1.46%, 1.56%] | 45.4% [43.33%, 47.38%] | 8.0e-148 |
| 80% | 3000 | 9.42% | 1.92% [1.85%, 2.00%] | 1.62% | 1.03% [0.99%, 1.07%] | 46.5% [44.33%, 48.60%] | - |
| 90% | 3000 | 9.00% | 1.05% [1.00%, 1.10%] | 0.99% | 0.55% [0.53%, 0.57%] | 47.7% [45.38%, 49.92%] | - |
| 95% | 3000 | 8.83% | 0.61% [0.58%, 0.65%] | 0.66% | 0.33% [0.31%, 0.34%] | 46.7% [43.58%, 49.65%] | - |

Raw rows: `results/e1_cost_forecast.csv`. The tiny p-values reflect 3,000 paired projects on a generator where the methods differ systematically; they do not indicate a large practical effect on real projects.

### E2. Value of change-order data

**Manipulation:** With and without pending change-order features
**Measured:** Error reduction by change volume (per-project mean over the 30-70% checkpoints)

<!-- table:e2 -->
| Change volume | Projects | MAPE: CPI-based | MAPE: regression without exposure | MAPE: regression with exposure | Reduction from adding exposure [95% CI] | Holm p |
| --- | --- | --- | --- | --- | --- | --- |
| low | 1000 | 1.80% | 3.43% | 2.04% | 40.5% [38.85%, 42.17%] | 8.1e-119 |
| medium | 1000 | 3.56% | 2.27% | 2.06% | 9.2% [6.38%, 12.04%] | 4.2e-8 |
| high | 1000 | 7.52% | 4.03% | 3.08% | 23.7% [21.22%, 26.15%] | 4.9e-47 |

### E3. Schedule forecasting

**Manipulation:** SPI-based vs. earned-schedule vs. regression
**Measured:** Finish-date error (mean absolute error in days)

<!-- table:e3 -->
| Checkpoint (% complete) | Projects | Mean abs. error (days): planned finish | Traditional SPI | Earned schedule | Regression | Earned schedule vs traditional: reduction [95% CI] | Holm p (50-90% only) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 10% | 3000 | 39.8 | 42.1 | 17.1 | 15.8 | 59.4% [57.89%, 61.00%] | - |
| 20% | 3000 | 39.8 | 35.6 | 16.7 | 14.7 | 53.1% [51.22%, 54.80%] | - |
| 30% | 3000 | 39.8 | 30.7 | 15.5 | 13.4 | 49.4% [47.56%, 51.20%] | - |
| 40% | 3000 | 39.8 | 26.0 | 13.9 | 11.9 | 46.4% [44.40%, 48.23%] | - |
| 50% | 3000 | 39.8 | 21.1 | 12.2 | 10.4 | 42.2% [40.03%, 44.33%] | 2.2e-161 |
| 60% | 3000 | 39.8 | 16.2 | 10.6 | 9.1 | 34.3% [32.20%, 36.39%] | 4.3e-116 |
| 70% | 3000 | 39.8 | 11.2 | 8.9 | 7.8 | 20.2% [18.04%, 22.27%] | 2.5e-45 |
| 80% | 3000 | 39.8 | 9.8 | 7.0 | 6.5 | 28.1% [25.82%, 30.12%] | 2.7e-92 |
| 90% | 3000 | 39.8 | 16.2 | 5.2 | 5.1 | 67.7% [66.31%, 69.28%] | 3.7e-221 |
| 95% | 3000 | 39.8 | 24.2 | 4.5 | 4.4 | 81.6% [80.66%, 82.50%] | - |

Raw rows: `results/e3_finish_forecast.csv`.

### E4. Data-lag robustness

**Manipulation:** Inject reporting delays of 1 to 4 weeks (the forecast at the 50% checkpoint uses the figures reported that many weeks earlier; the regression models are trained on undelayed data)
**Measured:** Error growth

<!-- table:e4 -->
| Reporting delay (weeks) | Projects | MAPE: budget at completion | MAPE: CPI-based | MAPE: regression | Mean abs. finish error (days): earned schedule | Traditional SPI |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | 3000 | 10.74% | 4.30% | 2.37% | 12.2 | 21.1 |
| 1 | 3000 | 10.83% | 4.45% | 2.77% | 12.4 | 22.2 |
| 2 | 3000 | 10.94% | 4.64% | 3.37% | 12.8 | 23.2 |
| 3 | 3000 | 11.07% | 4.83% | 4.08% | 13.1 | 24.4 |
| 4 | 3000 | 11.18% | 5.03% | 4.82% | 13.4 | 25.5 |

<!-- table:e4ratio -->
| Method | Projects with all five delays | MAPE at 4 weeks / MAPE at 0 weeks [95% CI] | Slope (percentage points of MAPE per week) |
| --- | --- | --- | --- |
| CPI-based | 3000 | 1.17 [1.16, 1.18] | 0.18 |
| Regression | 3000 | 2.04 [1.97, 2.11] | 0.62 |

Raw rows: `results/e4_reporting_lag.csv`.

### Verdicts against the pre-stated criteria

<!-- table:verdicts -->
| Hypothesis | Verdict (synthetic data only) | Headline |
| --- | --- | --- |
| H1: regression cuts EAC error by at least 20% vs CPI at 30-70% complete | supported on this synthetic generator | reduction 43.5% to 45.4% across the five checkpoints |
| H2: pending change-order exposure cuts error materially in high-change projects | supported on this synthetic generator | high volume: 23.7% [21.22%, 26.15%] |
| H3: earned schedule beats the traditional SPI formula in the later half | supported on this synthetic generator | mean abs. finish error 5.2 to 12.2 days vs 9.8 to 21.1 days |
| H4: accuracy degrades gracefully with 1-4 weeks of reporting delay | not supported on this synthetic generator | MAPE ratio at 4 weeks: CPI-based 1.17, regression 2.04 (criterion: at most 1.5) |

## 8. Discussion

- **H1 and H2 are weaker evidence than the tables suggest.** The generator contains a systematic average overrun, a systematic slip and change orders that are approved with a fixed probability, so a regression trained on the same generator can learn those regularities and a formula that cannot (CPI-based) cannot. A 40%-48% error reduction here says the machinery works and can exploit structure when structure exists; it does not predict the size of any benefit on real projects, where data are noisier, regimes differ between projects and a model is trained on far fewer completed projects.
- **The exposure feature did two jobs in E2.** In low-change projects the regression *without* exposure was worse than the plain CPI formula (3.43% against 1.80%) because the model was trained on a mixture of volumes and pulled low-volume projects toward the average overrun; adding the exposure column, which also tells the model how change-heavy a project is, removed most of that bias (40.5% reduction). The high-volume effect (23.7%) is the cleaner reading of "value of pending change-order data".
- **Earned schedule against the traditional SPI.** The traditional formula's error shrinks to the 70-80% checkpoints and then grows again (9.8 days at 80%, 24.2 days at 95%) because the cost-based SPI drifts toward 1 as cumulative planned value flattens (worked example in [worked-examples](worked-examples.md), section 5). Earned schedule's error keeps falling. This is a property of the formulas, and the simulator reproduces it as expected.
- **Reporting delay (H4 not supported).** The regression, trained on current data, lost most of its advantage over the CPI-based formula after four weeks of delay (4.82% against 5.03%), and its error doubled. A model used with delayed data should be trained with delayed data; that was not tried. The CPI-based formula and the earned-schedule finish forecast degraded slowly.
- **Not evaluated:** the bottom-up method, commitment features, real data, other project types, forecast error at the 100% checkpoint (trivial), models other than ridge regression.

## 9. Threats to validity

- No real project data. Every result is conditional on the simulator's assumptions (section 4.2), several of which favour the regression by construction. The synthetic noise was not validated against published ranges; it is simply the author's choice.
- Projects in one replicate share a trained model, and the same projects appear at every checkpoint, so the checkpoint-level tests are not independent of each other. The Holm correction does not account for that dependence.
- The p-values come from a normal approximation to the Wilcoxon statistic; with 3,000 pairs this is adequate, but the extreme values (below 1e-100) are meaningless beyond "far below any threshold".
- The thresholds in the decision criteria (20%, 10%, 1.5 times) were chosen by the author. Different thresholds would change H2 (the medium-volume reduction is 9.2%) and H4.
- Training and test sets come from the same generator, so out-of-distribution behaviour (a different project type, a different change regime) was not tested. The protocol's leave-one-project-out validation was replaced by a split into training and held-out projects, which is simpler and with this much data equivalent in spirit.
- Percent complete is the share of the revised budget earned, taken at weekly resolution (the first week at or above the checkpoint).
- Researcher degrees of freedom: analysis choices were fixed before the reported run; see the change log for every disclosed deviation and rerun.

## 10. Reproducibility checklist

- [ ] Code tagged at the commit used for the reported results (no version control was initialised: nothing is committed or tagged)
- [x] Environment pinned: `package-lock.json` is committed and the run used Node v25.9.0 on win32-x64 (`results/run_metadata.json`); no container image was built
- [x] Random seeds and run counts recorded (`results/run_metadata.json`: base seed 20260101, 5 replicates, 300 training and 200 test projects per volume per replicate)
- [x] Raw per-project results are in `results/` as CSV next to the scripts that produced them (no notebook; the analysis is `experiments/summarize.ts`)
- [x] Hardware described: logical CPU count and platform in `results/run_metadata.json`; the machine was shared and loaded, which does not affect these deterministic results
- [x] One command reproduces every figure and table: `npm run measure -- --out=<dir>` (experiments only: `npm run experiments`); `scripts/render-tables.mjs` regenerates the markdown tables from the result files and `tests/docs.test.ts` checks that this document contains them
- [ ] Data sources, licenses and versions listed (there is no external data; the simulator is the only source, and its parameters are in the metadata)

## 11. References

- Fleming, Q. & Koppelman, J. (2016). Earned Value Project Management (4th ed.). Project Management Institute.
- Lipke, W. (2003). Schedule is Different. The Measurable News.

> These two citations are the original starting points and have **not** been verified against the originals.

## 12. Change log

| Date | Version | Change |
|---|---|---|
| 2026-11-10 | v0.1 | Protocol drafted; no results yet (blueprint stage). |
| 2026-11-10 | v1.0 | Protocol made specific before the reported run: sizes (5 replicates, 300 training and 200 test projects per volume), ten checkpoints (10% to 90% and 95%), decision criteria and thresholds, the traditional SPI formula added as the H3 comparator. Departures from the blueprint protocol: no real data (simulator only); bottom-up method and commitment features not evaluated; rank correlation not computed; train/test split instead of leave-one-project-out; Wilcoxon by normal approximation. |
| 2026-11-10 | v1.0 | Before the reported run: a tiny plumbing run (1 replicate, 60 training and 20 test projects per volume) was executed and inspected while building the harness; no simulator parameter or model setting was changed afterwards in response to its results. One simulator fix was made before the reported run: the weekly series now extends until every change order has been decided (previously a rejected change decided after the finish could leave pending exposure at the end of the project). |
| 2026-11-10 | v1.0 | The first full run is kept in `results/superseded-run-1/`. Its Wilcoxon p-values printed as 0 because the normal tail was computed as 1 minus a CDF with an absolute-error approximation. The tail function was replaced with a relative-accuracy complementary error function (`erfc` in `src/stats/index.ts`, tested against known tail values) and the full run was repeated on a fresh clean copy. The three CSV files are byte-identical between the two runs (same SHA-256); only `summary.json` differs, in the p-values. The tables above come from the second run. |
| 2026-11-10 | v1.0 | The convergence property test of the offline merge found a real bug in the product while this study was being prepared (unrelated to the experiments; recorded in the quality report). |

---

[README](../README.md) · [Client case study](case-study.md) · [Research report](research-report.md) · [Quality report](quality-report.md)
