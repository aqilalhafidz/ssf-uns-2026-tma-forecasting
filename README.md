# Water Level Forecasting for the Bengawan Solo River Basin - SSF UNS 2026

[Bahasa Indonesia](README_id.md)

**Summary.** Forecasting water levels (`tma_mdpl`, metres above sea level) at 30 river gauges in the Bengawan Solo basin, Indonesia, for the SSF UNS 2026 competition (private Kaggle competition; metric: RMSE; 8-month horizon; 3 observations per day). The final approach is LightGBM on weather, soil-moisture and rainfall-memory features, with the target normalised by each gauge's mean, label spikes removed, and predictions averaged over models trained with different cut-off dates and seeds. The repository documents what worked, what did not, and the evidence for each decision, using a validation scheme built to mimic the real train-to-test gap.

---

## 1. Problem

- **Goal:** predict `tma_mdpl` at 30 monitoring stations operated by BBWS Bengawan Solo, at 06:00, 12:00 and 18:00.
- **Training data:** 2023-01-01 to 2025-09-18. **Test data:** 2025-09-19 to 2026-05-18 (8 months, 21,780 rows, no water-level history provided).
- **Supporting data:** hourly environmental variables (rainfall, humidity, soil moisture, pressure, MJO, ENSO, among others).
- **Metric:** RMSE, which penalises large errors heavily.
- **Competition:** SSF UNS 2026, an invitation-only Kaggle competition. The data were provided by the organizers and are not redistributed here.

## 2. Results

| Stage | Private RMSE |
|---|---|
| Initial baseline | 1.815 |
| Data leakage removed, label spikes handled | 1.685 |
| Target normalised by per-gauge mean + rainfall accumulation features | 1.676 |
| Average of two models (different cut-off dates) | 1.672 |
| Average of four cut-offs x 5 seeds | 1.662 |
| Average of two final versions | **1.661** (public 1.601) |

- The original in-competition entry scored 1.676 private and ranked 47th of 95 teams. Submissions made after the deadline do not count towards the official ranking.
- For context, the leading private score on the leaderboard was about 1.42. The reasons for this gap are not identified (see Section 6).

## 3. Approach

1. **Data checks and cleaning:** automatic checks for duplicates, gaps, missing-value codes (for example `-999` in the solar radiation column), and values outside the training range; removal of label spikes using a 5-observation median rule (deviation > 30 m, or sudden zero/negative values).
2. **Target normalisation:** subtract each gauge's mean (computed from training data only), so one pooled model can learn from 30 gauges whose levels differ greatly.
3. **Features:** rainfall accumulations over several windows (2 to 90 days), long-term soil moisture, basin-average rainfall, antecedent precipitation indices (exponentially weighted rainfall, half-lives of 12 to 1,440 hours), calendar features, observation hour and astronomical tide harmonics.
4. **Model:** LightGBM with default parameters, 300 trees.
5. **Ensemble:** average of models trained with different cut-off dates (2024-01-18, 2024-09-19, 2025-01-18, full training set), each averaged over 5 seeds with bagging.
6. **Validation:** three forward-chaining folds, each evaluated on the 8 months after its cut-off (mimicking the real train-to-test gap), fixed number of trees (no early stopping on validation data), and RMSE reported separately for the rainy season.

## 4. Key decisions and evidence

All decisions are based on forward-chaining validation, not on the leaderboard. Negative results are recorded on purpose.

| Decision | Evidence |
|---|---|
| Target minus per-gauge mean | Fold 1 RMSE 2.85 -> 0.77, fold 2 1.62 -> 1.00 (the raw-target model was fragile) |
| Long-window rainfall sums with full windows only | Partial windows understated early-2023 rainfall; after the fix 0.814 -> 0.771 |
| Basin-average rainfall features | 0.850 -> 0.770 |
| Antecedent precipitation index (API) | 0.770 -> 0.745; Wonogiri Dam RMSE 1.54 -> 1.22 (strong in validation, neutral on the leaderboard) |
| Drop `solar_radiation_mj_m2` | Contains the code `-999` from 2025-12-31 onwards (57% of test rows) |
| Drop `nino_34` and MJO features | Better without them in all three folds once the features were cleaned |
| Average over 5 seeds (bagging) | RMSE 0.7698 -> 0.7624, consistent across folds and visible on the leaderboard |
| Average over cut-off dates | 2025 window: 0.7556 -> 0.7104; leaderboard improved by 0.0085 RMSE (the largest gain that transferred) |
| Observation hour and tide harmonics | One gauge (Arjowinangun) is clearly tidal; 0.7454 -> 0.7405 (small but consistent) |
| Default model parameters | Optuna tuning did not beat the defaults across 3 seeds (0.8419 vs 0.8372) |

**Tried and rejected:** features from the HydroRIVERS river network (worse than basin-average rainfall), 365-day target lag (harmful), CatBoost and per-gauge linear models (much weaker), blend with per-gauge models (+0.009 in validation, no change on the leaderboard), block bagging, shifted training windows (weak members dilute the average), median across models, clipping, linear calibration, temporal smoothing.

## 5. Lessons learned

1. **Check the test-period data first.** A missing-value code in one column was found only midway through the project, through an extrapolation diagnostic.
2. **Validation should mimic the train-to-test gap and season**, and its windows should not overlap when used for calibration.
3. **A large gap between validation and leaderboard is information.** Large validation gains from features (basin rainfall, API) barely appeared on the leaderboard, while variance reduction (averaging seeds and cut-offs) did.
4. **Averaging two equally accurate models reduces MSE by a quarter of the squared distance between their predictions.** This matched the leaderboard (0.0065 / 0.0057 observed vs 0.0059 predicted) and gives an upper bound that can be computed without labels.
5. **Record every decision with its evidence,** including the negative ones.

## 6. Limitations and open questions

- Almost all of the leaderboard MSE (about 2.56) is error shared by every model variant in this repository. Its source is not identified. Untested hypotheses: year-to-year regime differences at gauges such as Wonogiri Dam, and level shifts at Kali Anyar.
- The gap to the leading score (about 0.24 RMSE) is not explained by anything tried here.
- Differences below roughly 0.004 RMSE cannot be distinguished from leaderboard noise.

## 7. How to run

1. Obtain the competition data from the organizers (not included in this repository): `train.csv`, `test.csv`, `sample_submission.csv`, `data_pendukung/data_lingkungan.csv`.
2. Put them in one folder (for example on Google Drive).
3. Open `ssf_uns_2026_tma_final.ipynb` in Google Colab (Google Drive is mounted by the first cell), set `DATA_DIR` in the configuration cell, and run all cells. Set `RUN_CUTOFF_CV = False` to skip the slow cut-off averaging validation.
4. Output: `submission_final.csv` in `OUTPUT_DIR`.

Main dependencies: `pandas`, `numpy`, `scikit-learn`, `lightgbm` (see `requirements.txt`).

## 8. Repository structure

```
.
├── README.md
├── README_id.md
├── requirements.txt
├── .gitignore
├── LICENSE
└── ssf_uns_2026_tma_final.ipynb
```

## 9. Reproducibility notes

Seeds are fixed. Re-runs can differ very slightly (LightGBM thread ordering) without changing any conclusion.

## 10. License and data

Code is released under the MIT License (see `LICENSE`). The competition data are subject to the organizers' terms and are not redistributed.
