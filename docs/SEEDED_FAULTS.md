# Seeded Faults

Record of deliberately seeded faults, their detection by hooks/tests, and fixes.

Each entry includes:
- **Fault**: Description of the deliberate bug introduced
- **Diff**: The exact git diff of the faulty change
- **Hook stderr**: The PostToolUse hook's error output (verbatim)
- **Test output**: The red `make test` output (verbatim)
- **Fix turns**: Number of turns taken to restore (filled in after fix)

---

## Fault 1: Add "casual" to feature_columns return value

**Fault**: Added `"casual"` to the list returned by `feature_columns()` in `src/features.py`, violating the FORBIDDEN column contract. This is target leakage since `casual + registered == cnt` exactly.

**Diff**:
```diff
diff --git a/src/features.py b/src/features.py
index 03c8803..b638d56 100644
--- a/src/features.py
+++ b/src/features.py
@@ -85,7 +85,7 @@ def build_features(df: pd.DataFrame) -> list[str]:
     """Numeric columns that are safe to feed the model."""
     numeric = df.select_dtypes(include="number").columns
-    return [c for c in numeric if c not in FORBIDDEN]
+    return [c for c in numeric if c not in FORBIDDEN] + ["casual"]


```

**Hook stderr** (PostToolUse):
```
Leakage tests failed after editing /Users/mohak/Desktop/dsc-capstone/DSC180A-Methodology-6/bikeshare-open/src/features.py:
2   2011-0...9  ...          1346.0
712 2012-12-30  ...          -446.0
713 2012-12-31  ...           876.0

[714 rows x 29 columns]

    def test_no_forbidden_features(feats):
        """Guards the gate, not the data: feature_columns() excludes FORBIDDEN by
        construction, so this cannot discover a *new* leaky column. What it
        catches is someone editing feature_columns() or FORBIDDEN -- which is
        exactly the fault the Step 7 hook demonstrates."""
        cols = set(feature_columns(feats))
>       assert not (cols & FORBIDDEN), f"forbidden columns in feature set: {cols & FORBIDDEN}"
E       AssertionError: forbidden columns in feature set: {'casual'}
E       assert not ({'atemp', 'casual', 'cnt_lag_1', 'cnt_lag_7', 'cnt_roll_7', 'cnt_wow_change', ...} & {'casual', 'cnt', 'cnt_2d_bfr', 'days_since_2011', 'instant', 'registered', ...})

tests/test_leakage.py:31: AssertionError
======== short test summary info ========
FAILED tests/test_leakage.py::test_no_forbidden_features
1 failed, 8 passed in 0.68s
```

**Test output** (`make test`):
```
Exit code 2
python -m pytest tests/ -q
......F..........                                                        [100%]
=================================== FAILURES ===================================
__________________________ test_no_forbidden_features __________________________

feats =         dteday  instant  season  ...  cnt_lag_7   cnt_roll_7 cnt_wow_change
0   2011-01-10     10.0  WINTER  ...     1...857         -446.0
713 2012-12-31    731.0  SUMMER  ...      920.0  1531.428571          876.0

[714 rows x 29 columns]

    def test_no_forbidden_features(feats):
        """Guards the gate, not the data: feature_columns() excludes FORBIDDEN by
        construction, so this cannot discover a *new* leaky column. What it
        catches is someone editing feature_columns() or FORBIDDEN -- which is
        exactly the fault the Step 7 hook demonstrates."""
        cols = set(feature_columns(feats))
>       assert not (cols & FORBIDDEN), f"forbidden columns in feature set: {cols & FORBIDDEN}"
E       AssertionError: forbidden columns in feature set: {'casual'}
E       assert not ({'atemp', 'casual', 'cnt_lag_1', 'cnt_lag_7', 'cnt_roll_7', 'cnt_wow_change', ...} & {'casual', 'cnt', 'cnt_2d_bfr', 'days_since_2011', 'instant', 'registered', ...})

tests/test_leakage.py:31: AssertionError
=========================== short test summary info ============================
FAILED tests/test_leakage.py::test_no_forbidden_features - AssertionError: fo...
1 failed, 16 passed in 1.59s
make: *** [test] Error 1
```

**Fix turns**: 2 (edit to restore + verify tests pass)

---

## Fault 2: Delete .shift(1) from cnt_roll_7 causing target leakage

**Fault**: Removed `.shift(1)` from `cnt_roll_7` calculation in `src/features.py`, causing the rolling mean to include today's count instead of only past 7 days. This is target leakage since the feature now includes information that would not be available at prediction time.

**Diff**:
```diff
diff --git a/src/features.py b/src/features.py
index 03c8803..b4c0db6 100644
--- a/src/features.py
+++ b/src/features.py
@@ -73,7 +73,7 @@ def build_features(df: pd.DataFrame) -> pd.DataFrame:
     # If you add a longer lag, raise LAG_WARMUP in src/config.py.
     out["cnt_lag_1"] = out[TARGET].shift(1)
     out["cnt_lag_7"] = out[TARGET].shift(LAG_WARMUP)
-    out["cnt_roll_7"] = out[TARGET].shift(1).rolling(LAG_WARMUP).mean()
+    out["cnt_roll_7"] = out[TARGET].rolling(LAG_WARMUP).mean()
 
     # Week-over-week change: yesterday's count minus last week's count.
     # Both inputs are strictly historical (d-1 and d-7), so no leakage.


```

**Hook stderr** (PostToolUse):
```
Leakage tests failed after editing /Users/mohak/Desktop/dsc-capstone/DSC180A-Methodology-6/bikeshare-open/src/features.py:
[714 rows x 29 columns]

    def test_rolling_mean_excludes_today(feats):
        """cnt_roll_7 must be the mean of the 7 calendar days *before* today."""
        by_date = feats.set_index("dteday")[TARGET]
        date = feats.dteday.iloc[200]
        window = [date - pd.Timedelta(days=k) for k in range(1, 8)]
        if all(d in by_date.index for d in window):
            expected = sum(by_date[d] for d in window) / 7
>           assert feats.set_index("dteday").cnt_roll_7[date] == pytest.approx(expected)
E           assert np.float64(4389.0) == 4414.857142857143 ± 0.00441486
E             
E             comparison failed
E             Obtained: 4389.0
E             Expected: 4414.857142857143 ± 0.00441486

tests/test_leakage.py:57: AssertionError
======== short test summary info ========
FAILED tests/test_leakage.py::test_rolling_mean_excludes_today
1 failed, 8 passed in 0.70s
```

**Test output** (`make test`):
```
Exit code 2
python -m pytest tests/ -q
.........F.......                                                        [100%]
=================================== FAILURES ===================================
_______________________ test_rolling_mean_excludes_today _______________________

feats =         dteday  instant  season  ...  cnt_lag_7   cnt_roll_7 cnt_wow_change
0   2011-01-10     10.0  WINTER  ...     1...571         -446.0
713 2012-12-31    731.0  WINTER  ...      920.0  1789.857143          876.0

[714 rows x 29 columns]

    def test_rolling_mean_excludes_today(feats):
        """cnt_roll_7 must be the mean of the 7 calendar days *before* today."""
        by_date = feats.set_index("dteday")[TARGET]
        date = feats.dteday.iloc[200]
        window = [date - pd.Timedelta(days=k) for k in range(1, 8)]
        if all(d in by_date.index for d in window):
            expected = sum(by_date[d] for d in window) / 7
>           assert feats.set_index("dteday").cnt_roll_7[date] == pytest.approx(expected)
E           assert np.float64(4389.0) == 4414.857142857143 ± 0.00441486
E             
E             comparison failed
E             Obtained: 4389.0
E             Expected: 4414.857142857143 ± 0.00441486

tests/test_leakage.py:57: AssertionError
=========================== short test summary info ============================
FAILED tests/test_leakage.py::test_rolling_mean_excludes_today - assert np.fl...
1 failed, 16 passed in 1.71s
make: *** [test] Error 1
```

**Fix turns**: 2 (edit to restore + verify tests pass)

---

## Fault 3: Remove ValueError raise in encode_weather allowing unknown labels

**Fault**: Removed the `raise ValueError(...)` from `encode_weather()` in `src/features.py`, replacing it with `pass`. This allows unknown weather labels to silently become NaN and be deleted by `dropna()`, hiding data quality issues without warning.

**Diff**:
```diff
diff --git a/src/features.py b/src/features.py
index 03c8803..214d2fd 100644
--- a/src/features.py
+++ b/src/features.py
@@ -27,7 +27,7 @@ def encode_weather(series: pd.Series) -> pd.Series:
     known = series.dropna()
     unknown = set(known.unique()) - set(WEATHER_ORDER)
     if unknown:
-        raise ValueError(f"unmapped weather labels: {sorted(unknown)}")
+        pass
     return series.map(WEATHER_ORDER).astype(float)
```

**Hook stderr** (PostToolUse):
```
Leakage tests failed after editing /Users/mohak/Desktop/dsc-capstone/DSC180A-Methodology-6/bikeshare-open/src/features.py:
........F                         [100%]
=============== FAILURES ================
_ test_encode_weather_raises_on_unknown_label _

    def test_encode_weather_raises_on_unknown_label():
        """encode_weather must raise ValueError on unseen labels.

        This guards against silent NaN production: if the raise were removed,
        an unknown weather label would map to NaN and later dropna() would
        delete those rows without warning, hiding data quality issues.
        """
        series = pd.Series(["GOOD", "SNOWY", "MISTY"])
>       with pytest.raises(ValueError, match="unmapped weather labels"):
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
E       Failed: DID NOT RAISE ValueError

tests/test_leakage.py:91: Failed
======== short test summary info ========
FAILED tests/test_leakage.py::test_encode_weather_raises_on_unknown_label
1 failed, 8 passed in 0.59s
```

**Test output** (`make test`):
```
Exit code 2
python -m pytest tests/ -q
.............F...                                                        [100%]
=================================== FAILURES ===================================
_________________ test_encode_weather_raises_on_unknown_label __________________

    def test_encode_weather_raises_on_unknown_label():
        """encode_weather must raise ValueError on unseen labels.

        This guards against silent NaN production: if the raise were removed,
        an unknown weather label would map to NaN and later dropna() would
        delete those rows without warning, hiding data quality issues.
        """
        series = pd.Series(["GOOD", "SNOWY", "MISTY"])
>       with pytest.raises(ValueError, match="unmapped weather labels"):
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
E       Failed: DID NOT RAISE ValueError

tests/test_leakage.py:91: Failed
=========================== short test summary info ============================
FAILED tests/test_leakage.py::test_encode_weather_raises_on_unknown_label - F...
1 failed, 16 passed in 1.51s
make: *** [test] Error 1
```

**Fix turns**: 2 (edit to restore + verify tests pass)

---
