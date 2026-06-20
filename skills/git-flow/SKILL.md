---
name: git-flow
description: Manage git branching, commits, and PR creation following team conventions. Use when user says "create a branch", "open a PR", "commit this", "start a feature", "merge", or "git flow for". Second phase of the agentic SDLC pipeline.
---

# Git Flow

## Objective

Automate branch creation, commits, and PR lifecycle following the team conventions in `context/team-conventions.md`.

## Inputs Required

- Action: `branch` | `commit` | `pr` | `merge`
- Feature/ticket ID (for branch naming)
- Description or spec path (for PR body)

## Execution Steps

### Step 1: Read Team Conventions

Always read `context/team-conventions.md` before any git operation to apply correct naming, commit format, and PR standards.

### Step 2: Branch — Create Feature Branch

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-flow/scripts/branch_create.py \
  --type feature \
  --ticket PROJ-123 \
  --description "short-description"
```

**Input**: branch type, ticket ID, description
**Output**:
```json
{ "branch": "feature/PROJ-123-short-description", "created": true }
```

### Step 3: Commit — Stage and Commit Changes

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-flow/scripts/commit.py \
  --type feat \
  --scope auth \
  --message "add email login flow"
```

**Input**: conventional commit type, scope, message
**Output**:
```json
{ "commit_hash": "abc1234", "message": "feat(auth): add email login flow" }
```

### Step 4: PR — Open Pull Request

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-flow/scripts/pr_create.py \
  --spec specs/feature-name.md \
  --branch feature/PROJ-123-short-description
```

**Input**: spec file path (for description), branch name
**Output**:
```json
{ "pr_url": "https://github.com/org/repo/pull/42", "pr_number": 42 }
```

## Process Flow

```
spec-kit output (confirmed spec)
    ↓
branch_create.py → feature/ticket-desc
    ↓
[code written]
    ↓
commit.py → conventional commit
    ↓
pr_create.py → PR with spec as description
    ↓
→ pr-review skill
```

## Edge Cases & Error Handling

### Branch already exists
- Append `-v2` suffix and warn user

### Uncommitted changes on current branch
- Stash before branching, warn user to pop after

### PR already open for branch
- Return existing PR URL, do not create duplicate
