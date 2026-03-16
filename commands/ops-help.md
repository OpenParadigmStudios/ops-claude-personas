---
description: List available OPS agents, skills, and commands with usage guidance
allowed-tools: [Read]
---

# OPS Help

Present a condensed overview of the OPS Claude Personas plugin — available agents, skills, commands, and recommended workflow.

**Arguments**: `$ARGUMENTS` — optional filter: `agents`, `skills`, `commands`, or omit for full overview.

---

## Workflow

### Step 1: Load Plugin Documentation

Read the plugin's README:
- `ops-claude-personas/README.md`

### Step 2: Present Overview

Based on the README and the optional filter, present:

**If no filter (full overview)**:

```markdown
## OPS Claude Personas — Quick Reference

### Agents

#### Strategic (Opus) — read-only analysis
| Agent | Role |
|-------|------|
| orchestrator | Reads Epic specs, delegates Stories, tracks completion |
| code-reviewer | Reviews code for correctness, patterns, security, spec alignment |
| architect | Identifies DRY violations, boundary issues, structural improvements |
| game-designer | OPS design theory and player experience advisor |

#### Implementation (Sonnet) — read-write
| Agent | Role |
|-------|------|
| backend-dev | Server-side: endpoints, services, schemas, business logic |
| db-admin | Data layer: models, migrations, query optimization |
| frontend-dev | Client-side UI consuming the game API |
| qa-engineer | Tests: integration tests, edge cases, fixtures |
| tech-writer | Documentation: specs, glossary, indexes kept in sync |

### Skills (agent-bindable workflows)
| Skill | Usage |
|-------|-------|
| /implement-story | `/implement-story <epic-file> <story-number>` |
| /run-tests | `/run-tests [path]` |
| /review-changes | `/review-changes [path]` |
| /epic-status | `/epic-status [phase]` |
| /check-spec | `/check-spec <spec-path>` |
| /game-audit | `/game-audit <domain>` |
| /plan-dependencies | `/plan-dependencies <epic-file\|all>` |
| /analyze-patterns | `/analyze-patterns [directory\|domain]` |
| /verify-migration | `/verify-migration [range]` |
| /check-api-contract | `/check-api-contract [domain]` |
| /check-glossary | `/check-glossary [scope]` |

### Commands (user-only entry points)
| Command | Usage |
|---------|-------|
| /ops-question | `/ops-question <spec-path>` — Deepen specs via structured Q&A |
| /ops-start | `/ops-start <type> <name>` — Generate scaffolding |
| /ops-sync-spec | `/ops-sync-spec [spec-path\|domain]` — Full spec reconciliation |
| /ops-orchestrate | `/ops-orchestrate <epic-file>` — Full Epic orchestration |
| /ops-help | `/ops-help [agents\|skills\|commands]` — This help |
| /ops-status | `/ops-status [domain]` — Quick project health check |

### Recommended Workflow
1. `/ops-start domain <name>` — scaffold a new domain spec
2. `/ops-question <spec-path>` — deepen the spec through Q&A
3. `/game-audit <domain>` — validate design against OPS principles
4. `/implement-story <epic> <story>` — build features
5. Orchestrator agent for full Epic execution
```

**If filtered**, show only the relevant section with expanded descriptions.

---

## Rules

- Keep the output concise — this is a quick reference, not documentation
- Include invocation syntax for every skill and command
- Point users to the full README for detailed information
