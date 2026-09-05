# ICU Anomaly Detection (MIMIC-IV)

Time-series anomaly detection pipeline on ICU vitals from the MIMIC-IV clinical database.
Faculty-supervised research project,using PhysioNet-credentialed access to MIMIC-IV.

## Overview

The pipeline extracts multivariate ICU vitals from MIMIC-IV's `chartevents` table, builds a
uniform time-series representation per ICU stay, windows it into fixed-length segments(of 20 minute interval gaps), extracts
statistical/time-series features per window via `tsfresh`, and scores each window for anomalous nature (between -1 and 1, with -1 being most anomalous) using Isolation Forest.

## Dataset and charts used:

- Source: MIMIC-IV (`chartevents.csv.gz`, `icustays.csv.gz`, `patients.csv.gz`, `admissions.csv.gz`,
  `d_items.csv.gz`)
- Access requires PhysioNet credentialing (not included in this repo)
- Raw extraction performed with DuckDB and its supported SQL based querying format (`read_csv_auto`) to avoid loading multi-GB files into memory

## Vitals used

| Vital | itemid | Notes |
|---|---|---|
| Heart Rate | 220045 | Continuously monitored, near-complete coverage |
| SpO2 | 220277 | Continuously monitored, near-complete coverage |
| Respiratory Rate | 220210 | Corrected from an earlier itemid (224688, ventilator-set rate, not patient rate) — fix raised window completeness comparitively |
| Temperature (°C) | 223762 + 223761 | Coalesced from Celsius and Fahrenheit sources (`combine_first`, F→C converted) — roughly halved missingness vs. Celsius alone |
| NBP Systolic / Diastolic / Mean | 220179 / 220180 / 220181 | Bimodal completeness (~64%) — present in full or not charted at all per window, consistent with spot-check cuffs |
| ART Mean | 225312 | Kept despite high sparsity (~96% missing); ARTsys/ARTdia dropped entirely for the same reason |

Glucose (itemid 220621) and GCS were investigated and excluded — both were meaningfully sparser
than the core vitals, and completing GCS would have required summing three separate components
(Eye/Verbal/Motor), adding pipeline complexity for limited coverage gain. Urine output (Foley,
itemid 226559) was identified as a strong candidate but lives in `outputevents`, not `chartevents`,
and was deferred.

## Pipeline stages

1. **Extraction** — DuckDB query per itemid group against raw `chartevents.csv.gz`
2. **Timestamp validation** — merge with `icustays`, flag/clean chart times far outside a stay's
   `intime`/`outtime` (thresholds: 60hr tolerance window, 1-day max deviation) and only those entries with credible amount of data (more than 4hrs is taken into account)
3. **Resampling** — uniform 20-minute grid per `stay_id` (chosen after observing real measurement
   intervals of ~55min–1h20min across vitals), with `ffill(limit=4)` (max 80-minute carry-forward)
4. **Checkpoint** — saved as `uniform_20min.parquet` 
5. **Windowing** — fixed 2-hour windows (`window_size=6`) via
   `groupby("stay_id").cumcount() // 6`
6. **Completeness filtering** — windows kept only where Heart Rate, SpO2, and Respiratory Rate
   completeness are each ≥ 0.8
7. **Feature extraction** — `tsfresh.extract_features` (long-format melt of vitals), using
   `EfficientFCParameters` with `binned_entropy` removed (incompatible with short/flat 6-point
   windows), run in batches of 10,000 windows with a shrinking-pool sampling scheme for
   crash-recoverable, non-overlapping batches
8. **Anomaly scoring** — batches combined into a single feature matrix, imputed
   (`tsfresh.utilities.dataframe_functions.impute`), scaled (`StandardScaler`), then scored with a
   single `IsolationForest` fit on the full combined set (per-batch fits were validation only, not
   the final model)

## Validation

Flagged top-anomaly windows were manually spot-checked against raw vitals across multiple batches
and consistently showed coordinated multivariate deviations (e.g. simultaneous HR/RR/temp/BP
spikes), not single noisy readings — supporting that the pipeline is capturing physiologically
meaningful anomalies rather than artifacts.

## Status / next steps

- Preprocessing, windowing, and per-batch validation: done
- Remaining tsfresh batches (toward ~90–100k validation windows): in progress
- Full combined-matrix Isolation Forest fit: pending completion of all batches
- Hyperparameter analysis: planned focus on `contamination` (score-to-label threshold, a modeling
  assumption about expected anomaly prevalence) and `max_samples` (subsample size per tree, affects
  isolation-path-length contrast between anomalous and normal points)
- Frontend/serving layer: precompute-once, serve-cheap design — heavy pipeline runs offline and
  produces a static scored results table plus saved `scaler`/`IsolationForest` artifacts; the app
  only reads and visualizes precomputed results (per-stay vitals + anomaly overlay), with live
  model inference reserved for scoring new incoming windows only

## Notes on known pitfalls (for future reference)

- Two itemids initially picked from `d_items` were wrong in ways that only showed up as
  suspiciously high missingness, not errors: Respiratory Rate and Temperature. Both were fixed by
  checking `d_items` completeness directly against `chartevents` row counts rather than trusting
  label text alone.
- `tsfresh`'s `binned_entropy` calculator fails on short, low-variance windows (`ValueError: Too
  many bins for data range`) — removed from `EfficientFCParameters` before extraction.
- `pivot()` fails on duplicate index/column combinations; `pivot_table(aggfunc=...)` handles it.
- Boolean multi-column filtering needs `.all(axis=1)` to collapse per-column comparisons into a
  single per-row mask.
