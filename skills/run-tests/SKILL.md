---
name: run-tests
description: Run the test suite, report pass/fail results with failure details
argument-hint: [path]
allowed-tools: [Read, Glob, Grep, Bash]
---

# Run Tests

Run the project's test suite and report results.

**Arguments**: `$ARGUMENTS` — optional path filter (e.g., `tests/test_characters.py`). If omitted, runs the full suite.

---

## Workflow

### Step 1: Determine Test Command

Check for the project's test configuration:
1. Look for `pyproject.toml` — check for `[tool.pytest]` section
2. Look for `pytest.ini` or `setup.cfg` with pytest config
3. Default to: `pytest tests/ -v`

### Step 2: Run Tests

If a path filter is provided:
```bash
pytest <path> -v --tb=short
```

If no filter, run the full suite:
```bash
pytest tests/ -v --tb=short
```

If the first run has failures, re-run just the failures with full tracebacks:
```bash
pytest --lf -v --tb=long
```

### Step 3: Analyze Results

Parse the output for:
- Total tests run
- Tests passed
- Tests failed
- Tests skipped
- Tests errored (setup/teardown failures)

For each failure:
- Test name and file location
- The assertion that failed
- Relevant context (expected vs actual values)
- Whether it looks like a code bug vs. a test bug

### Step 4: Report

```markdown
## Test Results

### Summary
| Metric | Count |
|--------|-------|
| Total | X |
| Passed | X |
| Failed | X |
| Skipped | X |
| Errors | X |

### Failures

#### [test_name] (`tests/path/test_file.py:line`)
**Expected**: [what the test expected]
**Actual**: [what happened]
**Likely cause**: [brief analysis — code bug, test bug, missing fixture, etc.]

### Skipped Tests
- [test_name] — [reason if available]

### Verdict
[ALL PASS / X FAILURES — summary of what's broken]
```

---

## Tips

- If tests fail due to missing database tables, check if migrations need to run (e.g., `alembic upgrade head`, `npx prisma migrate dev`)
- If tests fail due to import errors, check if dependencies are installed (e.g., `pip install -e .`, `npm install`)
- If a large number of tests fail with the same error, report the root cause once rather than listing every failure
- Watch for flaky tests — if a test passes on re-run, flag it as potentially flaky
