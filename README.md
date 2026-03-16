# OPS Claude Personas

A **[Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin** providing specialist agent personas, workflow skills, commands, and OPS design reference for building games on the Open Paradigm System.

It covers the full development lifecycle: spec deepening, design advisory, implementation, testing, review, and documentation. The agents understand OPS patterns natively — they evaluate, advise, and build with OPS architectural invariants and game design principles in mind.

## Installation

This is a [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins). Clone the repo into your game project's root directory:

```bash
cd /path/to/your-game-project
git clone https://github.com/OpenParadigmStudios/ops-claude-personas.git
```

Claude Code automatically detects plugins via the `.claude-plugin/plugin.json` manifest — no additional configuration needed. The next time you start Claude Code in your project directory, the agents, `/slash-command` skills, and commands from this plugin will be available.

To update:

```bash
cd ops-claude-personas && git pull
```

## What Is OPS?

The **Open Paradigm System** (OPS) is a portable pattern for managing game state through immutable events, schema-driven entities, and data-driven reactive rules. It works across any game medium — tabletop RPGs, video games, interactive narratives, mobile games, auto-battlers, and more. The architecture is always the same; only the presentation layer changes.

For the technical specification, see [`ops/ops.md`](ops/ops.md). For the human-readable design philosophy, see [`ops/ops-principles.md`](ops/ops-principles.md).

## OPS Reference Documents

### Canonical

| Document | Purpose |
|----------|---------|
| [`ops/ops.md`](ops/ops.md) | Portable game state pattern — the technical specification (GameObjects, Events, Actions, Triggers, Drafts, Projections) |
| [`ops/ops-principles.md`](ops/ops-principles.md) | Game design philosophy — human-readable guide to why OPS patterns produce good games |

### Pattern Extractions

These project-specific extractions informed the universal principles above. They're included as concrete examples of OPS patterns applied to different media.

| Document | Purpose |
|----------|---------|
| [`ops/ops-wizards.md`](ops/ops-wizards.md) | Pattern extraction from a tabletop RPG project (Wizards Engine) |
| [`ops/ops-arena.md`](ops/ops-arena.md) | Pattern extraction from an auto-battler project (Arena) |
| [`ops/ops-reality-engine.md`](ops/ops-reality-engine.md) | Pattern extraction from a narrative simulation project (Reality Engine) |

## Agent Personas

### Strategic (Opus)

| Agent | Role | Bound Skills | maxTurns | Color |
|-------|------|-------------|----------|-------|
| **orchestrator** | Reads Epic specs, delegates Stories to specialist agents, tracks completion | implement-story, run-tests, review-changes, epic-status, plan-dependencies, analyze-patterns | 200 | blue |
| **code-reviewer** | Reviews code for correctness, patterns, security, and spec alignment (read-only) | review-changes, check-spec, check-api-contract, check-glossary | 30 | red |
| **architect** | Identifies cross-cutting patterns, DRY violations, and service boundary issues (read-only) | check-spec, analyze-patterns | 30 | blue |
| **game-designer** | OPS design theory and player experience advisor — validates mechanics, patterns, and balance across any game medium (read-only) | game-audit | 30 | blue |

### Implementation (Sonnet)

| Agent | Role | Bound Skills | maxTurns | Color |
|-------|------|-------------|----------|-------|
| **backend-dev** | Server-side implementation — endpoints, services, schemas, business logic | implement-story, run-tests, check-api-contract | 100 | green |
| **db-admin** | Data layer — models, migrations, query optimization | implement-story, verify-migration | 75 | green |
| **frontend-dev** | Client-side UI — interfaces that consume the game API | implement-story, run-tests, check-api-contract | 100 | green |
| **qa-engineer** | Test writing and validation — integration tests, edge cases, fixtures | run-tests, verify-migration, check-api-contract | 75 | yellow |
| **tech-writer** | Documentation maintenance — specs, glossary, and index files kept in sync with code | check-spec, epic-status, check-glossary | 50 | green |

Implementation agents are **templates** — their tech stack prompts should be customized to match your project. The agent files contain specific stack references (e.g., Python/FastAPI, SQLAlchemy) that you'll want to update.

## Skills

Skills are agent-bindable workflows — they can be invoked by users and are injected into agent context when bound.

| Skill | Invocation | Purpose | Forked |
|-------|-----------|---------|--------|
| **implement-story** | `/implement-story <epic-file> <story-number>` | Execute a single Story from an Epic spec | |
| **review-changes** | `/review-changes [path]` | Review code changes for quality and spec alignment | |
| **run-tests** | `/run-tests [path]` | Run tests, report pass/fail with details | |
| **epic-status** | `/epic-status [phase]` | Report Epic/Story implementation progress | |
| **check-spec** | `/check-spec <spec-path>` | Verify spec matches code, update "Last verified" | |
| **game-audit** | `/game-audit <domain>` | Audit mechanics for OPS pattern alignment and player experience | |
| **plan-dependencies** | `/plan-dependencies <epic-file\|all>` | Analyze Story/Epic dependencies, generate ordered execution plan | fork |
| **analyze-patterns** | `/analyze-patterns [directory\|domain]` | Structured codebase pattern analysis — DRY, boundaries, engineering quality | fork |
| **verify-migration** | `/verify-migration [range]` | Verify migration integrity — reversibility, spec alignment, clean application | |
| **check-api-contract** | `/check-api-contract [domain]` | Verify API endpoints match schemas, frontend, and spec | fork |
| **check-glossary** | `/check-glossary [scope]` | Verify consistent terminology across specs, code, and docs | fork |

Skills marked **fork** run in an isolated context to keep the main conversation clean.

Some skills reference project-specific file paths and may need path adjustments for your project.

## Commands

Commands are user-only entry points — not bindable to agents. They handle interactive, user-initiated workflows.

| Command | Invocation | Purpose |
|---------|-----------|---------|
| **question** | `/question <spec-path>` | Deepen specs via structured Q&A rounds |
| **start** | `/start <type> <name>` | Generate scaffolding — new domain specs, Epics, or Story stubs |
| **sync-spec** | `/sync-spec [spec-path\|domain]` | Full spec reconciliation — update specs to match code reality |
| **orchestrate** | `/orchestrate <epic-file>` | Orchestrate full Epic implementation — plan, delegate, test, review |
| **help** | `/help [agents\|skills\|commands]` | List available OPS agents, skills, and commands with usage guidance |
| **status** | `/status [domain]` | Quick project health check — specs, implementation progress, next steps |

## Getting Started

1. **Ground yourself in OPS design** — Read `ops/ops-principles.md` to understand the design philosophy. This is the foundation everything else builds on.

2. **Deepen your specs** — Use `/question <spec-path>` to refine game design documents through structured Q&A. The interrogation process surfaces ambiguities and missing decisions before implementation begins.

3. **Audit your design** — Use `/game-audit <domain>` to have the game-designer agent evaluate mechanics against OPS principles and player experience for your specific medium.

4. **Build features** — Use `/implement-story <epic-file> <story-number>` to implement individual Stories from your Epic specs. The skill coordinates the right implementation agents and runs tests.

5. **Full orchestration** — Use `/orchestrate <epic-file>` to run an entire Epic end-to-end. It plans the execution order, delegates Stories to specialist agents, runs QA after each, and finishes with a code review gate.

## Expected Project Files

The plugin expects consumer projects to provide some or all of these:

| File/Directory | Purpose |
|----------------|---------|
| `CLAUDE.md` | Project conventions, tech stack, coding standards |
| `spec/MASTER.md` | Project overview and status |
| `spec/glossary.md` | Canonical term definitions |
| `spec/domains/` | Domain specification documents |
| `spec/implementation/` | Epic and Story specs |
| Game design doc | Your game's design document (e.g., `engine.md`, `game-design.md`) |

Not all files are required — the agents and skills gracefully handle missing docs by working with what's available.

## Customizing for Your Project

### Works as-is

The **strategic agents** (orchestrator, code-reviewer, architect, game-designer) are project-agnostic. They reason about specs, patterns, and design principles — not specific tech stacks.

### Needs customization

- **Implementation agents** (backend-dev, db-admin, frontend-dev, qa-engineer) contain tech stack prompts (Python/FastAPI, SQLAlchemy, pytest, etc.) that should be updated to match your project's stack.
- **Skills** that reference file paths (`implement-story`, `check-spec`, `epic-status`) may need path adjustments depending on your project's directory structure.
- **tech-writer** references specific doc conventions (MASTER.md, glossary.md) — update if your project uses different documentation patterns.

## License

MIT
