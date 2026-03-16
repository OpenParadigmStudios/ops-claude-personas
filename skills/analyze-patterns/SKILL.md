---
name: analyze-patterns
description: Structured codebase pattern analysis — DRY violations, boundary issues, over/under-engineering assessment
argument-hint: [directory|domain]
allowed-tools: [Read, Glob, Grep]
context: fork
agent: Explore
---

# Analyze Patterns

Perform a structured analysis of codebase patterns across a target area — identifying repetition, boundary violations, engineering quality issues, and OPS pattern alignment.

**Arguments**: `$ARGUMENTS` — target directory or domain (e.g., `src/services/`, `models`, `api`). If omitted, scans the full source tree.

---

## Workflow

### Step 1: Load Context

Read project conventions:
- `CLAUDE.md` — project structure and conventions
- `ops/ops.md` — OPS architectural patterns (the eight operations, GameObjects, Events, Triggers)

Identify the target directories to scan based on the argument.

### Step 2: Scan for Repetition (DRY Violations)

Search for code blocks that appear 3+ times with minor variations:

**Validation Logic**
- Similar input validation across multiple endpoints or services
- Repeated permission/auth check patterns
- Duplicated error formatting or response shaping

**Query Patterns**
- Similar database queries across services (same filters, same joins)
- Repeated pagination logic
- Common lookup-by-id-with-soft-delete-check patterns

**Business Logic**
- Similar event creation/emission patterns
- Repeated meter delta + boundary check sequences
- Duplicated ref management (add/remove/validate) logic

For each pattern found, record:
- The repeated code (representative example)
- All locations where it appears (file:line)
- How the instances differ from each other

### Step 3: Check Service Boundaries

**Import Analysis**
- Map imports between service modules
- Flag circular imports (A imports B, B imports A)
- Flag services importing from other services' internal modules

**Data Access Patterns**
- Services should access their own domain's models
- Flag services querying other domains' tables directly
- Flag business logic in route handlers (should be in services)
- Flag database queries outside services/repositories

**Interface Clarity**
- Do services expose clean public APIs?
- Are there services that are just pass-throughs with no logic?
- Are there "god services" that handle too many concerns?

### Step 4: Engineering Quality Assessment

**Over-Engineering**
- Abstractions with only one implementation
- Configuration for things that never vary
- Generic frameworks solving specific problems
- Inheritance deeper than 2 levels
- Adapter/wrapper layers that add no value

**Under-Engineering**
- Copy-pasted code that diverged slightly (bug risk)
- Magic strings/numbers that should be constants
- Missing shared types (same shape defined in multiple places)
- Inconsistent error handling across similar operations

### Step 5: OPS Pattern Alignment

Check if the codebase follows OPS structural patterns:

- **GameObjects**: Do entity models share a common base pattern? Are spec/status separated?
- **Events**: Is there a single event creation path, or are events constructed ad-hoc in multiple places?
- **Operations**: Are the eight primitives (create, update, archive, meter.delta, meter.set, ref.set, ref.add, ref.remove) implemented as reusable functions?
- **Triggers**: Is the trigger/cascade mechanism centralized, or scattered as if-else chains?
- **Drafts**: Is draft/approve/commit a single workflow, or reimplemented per feature?

### Step 6: Report

```markdown
## Pattern Analysis: [target]

### Extraction Opportunities
| # | Pattern | Occurrences | Locations |
|---|---------|-------------|-----------|
| 1 | [description] | X | [file1:line], [file2:line], ... |

**Proposed extraction for #1**: [What the shared utility/base would look like]

### Boundary Issues
| # | Issue | Impact | Fix |
|---|-------|--------|-----|
| 1 | [description] | [what breaks or gets harder] | [proposed restructuring] |

### Over-Engineering
- **[area]**: [What's too complex] → [What it should be]

### Under-Engineering
- **[area]**: [What's too simple/inconsistent] → [What it needs]

### OPS Pattern Alignment
| Pattern | Status | Notes |
|---------|--------|-------|
| GameObject base | [Aligned/Missing/Partial] | [details] |
| Event creation | [Aligned/Missing/Partial] | [details] |
| Operation primitives | [Aligned/Missing/Partial] | [details] |
| Trigger mechanism | [Aligned/Missing/Partial] | [details] |
| Draft workflow | [Aligned/Missing/Partial] | [details] |

### Structural Health
- [Positive observations — patterns that work well and should be preserved]

### Priority Actions
1. [Highest-impact extraction or fix]
2. [Second priority]
3. [Third priority]
```

---

## Rules

- Only flag repetition at 3+ occurrences — two similar blocks may be coincidence
- Propose extractions, don't implement them — this is an analysis skill
- Distinguish between "similar by design" (e.g., CRUD endpoints that naturally look alike) and "similar by copy-paste" (diverged duplicates)
- OPS alignment checks only apply to projects that are implementing OPS patterns — skip this section if the codebase isn't OPS-based
- Keep the report actionable — every finding should have a clear "what to do about it"
