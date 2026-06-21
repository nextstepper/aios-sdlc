<p align="center">
  <img src="https://raw.githubusercontent.com/nextstepper/aios-sdlc/main/assets/logo.svg" alt="aios-sdlc — Agentic SDLC Engine" width="360" />
</p>

<p align="center">
  <strong>Multi-agent SDLC engine for Claude Code. Spec to ship — with parallel review, peer verdicts, and zero hand-rolled orchestration.</strong>
</p>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-blue)](https://docs.claude.com/en/docs/claude-code)

## The 30-Second Pitch

Stop vibe-coding production features. `aios-sdlc` extends [GitHub Spec Kit](https://github.com/github/spec-kit) with six specialized AI agents (BA, UX, Security, Dev, QA, Code Reviewer) coordinated by an orchestrator. Every spec gets reviewed before code. Every PR gets peer-reviewed by three independent agents in parallel. Findings raised by multiple agents independently surface as **consensus signals** — the real bugs, not the noise.

## What You Get

```
┌──────────── Spec Phase ────────────┐  ┌───────────── Build + Ops ──────────┐
│ /speckit-constitution              │  │ /orchestrator    (multi-agent)     │
│ /speckit-specify  /speckit-clarify │  │ /git-flow                          │
│ /speckit-plan     /speckit-tasks   │  │ /pr-review                         │
│ /speckit-analyze  /speckit-implement│  │ /test-aggregator                  │
│       ↑ GitHub Spec Kit            │  │ /deployment-tracker                │
│                                    │  │ /incident-manager                  │
│                                    │  │       ↑ aios-sdlc                  │
└────────────────────────────────────┘  └────────────────────────────────────┘
```

## Specialized Agents

| Agent | Role | Model |
|-------|------|-------|
| `ba-agent` | Refines requirements, removes ambiguity, max-3 clarification questions | Sonnet |
| `ux-agent` | Reviews user flows, accessibility, journey coherence | Sonnet |
| `security-agent` | OWASP, threat modeling, secret detection | Opus |
| `dev-agent` | Writes implementation code from approved specs | Sonnet |
| `qa-agent` | Designs test cases, writes test automation | Sonnet |
| `code-reviewer` | Final code review pass for quality | Opus |

The **orchestrator** spawns these in parallel where independent (e.g., BA + UX during spec review) and aggregates findings into a single APPROVED / NEEDS CLARIFICATION / BLOCKED verdict.

## Install

### 1. Install Spec Kit (one-time)

```bash
brew install uv
UV_NATIVE_TLS=true uv tool install specify-cli \
  --native-tls \
  --from git+https://github.com/github/spec-kit.git
```

### 2. Initialize in your project

```bash
cd your-project
specify init . --ai claude --ai-skills --here
```

### 3. Install aios-sdlc

In Claude Code:

```
/plugin marketplace add nextstepper/aios-sdlc
/plugin install aios-sdlc
```

### 4. Add context files

```
context/
├── architecture.md      # tech stack, system design, constraints
├── team-conventions.md  # git workflow, naming, PR standards
└── runbook.md           # deploy procedure, rollback, on-call
```

All agents read these before acting. Quality in → quality out.

## Typical Workflow

```bash
# Spec phase (Spec Kit)
/speckit-constitution
/speckit-specify Add ability to mark a quote as favorite. Session-based, no auth.

# Multi-agent spec review
/orchestrator review the spec with ba-agent and ux-agent in parallel

# Iterate on blockers, then continue
/speckit-plan
/speckit-tasks
/speckit-analyze         # cross-artifact consistency gate
/speckit-implement       # dev-agent writes the code

# Build/deploy phase (aios-sdlc)
/orchestrator peer review the implementation with qa-agent, security-agent, code-reviewer in parallel
/git-flow                # branch + PR
/pr-review               # final review
/deployment-tracker record env=staging version=v1.1.0
```

## Real-World Pilot

A two-developer simulation (Alice as BA, Bob as Dev) shipped a small HTTP API + favorites feature through the full pipeline. The multi-agent review surfaced **three design-level issues before any code was written** — a method-guard contradiction, an unbounded session-map (DoS risk), and a stable-id contradiction. All three would have caused rework if implemented as-is.

📖 **Full walkthrough:** [docs/AGENTIC-SDLC-PILOT.md](docs/AGENTIC-SDLC-PILOT.md)

## When to Use This

### Strong fit
- Cross-functional teams (BA, UX, Dev, QA are distinct humans)
- Regulated industries — audit trail of spec + reviews is required anyway
- Greenfield projects where architecture decisions matter
- Teams adopting AI but worried about quality drift

### Weak fit
- Solo developers on small projects
- Throwaway prototypes
- Pure exploration / research code

### Hybrid (most common)
- Full pipeline for **new features**
- Single-agent `/pr-review` for **bug fixes / refactors**
- Skip orchestration for **typos / doc changes**

## Cost

For a 5-developer team shipping ~10 features per sprint:

| Approach | Token cost (Opus 4.8) | Production bugs | Rework cost |
|---|---|---|---|
| Vibe coding | ~$45 | 2-3 per sprint | $800-1600 |
| aios-sdlc | ~$270 | 0-1 per sprint | $0-400 |

Token cost is dwarfed by labor cost of rework. See the [pilot doc](docs/AGENTIC-SDLC-PILOT.md#5-cost--efficiency-analysis) for optimization levers (model routing, phase skip, prompt caching).

## Requirements

- Claude Code >= 1.0.0
- Python >= 3.9
- `uv` (for Spec Kit installation)
- `gh` CLI (for git-flow and pr-review)

## Repository

- **Source:** https://github.com/nextstepper/aios-sdlc
- **Spec Kit:** https://github.com/github/spec-kit
- **License:** MIT
