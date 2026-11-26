# Decisions

One line per methodological decision: what you chose, why, and what it changed.
Agents append here via the `/new-feature` skill; you append here by hand.

- Exclude `casual`, `registered`: they sum to `cnt` exactly. Leaky model R^2 1.000 -> honest ridge 0.583.
- Exclude `cnt_2d_bfr`: a shipped 2-day lag we cannot test the alignment of. Build our own lags instead.
- Exclude `instant`, `yr`, `days_since_2011`: time indexes let a tree predict a flat constant past 2011 and a linear model extrapolate a trend we have no grounds for.
- Split 2011 / H1 2012 / H2 2012 by date. Test set touched once, after all feature work.
- Report mean and persistence baselines with every score. Persistence (test R^2 0.510, MAE 840) is within 0.07 R^2 of ridge and beats it on MAE; that caveat goes in the report.
- 2026-11-25: Feature cnt_wow_change = cnt_lag_1 - cnt_lag_7 (uses only d-1 and d-7). Validation R2: ridge 0.6475 -> 0.6474, hgb 0.2185 -> 0.198. Kept in the pipeline; adds nothing for ridge and hurts hgb.
- 2026-11-25: The agent reported "before" as 0.6474/0.198; the real values (reports/metrics_before.json) are 0.6475/0.2185. Used the file.
- 2026-11-25: Rejected the agent's first contract test (it auto-classified any numeric column). Every raw column is now listed explicitly in src/config.py.
- 2026-11-25: Added a test that encode_weather raises on unknown labels; before it, removing the raise turned no test red.
- 2026-11-25: Switched from local qwen3.5:9b to qwen3.5:27b on a Brev A6000 (local turns took 2-6 minutes).
- 2026-11-25: Exercise 4: proposed predicting cnt / cnt_roll_7; defensible; not run before the deadline; committed metrics are untransformed.
- 2026-11-25: Exercise 5: would not deploy ridge over persistence (loses test MAE 947.0 vs 840.1 for +0.07 R2).
- 2026-11-25: CLAUDE.md: added the /seed-fault line.
