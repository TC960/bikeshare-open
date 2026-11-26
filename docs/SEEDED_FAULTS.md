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
