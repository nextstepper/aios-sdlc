# Agentic SDLC — Pilot Documentation

> A walkthrough of what agentic SDLC is, what aios-sdlc provides, and the end-to-end pilot test that proved the pipeline works on a real (if small) feature.

---

## 1. What is Agentic SDLC

Traditional software development treats AI as a coding assistant — you write code, it autocompletes. **Agentic SDLC inverts this**: AI agents own discrete phases of the lifecycle (spec, plan, review, implement, test, deploy), each operating with restricted scope and producing structured artifacts that flow into the next phase.

The shift is conceptual:

| Traditional | Agentic SDLC |
|---|---|
| Code-first, docs catch up | Spec-first, code generated from spec |
| Human reviewer (often skipped) | Multi-agent peer review every PR |
| Knowledge in developers' heads | Knowledge in committed context files |
| Bugs found in production | Design issues caught pre-implementation |
| Linear handoffs | Parallel agent dispatch where independent |

---

## 2. The aios-sdlc Architecture

Two layers working together:

```
┌────────────────────────────────────────────────────────────────────────┐
│  Spec Phase — GitHub Spec Kit (separate install)                       │
│  /speckit-constitution → /speckit-specify → /speckit-clarify →         │
│  /speckit-plan → /speckit-tasks → /speckit-analyze →                   │
│  /speckit-implement                                                    │
└────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Build / Deploy / Ops Phase — aios-sdlc plugin                         │
│  /orchestrator → /git-flow → /pr-review → /test-aggregator →           │
│  /deployment-tracker → /incident-manager                               │
└────────────────────────────────────────────────────────────────────────┘
```

### Skills (workflows, slash-invokable)

| Skill | Purpose |
|-------|---------|
| `/orchestrator` | Routes work between specialized agents; aggregates verdicts |
| `/git-flow` | Branch, commit, PR automation following team conventions |
| `/pr-review` | Single-pass AI code review (blocking/NIT format) |
| `/test-aggregator` | Parse CI test results into actionable summaries |
| `/deployment-tracker` | Record deploys + identify rollback candidates |
| `/incident-manager` | P1-P3 incident lifecycle tracking |
| `/memory` | Persistent cross-session memory via mem0 |
| `/skill-creator` | Scaffold new custom skills |

### Specialized Agents (sub-agents invoked by orchestrator)

| Agent | Role | Tools | Model |
|-------|------|-------|-------|
| `ba-agent` | Refines requirements, removes ambiguity, asks max-3 clarification questions | Read, Edit | Sonnet |
| `ux-agent` | Reviews user flows, accessibility, journey coherence | Read-only | Sonnet |
| `security-agent` | OWASP review, threat modeling, secret detection | Read-only | Opus |
| `dev-agent` | Implements features from approved specs | Read, Write, Edit, Bash | Sonnet |
| `qa-agent` | Designs test cases, writes test automation, validates coverage | Read, Write, Edit, Bash | Sonnet |
| `code-reviewer` | Final code review pass for quality/correctness | Read-only | Opus |

---

## 3. Pilot Test — Random Quote API

To validate the pipeline end-to-end, we picked a deliberately small project (a random quote HTTP API) and ran a complete two-feature SDLC cycle simulating a two-developer team.

### Setup

```
~/git-repo/quote-team/
├── shared.git/         # bare repo acting as "GitHub"
├── alice-clone/        # Developer 1 (BA + Reviewer role)
└── bob-clone/          # Developer 2 (Dev role)
```

Both clones had Spec Kit + ai-os plugin installed. Coordination happened entirely through git push/pull against `shared.git`.

### Feature 1 — Random Quote API (Alice-led)

Workflow followed:

```
/speckit-constitution    → project principles
/speckit-specify         → "HTTP API that returns a random quote"
/orchestrator (BA + UX)  → parallel spec review
/speckit-plan            → tech stack: Node.js + http, no deps
/speckit-tasks           → broken into 30+ tasks with dependency order
/speckit-analyze         → cross-artifact consistency check
/speckit-implement       → code + tests generated
```

**Outcome:** 8 tests passing. Spec, plan, tasks, contracts, and code all committed and pushed to `shared.git`.

### Feature 2 — Favorite Quotes (Alice specs, Bob implements)

This was the real multi-developer test.

#### Step 1: Alice specs the feature

```
/speckit-specify Add the ability to mark a quote as favorite.
                 Favorites are stored per-session (no auth).
                 Listing favorites and clearing all favorites are supported.
```

#### Step 2: Alice runs orchestrator with BA + UX agents

Both agents spawned in parallel. Result:

- **BA Agent (NEEDS CLARIFICATION):** 4 issues — error body shape ambiguous, ungrounded success criterion, weak randomness threshold, missing edge case
- **UX Agent (PASSES):** 4 nice-to-haves
- **Consensus finding (both independently flagged):** SC-002 randomness criterion too weak

**This is the critical signal:** when multiple agents independently raise the same issue, that's a real problem, not noise.

#### Step 3: Alice updates spec, runs plan + tasks + analyze, pushes

#### Step 4: Bob pulls, switches to `002-favorite-quotes` branch, runs `/speckit-implement`

Implementation produced:
- `server.js` extended with POST/GET/DELETE `/favorites`
- Session handling via `X-Session-Id` header
- 18 new tests for favorites (total now 26 passing)

#### Step 5: Alice pulls implementation, runs `/orchestrator` peer review

Three agents in parallel: qa-agent, security-agent, code-reviewer.

- **First attempt (wrong branch state):** All three correctly reported "no implementation exists" because Alice had not yet pulled. **Agents reported what is, not what was assumed.**
- **After git pull:** All three independently APPROVED. 30/30 tests passing after the polish round.

**Critical findings during peer review:**

1. `405/415` method guard contradiction — would have caused valid POST/DELETE to return 405 if not caught pre-implementation
2. Unbounded `sessions` Map growth — memory-exhaustion DoS vector, documented for v2
3. `id` field marked "stable across releases" but assigned positionally — would silently re-map stored favorites
4. `413` body cap implemented and tested but missing from spec contract
5. `400` error message shape diverged from spec

**Three of these (405 guard, stable id, session DoS) were design-level issues that would have caused rework if implemented without review.**

#### Step 6: Polish — dev-agent + qa-agent in parallel writing mode

When asked to action the open items, the orchestrator spawned dev-agent and qa-agent simultaneously:
- **dev-agent:** fixed `server.js` 400 message, updated `spec.md` with FR-014 (413/body cap) and v2 deferred notes, added research.md notes
- **qa-agent:** fixed T014 assertion + added 4 missing tests (DELETE unknown token, PATCH 405, absent Content-Type 415, SC-003 determinism)

File scopes were disjoint — no clobbering, no race conditions. Tests went from 26 → 30, all green.

#### Step 7: Merge to main, push

```bash
git checkout main
git merge 002-favorite-quotes
git push origin main
```

60 files changed, 5325 insertions. Pipeline complete.

---

## 4. What the Pilot Proved

| Capability | Validated |
|---|---|
| Spec Kit installs and integrates with Claude Code | ✓ |
| Multi-developer setup via shared bare repo | ✓ |
| BA + UX parallel review with consensus signal | ✓ |
| Cross-developer git handoff (push/pull) | ✓ |
| 3-agent peer review (QA + Security + Code Reviewer) in parallel | ✓ |
| Dev + QA agents writing files in parallel without conflict | ✓ |
| Orchestrator verdict aggregation (APPROVED / NEEDS CLARIFICATION / BLOCKED) | ✓ |
| Defensive bias enforcement (Security can block even when others approve) | ✓ |
| Agents reporting actual state vs assumed state (no hallucination of completed work) | ✓ |
| Pre-implementation design issue detection | ✓ — 3 real design issues caught before coding |

---

## 5. Cost & Efficiency Analysis

The pilot consumed approximately **260k tokens** for a feature that vibe-coding would have completed in roughly **20-30k tokens**. That's a 10x overhead — but the comparison is misleading.

### When agentic SDLC is NOT efficient

- Tiny features where spec is longer than the code
- Throwaway prototypes
- Solo developer with deep domain knowledge
- Time-pressured "good enough" work

### When agentic SDLC IS efficient

| Condition | Why |
|---|---|
| Multi-developer team | Misalignment cost > token cost by 10-100x |
| Production code | Bug fix in prod is 100x more expensive than catching it pre-impl |
| Regulated / compliance domain | Audit trail (spec + reviews) is already required |
| Long-lived code | Future maintainers need committed context |
| Unfamiliar problem space | Pre-implementation review prevents rework |

### Real-world cost example

A 5-developer team shipping 10 features per two-week sprint:

| Approach | Token cost (Opus 4.8) | Production bugs | Rework cost |
|---|---|---|---|
| Vibe coding | ~$45 | 2-3 bugs/sprint | 1-2 dev-days ≈ $800-1600 |
| Agentic SDLC | ~$270 | 0-1 bugs/sprint | 0-0.5 dev-days ≈ $0-400 |

**The token cost is dwarfed by the labor cost of rework.**

### Cost optimization levers

1. **Model routing** — Haiku for mechanical agents (QA test cases), Sonnet for default, Opus only for hard reasoning (Security, Architecture)
2. **Phase skip** — typo PRs don't need orchestrator, just `/pr-review`. Pipeline isn't required for every change.
3. **Prompt caching** — Anthropic API caches agent definitions + project context, ~90% discount on repeat reads
4. **Sequential when budget-constrained, parallel when speed-constrained**
5. **Trust ladder** — junior devs get full pipeline, senior devs get focused review only

---

## 6. When to Adopt

### Strong fit
- Cross-functional teams where BA, UX, Dev, QA are different humans
- Regulated industries (healthcare, finance, government)
- Greenfield projects where architecture decisions matter
- Teams adopting AI but worried about quality/security drift
- Distributed teams where async handoffs benefit from structured artifacts

### Weak fit
- Solo developers on small projects
- Pure exploration / research code
- Throwaway prototypes
- Teams where one experienced developer holds all context

### Hybrid approach (most common)
- Use Spec Kit + orchestrator for **new features**
- Use single-agent `/pr-review` for **bug fixes / refactors**
- Skip orchestration for **typos / doc changes**

---

## 7. Setup Reference

### Install Spec Kit (one-time, per machine)

```bash
brew install uv
UV_NATIVE_TLS=true uv tool install specify-cli \
  --native-tls \
  --from git+https://github.com/github/spec-kit.git
```

### Initialize in a project

```bash
cd your-project
specify init . --ai claude --ai-skills --here
```

This creates `.specify/` (templates, hooks, workflows) and `.claude/skills/speckit-*/` (9 slash-invokable spec skills).

### Install the aios-sdlc plugin

In Claude Code:

```
/plugin marketplace add nextstepper/ai-os
/plugin install ai-os
```

This adds the orchestrator skill, 5 specialized agents, and the build/deploy/ops skills.

### Populate context files

```
context/
├── architecture.md      # tech stack, system design, constraints
├── team-conventions.md  # git workflow, naming, PR standards
└── runbook.md           # deploy procedure, rollback, on-call
```

All agents read these before acting. The quality of agent output scales with the quality of these files.

---

## 8. Repository

- **Plugin source:** https://github.com/nextstepper/ai-os
- **Spec Kit source:** https://github.com/github/spec-kit
- **License:** MIT
