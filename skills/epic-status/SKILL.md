---
name: epic-status
description: Report Epic and Story progress by cross-referencing implementation specs with existing code
argument-hint: [phase]
allowed-tools: [Read, Glob, Grep]
---

# Epic Status

Report the current implementation progress across all Epics and Stories.

**Arguments**: `$ARGUMENTS` — optional phase filter (e.g., `3` for Phase 3 only). If omitted, reports on all phases.

---

## Workflow

### Step 1: Load Implementation Index

Read `spec/implementation/README.md` for:
- The full Epic list with dependencies
- Current status tracking table
- Phase organization

### Step 2: Load Epic Files

For each Epic (or filtered by phase):
- Read the Epic spec file
- Extract all Stories with their:
  - Story number and title
  - Files to create/modify
  - Acceptance criteria

### Step 3: Cross-Reference with Code

For each Story, check whether its deliverables exist:

1. **Files to create**: Do they exist? (glob for them)
2. **Key functions/classes**: Are they defined? (grep for them)
3. **Tests**: Do test files exist for this domain?
4. **Migrations**: Are relevant database migrations present?

Classify each Story:
- **Complete**: All files exist, tests present and passing
- **In Progress**: Some files exist, partial implementation
- **Not Started**: No deliverables found
- **Blocked**: Depends on an incomplete Story

### Step 4: Report

```markdown
## Implementation Status

### Summary
| Metric | Count |
|--------|-------|
| Phases complete | X / [total] |
| Epics complete | X / [total] |
| Stories complete | X / [total] |
| Stories in progress | X |
| Stories blocked | X |

### Phase [N]: [Name]

#### Epic [X.Y] — [Title] [STATUS]

| Story | Title | Status | Notes |
|-------|-------|--------|-------|
| X.Y.1 | [title] | [status emoji] | [what exists / what's missing] |
| X.Y.2 | [title] | [status emoji] | [notes] |

**Epic progress**: X/Y stories complete
**Blockers**: [list any blocking dependencies]

### Next Steps
- **Ready to start**: [Stories whose dependencies are all met]
- **Blocked on**: [What needs to complete first]
- **Parallel opportunities**: [Stories that can run concurrently]
```

### Status Emojis
- :white_check_mark: Complete
- :construction: In Progress
- :red_circle: Not Started
- :no_entry: Blocked

---

## Depth Options

If the user wants a quick overview, report just the Epic-level status. If they want detail, drill into Stories. Default to Story-level detail for a single phase, Epic-level for all phases.
