---
name: deployment-tracker
description: Track deployments, record history, and manage rollbacks. Use when user says "record a deploy", "deployment history", "rollback", "what was last deployed", "deploy status", or "track this release". Fourth phase of the agentic SDLC pipeline.
---

# Deployment Tracker

## Objective

Record every deployment to a persistent log, enable rollback identification, and surface deployment history on demand.

## Inputs Required

- Action: `record` | `list` | `rollback-candidate` | `status`
- For `record`: environment, version/tag, PR number, deployer
- For `rollback-candidate`: environment

## Execution Steps

### Step 1: Record a Deployment

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/deployment-tracker/scripts/deploy_record.py \
  --env production \
  --version v1.4.2 \
  --pr 42 \
  --deployer "github-actions"
```

**Output**:
```json
{
  "deploy_id": "dep_20260421_143022",
  "env": "production",
  "version": "v1.4.2",
  "pr": 42,
  "deployed_at": "2026-04-21T14:30:22Z",
  "status": "success"
}
```

Storage: `data/deployments.db` (SQLite)

### Step 2: List Deployment History

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/deployment-tracker/scripts/deploy_list.py \
  --env production \
  --limit 10
```

**Output**: JSON array of last N deployments, newest first

### Step 3: Find Rollback Candidate

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/deployment-tracker/scripts/rollback_candidate.py \
  --env production
```

**Output**:
```json
{
  "rollback_to": "v1.4.1",
  "deploy_id": "dep_20260420_091500",
  "reason": "last successful deploy before current"
}
```

### Step 4: Update Deploy Status

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/deployment-tracker/scripts/deploy_record.py \
  --update dep_20260421_143022 \
  --status failed \
  --note "error rate spiked post-deploy"
```

## Process Flow

```
pr-review → approved PR merged
    ↓
CI/CD pipeline deploys
    ↓
deploy_record.py (called via n8n webhook)
    ↓
Monitor health (see runbook.md)
    ↓
[issue?] rollback_candidate.py → runbook rollback steps
[ok?]    → incident-manager (if needed)
```

## Rollback Decision Logic

Read `context/runbook.md` for rollback thresholds before recommending.
Never execute rollback directly — output the candidate version and steps.
Rollback execution requires explicit user confirmation.

## Edge Cases & Error Handling

### No prior successful deploy
- Warn: no safe rollback candidate, escalate to manual review

### Deploy recorded but status unknown
- Mark as `unknown`, prompt user to update manually
