---
name: seed-fault
description: Seed a fault into src/ to test the hook and contract tests
allowed-tools: Read, Edit, Write, Bash(make*), Bash(python -m pytest*), Bash(git diff*)
---

# Seed a Fault

This skill seeds a deliberate fault into `src/` to demonstrate the PostToolUse hook and contract tests catching it. The argument `$ARGUMENTS` is the fault description.

## Steps

1. **Make the faulty edit** using the Edit tool on the relevant file in `src/`. This triggers the PostToolUse hook which will run leakage tests.

2. **Capture the failure output**: Show me verbatim:
   - The hook's stderr (the red test output from the hook)
   - The `make test` output showing which tests fail

3. **Record the fault** in `docs/SEEDED_FAULTS.md`:
   - Fault description
   - Exact `git diff` of the change
   - Red test output (from hook and make test)
   - Hook stderr verbatim

4. **STOP** and wait for me before fixing. Do not restore the code yet.

5. **When I say "fix"**: Restore the faulty edit, run `make test` until green, and add the number of turns the fix took to that section in `docs/SEEDED_FAULTS.md`.

## Example Usage

```
/seed-fault Remove the ValueError raise from encode_weather when encountering unknown labels
```

This would:
1. Edit `src/features.py` to remove the `raise ValueError(...)` line
2. Show you the hook catching it
3. Record everything in `docs/SEEDED_FAULTS.md`
4. Wait for your "fix" command before restoring
