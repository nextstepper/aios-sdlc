---
name: pr-review
description: Automated pull request review using the code-reviewer agent. Use when user says "review this PR", "check the PR", "code review", "review pull request", or provides a PR URL/number. Third phase of the agentic SDLC pipeline.
---

# PR Review

## Objective

Fetch a pull request diff, review it against team conventions and spec, and produce a structured review with blocking issues and suggestions.

## Inputs Required

- PR number or URL
- Optional: spec file path to validate implementation against requirements

## Execution Steps

### Step 1: Fetch PR Diff

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/pr-review/scripts/pr_fetch.py --pr PR_NUMBER
```

**Input**: PR number (integer) or full GitHub/GitLab URL
**Output**:
```json
{
  "pr_number": 42,
  "title": "feat(auth): add email login",
  "author": "username",
  "diff_path": ".tmp/pr-42.diff",
  "files_changed": ["src/auth.ts", "tests/auth.test.ts"],
  "additions": 120,
  "deletions": 15
}
```

### Step 2: Read Conventions and Spec

Read `context/team-conventions.md` for review standards.
If spec path provided, read it to check implementation completeness.

### Step 3: Review the Diff

Analyze the diff at `.tmp/pr-<number>.diff` focusing on:
1. **Correctness** — Does it do what the spec says?
2. **Security** — OWASP Top 10, secrets in code, injection risks
3. **Conventions** — Naming, commit format, test coverage
4. **Maintainability** — Clarity, duplication, dead code

### Step 4: Generate Review Report

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/pr-review/scripts/review_report.py \
  --pr PR_NUMBER \
  --blocking "BLOCKING: issue 1|BLOCKING: issue 2" \
  --suggestions "NIT: suggestion 1" \
  --verdict approved
```

**Verdict options:** `approved` | `changes_requested` | `comment`

**Output**: `data/reviews/pr-<number>.md` + JSON summary

## Review Output Format

```markdown
## PR #42 Review — feat(auth): add email login

**Verdict:** CHANGES REQUESTED

### Blocking Issues
- BLOCKING: No rate limiting on login endpoint — brute force risk
- BLOCKING: Password compared without constant-time function

### Suggestions (non-blocking)
- NIT: `getUserByEmail` could reuse existing `findUser` helper
- NIT: Test file missing edge case for expired tokens

### Spec Coverage
- [x] Scenario 1: Happy path login ✓
- [ ] Scenario 3: Empty state — not implemented
```

## Process Flow

```
git-flow output (PR number)
    ↓
pr_fetch.py → diff + metadata
    ↓
Review analysis (Claude reasoning)
    ↓
review_report.py → structured report
    ↓
approved → deployment-tracker
changes_requested → back to developer
```

## Edge Cases & Error Handling

### PR not found
- Verify repo and PR number; output clear error

### Diff too large (>2000 lines)
- Review by file/module, note coverage gaps in report
