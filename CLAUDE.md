# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

This is a **Claude Code plugin** — not an application with build/test/lint commands. It provides agent personas, workflow skills, commands, and OPS (Open Paradigm System) design reference for building games. It's installed by cloning into a consumer project's directory; Claude Code auto-detects it via `.claude-plugin/plugin.json`.

There is no build system, no test suite, no package manager, and no compilation step. All content is Markdown files with YAML frontmatter.

## Repository Structure

```
.claude-plugin/plugin.json  — Plugin manifest (points to agents/, skills/, commands/)
agents/                     — Agent persona definitions (Markdown + YAML frontmatter)
skills/                     — Slash command skill definitions (each in its own directory with SKILL.md)
commands/                   — User-only slash commands (standalone Markdown files)
ops/                        — OPS reference documents (the design system these agents understand)
```

## Architecture: Two-Tier Agent Model

Agents are split into **strategic** (Opus, read-only) and **implementation** (Sonnet, read-write):

- **Strategic agents** (orchestrator, code-reviewer, architect, game-designer) are project-agnostic — they reason about specs, patterns, and design principles. They work as-is for any OPS project.
- **Implementation agents** (backend-dev, db-admin, frontend-dev, qa-engineer, tech-writer) are **templates** — their tech stack references (Python/FastAPI, SQLAlchemy, pytest, etc.) are placeholders that consumers must customize to their actual stack.

The orchestrator delegates Stories to implementation agents via the `/implement-story` skill, then validates via qa-engineer + code-reviewer.

## Skills vs Commands

- **Skills** (`skills/*/SKILL.md`) are agent-bindable workflows. They appear in agent `skills:` lists and are injected into subagent context when that agent is spawned. Skills can also be invoked directly by users.
- **Commands** (`commands/*.md`) are user-only entry points. They are **not** bindable to agents — they exist for interactive, user-initiated workflows like spec Q&A, scaffolding, and spec reconciliation.

The distinction: if an agent needs to invoke it during delegation, it's a skill. If only a human would trigger it, it's a command.

## Agent/Skill/Command File Format

Agent files (`agents/*.md`) use this frontmatter:
```yaml
---
name: agent-name
description: One-line role description
model: opus|sonnet
maxTurns: 100          # Safety limit on agent turns
color: blue            # UI color: blue (strategic), green (implementation), red (quality gate), yellow (QA)
tools:
  - ToolName
skills:
  - skill-name
---
```

- `maxTurns` prevents runaway agents. Strategic/read-only agents use lower limits (30); implementation agents use higher (75-100); the orchestrator uses 200.
- `color` provides visual differentiation by tier in the Claude Code UI.

Skill files (`skills/*/SKILL.md`) use:
```yaml
---
name: skill-name
description: One-line purpose
argument-hint: <required> [optional]   # Shown in autocomplete
allowed-tools: [Read, Glob, Grep]      # Pre-approved tools (no user prompt)
context: fork                          # Optional: fork runs in isolated context
agent: Explore                         # Optional: agent type for forked context
---
```

- `argument-hint` shows users what arguments the skill expects in autocomplete.
- `allowed-tools` pre-approves tool usage so users aren't prompted during skill execution.
- `context: fork` with `agent` runs the skill in an isolated context — used for read-only analysis skills that produce standalone reports.

Command files (`commands/*.md`) use:
```yaml
---
description: One-line purpose
argument-hint: <required> [optional]   # Shown in autocomplete
allowed-tools: [Read, Write, Edit]     # Pre-approved tools
---
```

Commands have no `name` field — the filename (minus `.md`) becomes the command name. Commands are not bindable to agents.

Skills and commands receive user arguments via the `$ARGUMENTS` placeholder in their body text.

## OPS Reference Documents

The `ops/` directory contains the design system all agents reference:

- `ops/ops.md` — The technical specification (GameObjects, Events, Actions, Triggers, Drafts, Projections, eight operation primitives)
- `ops/ops-principles.md` — Human-readable design philosophy explaining *why* these patterns produce good games
- `ops/ops-wizards.md`, `ops/ops-arena.md`, `ops/ops-reality-engine.md` — Concrete pattern extractions from specific game projects (tabletop RPG, auto-battler, narrative simulation)

The canonical docs (`ops.md` and `ops-principles.md`) are universal. The pattern extractions are project-specific examples.

## Key Design Concepts to Preserve

When editing agents or skills:

- **Spec-first workflow**: Agents always read specs before writing code. The `/question` command deepens specs before implementation begins.
- **Story-driven implementation**: Work is organized as Epics containing Stories. The `/implement-story` skill is the primary entry point for building features.
- **Quality gates are mandatory**: Every Story gets qa-engineer validation. Every Epic gets code-reviewer sign-off. The orchestrator enforces this sequence.
- **Read-only strategic agents**: The architect, code-reviewer, and game-designer agents must never be given write tools. They analyze and report — they don't modify.
- **Consumer project expectations**: Skills reference `spec/MASTER.md`, `spec/glossary.md`, `spec/domains/`, `spec/implementation/`, and `CLAUDE.md` in the consuming project (not in this repo).

## Editing Guidelines

- Agent and skill files are self-contained — each file is a complete prompt. Changes to one should not require changes to others unless you're modifying the inter-agent delegation protocol.
- The orchestrator's delegation flow (implement → test → review) is the backbone workflow. Changes there cascade to how all skills interact.
- Status tracking uses emoji indicators: 🔴 Not started, 🟡 In progress, 🟢 Complete, 🔄 Needs revision.
