---
name: test-aggregator
description: Parse and summarize test results from CI pipelines. Use when user says "summarize test results", "what tests failed", "test report", "CI results", or pastes test output. Supports JUnit XML, pytest JSON, and plain text logs.
---

# Test Aggregator

## Objective

Parse raw test output from CI pipelines into actionable summaries — failures, flaky tests, coverage gaps — so developers act on signal, not noise.

## Inputs Required

- Test results: file path OR piped stdin (JUnit XML, pytest JSON, plain log)
- Optional: previous run results for regression detection

## Execution Steps

### Step 1: Parse Test Results

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/test-aggregator/scripts/parse_results.py \
  --input path/to/results.xml \
  --format junit
```

Supported formats: `junit` (XML), `pytest` (JSON), `text` (auto-detect)

**Output**:
```json
{
  "total": 142,
  "passed": 138,
  "failed": 3,
  "skipped": 1,
  "duration_seconds": 47.2,
  "failures": [
    { "test": "test_user_login", "file": "tests/auth_test.py", "message": "AssertionError: expected 200, got 401" }
  ]
}
```

### Step 2: Detect Regressions (Optional)

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/test-aggregator/scripts/regression_check.py \
  --current .tmp/current-results.json \
  --previous data/test-history/last-run.json
```

**Output**:
```json
{
  "new_failures": ["test_user_login"],
  "fixed_tests": ["test_password_reset"],
  "flaky_suspects": []
}
```

### Step 3: Generate Summary Report

Produce a human-readable summary:

```
Test Run Summary — 2026-04-21 14:30
────────────────────────────────────
Passed:  138 / 142  (97.2%)
Failed:  3
Skipped: 1
Duration: 47s

FAILURES (action required):
  ✗ test_user_login — AssertionError: expected 200, got 401
  ✗ test_token_expiry — TimeoutError after 30s
  ✗ test_concurrent_sessions — Race condition detected

REGRESSIONS (new since last run):
  ✗ test_user_login
```

## Process Flow

```
CI pipeline produces test output
    ↓
parse_results.py → structured JSON
    ↓
regression_check.py (if history exists)
    ↓
Summary report → developer / PR comment
    ↓
[failures?] → back to developer for fix
[all pass?] → deployment-tracker
```

## Edge Cases & Error Handling

### Unrecognized format
- Attempt text auto-parse; flag format as unknown in output

### No previous run for regression check
- Skip regression step, note it in report

### All tests skipped
- Flag as suspicious; likely a CI config issue
