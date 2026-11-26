# Seeded Faults

Record of deliberately seeded faults, their detection by hooks/tests, and fixes.

Each entry includes:
- **Fault**: Description of the deliberate bug introduced
- **Diff**: The exact git diff of the faulty change
- **Hook stderr**: The PostToolUse hook's error output (verbatim)
- **Test output**: The red `make test` output (verbatim)
- **Fix turns**: Number of turns taken to restore (filled in after fix)

---
