# Results

Raw outputs of the measurement scripts. Nothing in this folder was typed in by hand. All experiment data are SYNTHETIC (see `docs/research-report.md`). The documents in `docs/` quote these files, and `tests/docs.test.ts` checks that their tables match.

| File | Written by | What it holds |
|---|---|---|
| `e1_cost_forecast.csv` | `npm run experiments` (inside `npm run measure`) | one row per synthetic test project and checkpoint: absolute percentage error of four cost forecasts |
| `e3_finish_forecast.csv` | same | one row per project and checkpoint: absolute finish-date error in days for four methods |
| `e4_reporting_lag.csv` | same | one row per project and reporting delay (0 to 4 weeks) at the 50% checkpoint |
| `summary.json` | same | statistics, intervals, p-values, the decision criteria and the verdicts |
| `run_metadata.json` | same | seeds, sizes, simulator parameters, runtime, SHA-256 of the output files |
| `contrast.json` | `npm run check:contrast` | WCAG contrast ratios computed from the colour tokens |
| `licenses.json` | `npm run check:licenses` | licences of every production dependency |
| `web-weight.json` | `npm run check:weight` | exact bytes of the static files |
| `a11y-axe.json` | `scripts/browser-checks.mjs` | axe-core scan results (a partial, automated check) |
| `first-load.json` | `scripts/browser-checks.mjs` | requests and body bytes of the first load |
| `path-scan.json` | `npm run check:paths` | scan for local paths, user names and personal names |
| `measurement_run.json` | `npm run measure` | the steps of the one-shot run, exit codes and file hashes |
| `test-run.json` | `npm run test:record` | type-check result and test totals of the final run (no timings) |
| `superseded-run-1/`, `superseded-run-2/`, `superseded-run-3/` | earlier runs of `npm run measure` (and, for run 2, `test-run.json`) | kept for disclosure; see the change log in `docs/quality-report.md` |

No file here reports timing as performance; the machine was shared and loaded.
