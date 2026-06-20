---
name: incident-manager
description: Create, track, and resolve production incidents. Use when user says "open an incident", "production is down", "log an incident", "P1", "what incidents are open", or "close incident". References runbook for response procedures.
---

# Incident Manager

## Objective

Log production incidents, track their lifecycle (open → investigating → resolved), and surface the right runbook procedures for the severity level.

## Inputs Required

- Action: `open` | `update` | `resolve` | `list`
- For `open`: severity (P1–P3), title, description
- For `update`/`resolve`: incident ID

## Execution Steps

### Step 1: Open an Incident

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/incident-manager/scripts/incident_open.py \
  --severity P1 \
  --title "Login service returning 503" \
  --description "Users unable to authenticate since 14:30 UTC"
```

**Output**:
```json
{
  "incident_id": "INC-20260421-001",
  "severity": "P1",
  "status": "open",
  "opened_at": "2026-04-21T14:35:00Z",
  "runbook_section": "On-Call & Escalation"
}
```

Storage: `data/incidents.db` (SQLite)

### Step 2: Surface Runbook Procedures

After opening, always read `context/runbook.md` and output the relevant section for the severity level. For P1: escalation path + response SLAs.

### Step 3: Update Incident Status

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/incident-manager/scripts/incident_update.py \
  --id INC-20260421-001 \
  --status investigating \
  --note "Traced to database connection pool exhaustion"
```

### Step 4: Resolve Incident

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/incident-manager/scripts/incident_update.py \
  --id INC-20260421-001 \
  --status resolved \
  --note "Restarted app servers, connection pool restored" \
  --resolved-at "2026-04-21T15:10:00Z"
```

### Step 5: List Open Incidents

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/incident-manager/scripts/incident_list.py --status open
```

## Incident Lifecycle

```
Production issue detected
    ↓
incident_open.py → INC-YYYYMMDD-NNN
    ↓
Runbook section surfaced automatically
    ↓
incident_update.py (status: investigating)
    ↓
deployment-tracker.rollback-candidate (if deploy-related)
    ↓
Fix applied
    ↓
incident_update.py (status: resolved + postmortem note)
```

## Severity Definitions

| Level | Meaning | Response |
|-------|---------|----------|
| P1 | Production fully down or data loss | 15 min, escalate |
| P2 | Degraded performance, partial outage | 1 hour |
| P3 | Minor issue, workaround exists | Next business day |

## Edge Cases & Error Handling

### Multiple open P1 incidents
- List all; do not merge separate incidents

### Incident opened without deployment correlation
- Note in record; do not assume deploy cause
