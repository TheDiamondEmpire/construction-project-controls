# Quality report: Construction Project Controls Platform

> **Where the code is.** The measurement scripts that produced these results are part of the source package, which is distributed separately (see [SOURCE-ACCESS.md](../SOURCE-ACCESS.md)). The raw results they wrote are included in this repository under [`results/`](../results/).

This report states what was measured, the command that produced each number, what the numbers are, and what was **not** tested. Every figure in the tables comes from a file in [results/](../results/) written by a script; nothing here was typed in from memory, except the test totals in section 3, which are copied by hand from `results/test-run.json` (a test cannot check the totals of the run it is part of). The product is a **local prototype** by The Diamond Empire: it runs on one machine, was built and tested on synthetic data only, and has never been used on a real project. Its outputs are planning aids, not engineering, legal or safety advice, and nothing here is a claim about real construction outcomes. No timing is reported as performance: the machine was shared and loaded.

## 1. Scope and environment

| | |
|---|---|
| Product version | 0.1.0 |
| Runtime used | Node.js v25.9.0, npm 11, Windows 11, one Chromium-based browser (version 157.0.8103.39, driven with playwright-core 1.56.1) |
| Database used | SQLite through `node:sqlite`. **PostgreSQL was not used or tested.** |
| Forecasting code | TypeScript (not the blueprint's Python); Python was not installed and was not used |
| Verification copy | the source was copied (without `node_modules`, `.tmp` and `data`) to a short folder directly under the drive root, installed with `npm ci`, type-checked, measured and tested there; every output was written outside that copy, then placed in `results/` |
| Not run | the GitHub Actions workflow `.github/workflows/ci.yml` (written to run the same commands; **never executed**), Docker, Terraform |

## 2. Measurements

One command, run once after the code was frozen, in one uninterrupted invocation: `npm run measure -- --out=<dir>` (`scripts/measure-all.mjs`; the steps and their exit codes are in `results/measurement_run.json`). It runs the path scan, the production licence check, the colour-contrast computation, the static asset weights, the browser checks (axe-core scans and first-load requests) and the forecasting experiments.

<!-- table:measures -->
| Quantity | Value | Source file | Command |
| --- | --- | --- | --- |
| Colour pairs checked / below their minimum | 26 / 0 | `results/contrast.json` | `npm run check:contrast` |
| Tightest pair (ratio against its minimum) | --field-border on --surface: 3.65 against 3 | `results/contrast.json` | `npm run check:contrast` |
| Production packages (transitive) / outside the licence allow-list | 49 / 0 | `results/licenses.json` | `npm run check:licenses` |
| Static files served, total bytes (gzip) | 13 files, 82667 (29177) | `results/web-weight.json` | `npm run check:weight` |
| First load of the signed-out screen: requests, response body bytes, all same-origin | 11, 81003, true | `results/first-load.json` | `npm run measure` |
| axe-core scans / violation instances | 22 / 0 | `results/a11y-axe.json` | `npm run measure` |
| Text files scanned / path, user-name or personal-name findings | 119 / 0 | `results/path-scan.json` | `npm run check:paths` |
| Steps of the one-shot run that exited 0 | 6 of 6 | `results/measurement_run.json` | `npm run measure` |

How to read these:

- **Contrast.** Computed by `scripts/contrast.mjs` from the hex tokens in `web/styles.css` for 26 foreground/background pairs the interface uses (4.5:1 minimum for text, 3:1 for borders, focus and chart marks). It covers colour pairs of the light theme only; there is no dark theme. It says nothing about the rest of accessibility.
- **Accessibility scan.** axe-core 4.11.1 with the tags `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa` and `best-practice` on 11 screens (the signed-out login screen and ten screens as a project manager) at 1280x800 and 375x812, on synthetic data: 22 scans, 0 violations. **Automated rules find only part of the possible problems; this is not a claim that the interface is accessible to any WCAG level.** Manual keyboard-only and screen-reader testing was not done. Other roles, error states, offline states and the open detail panels and forms were not scanned. (The scan injects axe with the browser's CSP bypass switched on, only so the script can be loaded; the application's own CSP is tested separately in `tests/ui.test.ts`.)
- **Page weight and requests.** Exact bytes of the static files and of the response bodies of the first uncached load. The first load makes 11 requests, all to the same origin; no fonts, images, scripts or styles come from elsewhere. These are byte and request counts, not speed.
- **Licences.** All 49 production packages (fastify and its transitive dependencies) declare MIT, ISC or BSD-3-Clause. axe-core (MPL-2.0) is a development tool and is not shipped. See `THIRD-PARTY.md`, which a test compares with `package.json` and the installed packages.
- **Path scan.** No drive path, home-directory path, operating-system user name or personal name is in any text file, with one disclosed exception: the pre-existing banner and social-preview SVGs and the README footer, which the branding rules say to leave unchanged, still carry the original personal handle (`legacyUnchangedOnPurpose` in `results/path-scan.json`). The scan ran at the code freeze; the documents were written afterwards, and `tests/hygiene.test.ts` re-scans the final tree in every test run.

### Experiments E1 to E4 (synthetic projects only)

The forecasting experiments are described, with every table, in the [research report](research-report.md). Headline verdicts against the criteria written before the run (all on a simulator the author built; they do not transfer to real projects):

<!-- table:verdicts -->
| Hypothesis | Verdict (synthetic data only) | Headline |
| --- | --- | --- |
| H1: regression cuts EAC error by at least 20% vs CPI at 30-70% complete | supported on this synthetic generator | reduction 43.5% to 45.4% across the five checkpoints |
| H2: pending change-order exposure cuts error materially in high-change projects | supported on this synthetic generator | high volume: 23.7% [21.22%, 26.15%] |
| H3: earned schedule beats the traditional SPI formula in the later half | supported on this synthetic generator | mean abs. finish error 5.2 to 12.2 days vs 9.8 to 21.1 days |
| H4: accuracy degrades gracefully with 1-4 weeks of reporting delay | not supported on this synthetic generator | MAPE ratio at 4 weeks: CPI-based 1.17, regression 2.04 (criterion: at most 1.5) |

## 3. Test suite

Command: `npm run test:record -- --out=<dir>` (runs `tsc --noEmit` and `npm test`; writes `test-run.json`). Final run, from the clean short-path copy after `npm ci`:

- type check: `tsc --noEmit` exited 0 with no errors
- tests: **151 tests, 151 passed, 0 failed, 0 skipped, 0 cancelled** (`results/test-run.json`; the 20 test files include property tests, API tests, importer tests and one real-browser test file, whose result was checked by running it 10 times in a row, see the change log)

What the tests cover (files in `tests/`):

| File | What it checks |
|---|---|
| `money-dates`, `costs`, `payapp` | integer-cent arithmetic and rounding; cost roll-up (hand-computed example, plus property tests that every parent equals the sum of its children on random trees); pay-application identities (hand-computed example, plus property tests) |
| `cpm`, `evm` | critical path with FS/SS/FF/SF and lags (hand-computed networks; property tests against a path-enumeration oracle; constraints satisfied on random mixed networks); EVM formulas, earned schedule and ridge regression (hand-computed examples; inversion property) |
| `workflow`, `api-workflow` | RFI, submittal and change-order state machines; ball-in-court; aging buckets and median response; change orders as forecast exposure and their approval posting to budget, commitment and time; pay applications end to end |
| `importers` | XER and Microsoft Project XML import on hand-written samples, rounding warnings, malformed input, logic loops, DTD and entity refusal |
| `sync`, `api-field` | offline merge convergence (property test over random operations, orders and duplicates), validation, clock-skew correction, photos, permissions, replay equivalence |
| `api-access`, `api-cost` | authentication, session expiry, role and company boundaries, cross-project isolation, input validation, CSV imports, forecast overrides, HTTP hardening |
| `api-schedule` | schedule import via the API, critical-path analysis of the seeded schedule, forecast at the seeded status date, progress, planned-value points, regression calibration |
| `stats`, `experiments` | random-number moments, Wilcoxon (hand-computed z), Holm, bootstrap; simulator invariants and noise-free exact cases; reproducibility and train/test separation of the experiment harness |
| `ui` | one real browser: sign-in, role-limited navigation, no off-origin requests and no CSP violations, text-only rendering of hostile titles, offline editing with queued photo and manpower then sync, start-up with the server down, no horizontal scroll at phone width |
| `startup`, `scripts`, `hygiene` | `npm start` on an empty directory; contrast, licence and tail-probability helpers; path scan, THIRD-PARTY match, README structure, links, no external requests |
| `docs` | the tables in this report and the research report are exactly the tables rendered from `results/` |

## 4. What was not tested or not built

- Real data of any kind: no real P6 or Microsoft Project exports (the importer was only tested on files the author wrote following the format layout, so **fidelity on real exports is unknown**), no real project cost data, no real field use.
- PostgreSQL, S3-compatible storage, Docker, Terraform, a reverse proxy or TLS. A local directory and SQLite stand in.
- Other browsers than the one Chromium-based browser, real phones, installing the PWA on a device, the browser's offline emulation on a mobile network, service-worker update behaviour.
- Load, concurrency beyond the unit and API tests, long-running stability, backup and restore.
- Security: no penetration test; the gaps are listed in `SECURITY.md`.
- The bottom-up forecast and commitment features were not part of the experiments (only the earned-value, progress and change-order features were).
- The CI workflow has never run.

## 5. Limits and threats to validity

- The experiments use a simulator with assumed parameters, several of which favour the regression by construction (a systematic average overrun and slip, change orders approved with fixed probability). Effects are conditional on that model.
- Checkpoint-level tests reuse the same projects, so they are not independent of each other and the Holm correction does not account for that.
- The device clock-skew correction assumes negligible network latency and trusts the send time that a device reports.
- Change-order weights (25%, 50%, 75%) are illustrative defaults, not calibrated to data.
- The schedule engine reduces calendars to a five-day week without holidays, rounds durations up to whole days, and ignores constraints, resources and levels of effort; the planned-value curve comes from a profile or points, not from a cost-loaded schedule.
- One machine, one operating system, one browser engine.

## 6. Reproducibility checklist

- [x] Dependencies pinned by `package-lock.json`; installs from a clean state with `npm ci`
- [x] Every measured number is produced by a script and written to `results/`; one command (`npm run measure`) reproduces them, and `scripts/render-tables.mjs` regenerates the tables in the documents
- [x] Random seeds, run counts and simulator parameters recorded in `results/run_metadata.json`
- [x] Documents checked against results by a test (`tests/docs.test.ts`)
- [x] Outputs of the verification run were written outside the copy
- [ ] Container image digest (no container was built)
- [ ] Code tagged at the commit used (no version control was initialised)
- [ ] CI run (the workflow was never executed)

## 7. Change log (reruns, post-hoc changes and defects found)

| Item | What happened |
|---|---|
| Flaky browser test, real client race fixed | A reviewer's independent clean copy (fresh `npm ci`, loaded machine) failed the browser test "daily log: edit online, go offline ..." (149 of 150 passed; it failed in the full run and in the first isolated rerun, then passed). **My earlier "150 of 150" was therefore not reproduced on a second clean copy.** I reproduced a failure of the same test (and four tests after it) on my own machine by pinning the whole suite to 2 cores with 2 busy loops. Diagnosis from reproduction, not from guessing: (1) `sync()` in `web/offline.js` returned the already running sync when called again, so an edit queued after the running sync had read the queue was not sent until the next 30-second timer or the next edit. A deterministic regression test (a slow `/sync` response; edit during it) failed on the old code. No edit was lost, but it could be delayed by up to 30 seconds. The running sync now makes one more pass when it was asked again. (2) In `web/views-field.js`, status-line refreshes and redraws started by different events could finish out of order, so an older read could overwrite a newer pending count or log view with stale data. Only the most recently started refresh or redraw may now write to the page, and a background sync no longer redraws the form while the user is typing in it (typed text could be replaced before its change event fired). The test now waits for events with a diagnostic message (status line text and queued operations) and uses 30-second bounds instead of 8. The failure that the pinned run produced on the unfixed code did not recur in three further pinned full runs (3 of 3 passed after the fix). |
| Reruns after the fix | Files under `web/` changed, so everything that depends on them was rerun from a fresh clean copy: the UI test file ran 10 times in a row (10 of 10 passed, 8 tests each), then the one-shot `npm run measure` was repeated (6 of 6 steps exited 0) and the full suite recorded again (see section 3). The experiment CSV files and `summary.json` are byte-identical to the earlier runs; web weight, first-load bytes and the file count changed as expected. Earlier measurement runs are kept for disclosure: `results/superseded-run-1/` (before the p-value tail fix), `results/superseded-run-2/` (the run behind the earlier "150 of 150", including its `test-run.json`) and `results/superseded-run-3/` (after the sync fix but before the status and redraw fix). |
| Measurement rerun | The first full measurement run is kept in `results/superseded-run-1/` (its `summary.json`, `measurement_run.json` and `run_metadata.json`). Its Wilcoxon p-values printed as 0 because the tail probability was computed as 1 minus a CDF with an absolute-error approximation. The tail function was replaced with a relative-accuracy complementary error function and the whole measurement run was repeated on a fresh clean copy. The experiment CSV files are byte-identical between the two runs; only the p-values in `summary.json` differ. All tables here come from the second run. |
| Defect found by a property test | The convergence test of the offline merge failed on its first run: a delete that arrived before the entry it deleted left a different hidden value from the same operations arriving in the other order. Tombstones now always hold no value. |
| Defect found by a smoke run | Critical-path analysis against a required finish later than the computed finish reported zero critical activities, because "critical" was defined as zero float to the required finish. It is now defined against the computed finish and the float to the required finish is reported separately. |
| Hand arithmetic corrected | The first hand calculation of the seeded cost report missed that committed cost on cost code 26-200 exceeds its budget; the test showed the discrepancy and the worked example (section 7 of [worked-examples](worked-examples.md)) was corrected. The product code was right. |
| Defects found by the browser tests | The signed-in screen threw an error when the hash changed while the login screen was showing (the route handler ran without a page); the daily-log status line did not refresh after the connection came back. Both fixed. |
| Defect found by a unit test | The demo seed gave every demo user the same password hash (same salt); each user now has its own salt. |
| Accessibility finding fixed | A preliminary axe run reported 8 instances of `empty-table-header` (the action columns of four tables had empty header cells); the headers are now labelled "Actions". The reported run found 0 violations. |
| Simulator fix before the reported run | The weekly series now extends until every change order is decided. A tiny plumbing run of the experiments was inspected while building the harness; no parameter was changed in response. |
| Design change | An optional "required finish" argument was removed from the critical-path function; float to the required finish is now an offset applied in the risk flags. |
| Branding assets | The banner and social-preview SVGs and the README footer were left unchanged as instructed; they still show the original personal handle. Documents written for this product name only The Diamond Empire. |
