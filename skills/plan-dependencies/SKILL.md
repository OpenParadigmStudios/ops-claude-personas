---
name: plan-dependencies
description: Analyze Story/Epic dependencies and generate an ordered execution plan with parallelism opportunities
argument-hint: <epic-file|all>
allowed-tools: [Read, Glob, Grep]
context: fork
agent: Plan
---

# Plan Dependencies

Analyze the dependency graph of Stories within an Epic (or across Epics) and produce an ordered execution plan that maximizes parallelism while respecting dependencies.

**Arguments**: `$ARGUMENTS` — Epic file path (e.g., `spec/implementation/phase2-core-api.md`) or `all` for cross-Epic analysis.

---

## Workflow

### Step 1: Load Epic Data

Read the target Epic file(s). For each Story, extract:
- Story number and title
- Files to create/modify
- Spec references
- Explicit dependency declarations (e.g., "depends on Story X.Y.Z")
- Acceptance criteria

If `all` is specified, read `spec/implementation/README.md` and all Epic files.

### Step 2: Build Dependency Graph

Identify dependencies through three signals:

**Explicit Dependencies**
- Stories that declare dependencies on other Stories

**File-Based Dependencies**
- Story A creates a file that Story B modifies → B depends on A
- Story A creates a model that Story B's service imports → B depends on A
- Database model Stories before service Stories before API route Stories (natural layering)

**Domain Dependencies**
- Stories referencing the same spec section where one builds the foundation and another extends it
- Migration Stories must precede Stories that depend on those tables

Build an adjacency list representing the dependency graph.

### Step 3: Detect Issues

**Circular Dependencies**
- Run cycle detection on the graph
- Report any cycles with the full chain (A → B → C → A)

**Missing Dependencies**
- Stories that reference files/entities from other Stories without a declared dependency

**Bottlenecks**
- Stories that many others depend on (high fan-out)
- Long sequential chains that limit parallelism

### Step 4: Check Current Status

Cross-reference with existing code to determine which Stories are already complete:
- Glob for files each Story should create
- Check if key functions/classes exist
- Classify as: Complete, In Progress, Not Started

### Step 5: Generate Execution Plan

Group Stories into ordered **phases** (batches that can run in parallel):

1. **Phase 1**: All Stories with no unmet dependencies (or whose dependencies are already complete)
2. **Phase 2**: Stories whose dependencies are all satisfied after Phase 1 completes
3. Continue until all Stories are scheduled

Within each phase, note:
- Which specialist agent handles each Story (db-admin, backend-dev, frontend-dev)
- Estimated scope (number of files, complexity signals)

### Step 6: Report

```markdown
## Execution Plan: [Epic/scope]

### Dependency Graph
```
[Story X.Y.1] ──→ [Story X.Y.3] ──→ [Story X.Y.5]
[Story X.Y.2] ──→ [Story X.Y.3]
[Story X.Y.4] (independent)
```

### Issues
- **Circular**: [none / list cycles]
- **Bottlenecks**: [Stories with high fan-out]
- **Missing deps**: [Suspected undeclared dependencies]

### Current Status
| Story | Status | Notes |
|-------|--------|-------|
| X.Y.1 | [Complete/In Progress/Not Started] | [details] |

### Execution Phases

#### Phase 1 (parallel)
| Story | Agent | Files | Dependencies |
|-------|-------|-------|-------------|
| X.Y.1 | db-admin | 3 | none |
| X.Y.2 | db-admin | 2 | none |
| X.Y.4 | backend-dev | 4 | none |

#### Phase 2 (parallel, after Phase 1)
| Story | Agent | Files | Dependencies |
|-------|-------|-------|-------------|
| X.Y.3 | backend-dev | 5 | X.Y.1, X.Y.2 |

#### Phase 3
| Story | Agent | Files | Dependencies |
|-------|-------|-------|-------------|
| X.Y.5 | frontend-dev | 3 | X.Y.3 |

### Critical Path
[The longest sequential chain: X.Y.1 → X.Y.3 → X.Y.5 (3 phases)]

### Parallelism Summary
- Total Stories: X
- Maximum parallel width: X (in Phase N)
- Minimum sequential depth: X phases
```

---

## Rules

- Always check for circular dependencies before generating the plan — a cycle makes the plan invalid
- File-based dependencies take precedence over missing explicit declarations
- When two Stories touch the same file, assume they must be sequential unless they clearly modify different sections
- Don't over-infer dependencies — only flag what's demonstrable from file targets and spec references
- The plan is advisory — the orchestrator makes final execution decisions
