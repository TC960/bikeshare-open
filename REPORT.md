# Bikeshare Demand Forecasts — Results Report

## Results

| Model | Feature | Valid R² | Valid MAE | Test R² | Test MAE |
|-------|---------|----------|----------|---------|---------|
| baseline_mean | before | -0.9318 | 2002.2 | -2.1441 | 2847.7 |
| baseline_mean | after  | -0.9318 | 2002.2 | -2.1441 | 2847.7 |
| baseline_persistence | before | 0.4507 | 900.6 | 0.5099 | 840.1 |
| baseline_persistence | after  | 0.4507 | 900.6 | 0.5099 | 840.1 |
| ridge | before | 0.6475 | 832.9 | 0.5833 | 947.1 |
| ridge | after  | 0.6474 | 833.1 | 0.5833 | 947.0 |
| hgb | before | 0.2185 | 1296.6 | -0.4975 | 1972.6 |
| hgb | after  | 0.198 | 1312.5 | -0.4949 | 1966.9 |

## Headline finding

Evaluated naively, a model predicting cnt scores R² = 1.000 (python -m src.leakage_demo), but that result is invalid: casual + registered equals cnt on every row, so the model learns addition, not demand. Under a temporal split (train 2011, test H2 2012), predicting the 2011 mean scores R² = −2.1441 on test, because ridership grew. Ridge reaches 0.5833 on test against a persistence baseline of 0.5099, and loses to persistence on MAE (947.0 vs 840.1). Gradient boosting scores −0.4949: it still beats the mean baseline, but a tree cannot predict counts above the largest it saw in 2011. My feature cnt_wow_change left ridge's validation R² unchanged (0.6475 → 0.6474) and lowered hgb's (0.2185 → 0.198).

I kept it in the pipeline, and I'm reporting that it adds nothing for ridge and makes hgb worse (−0.02 validation R²).

## Exercise 4: target transformation

I didn't run this before the deadline. I'd train hgb on cnt / cnt_roll_7 (the ratio to the past 7-day average) and multiply predictions back. That removes the growth trend, so the tree no longer has to predict above the 2011 maximum. I think it's defensible, not just a way to win: it uses only past data and targets the real problem, a level shift. The test set was not checked.

## Exercise 5: ridge vs persistence

I wouldn't deploy ridge over persistence. On the newest data (test), ridge is further off on a typical day (MAE 947.0 vs 840.1 rides), and its R² gain is only 0.07 (0.5833 vs 0.5099). It avoids some large misses, but a model that loses on typical-day error to 'yesterday again' isn't worth maintaining.

## Attribution

Model: qwen3.5:27b via Ollama (Q4_K_M, 64k context) on a rented NVIDIA Brev RTX A6000. I started with qwen3.5:9b locally, but turns took 2–6 minutes, so I switched before any graded work.

Agent-generated: cnt_wow_change and its test; test_all_raw_columns_classified and the column sets in src/config.py; test_encode_weather_raises_on_unknown_label; the seed-fault skill; the fault seeding and fixes in docs/SEEDED_FAULTS.md; drafts of REPORT.md, docs/AUDIT.md and DECISIONS.md lines, which I edited.

My decisions (each in docs/DECISIONS.md): trying cnt_wow_change and keeping it; rejecting the first contract test; classifying every raw column explicitly; using the real "before" numbers instead of the agent's; switching to the hosted model; my Exercise 4 proposal and Exercise 5 verdict.

Other help: Claude (Opus 5.5) for environment setup, debugging, suggested prompts, and draft wording.
