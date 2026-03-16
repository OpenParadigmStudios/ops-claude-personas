---
name: implement-story
description: Execute a single Story from an Epic spec — read spec, implement code, write tests, verify acceptance criteria
argument-hint: <epic-file> <story-number>
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# Implement Story

You are implementing a single Story from an Epic specification.

**Arguments**: `$ARGUMENTS` — expects `<epic-file> <story-number>` (e.g., `spec/implementation/phase1-scaffolding-db.md 1.1.1`)

---

## Workflow

### Step 1: Load Context

Read these files in order:
1. The Epic file specified in the arguments
2. `spec/MASTER.md` — project overview
3. `spec/glossary.md` — term definitions
4. `CLAUDE.md` — project conventions
5. All **Spec refs** listed in the target Story

Extract from the Story:
- **Files to create/modify**
- **Spec refs** (read each one)
- **Acceptance criteria** (these are your success conditions)

### Step 2: Survey Existing Code

Before writing anything:
- Check if target files already exist (glob for them)
- Read any existing files you'll be modifying
- Understand the current patterns in the codebase
- Identify any prerequisite code from earlier Stories

### Step 3: Implement

Follow this order for backend Stories:
1. **Database models** — if the Story needs new tables or fields
2. **Migrations** — if models changed, generate and verify a migration using your project's tool
3. **Service layer** — business logic in the services directory
4. **Validation schemas** — request/response models in the schemas directory
5. **API routes** — endpoints in the API routes directory

For each file:
- Read existing code first if the file exists
- Prefer editing over creating new files
- Follow existing patterns and conventions
- Add type hints and docstrings on public functions

### Step 4: Write Tests

For each piece of functionality:
- Write a test that exercises the happy path
- Write tests for error cases mentioned in acceptance criteria
- Use the project's test framework and API test client
- Reuse existing fixtures from `tests/fixtures/`

### Step 5: Verify

Run the full relevant test suite:
```bash
your-test-command -x -v
```

Check each acceptance criterion against what you implemented. Report any criteria that aren't fully met.

### Step 6: Report

Output a summary:
```markdown
## Story [number] — [title] Complete

### Files Created
- `path/to/new/file.py` — [what it does]

### Files Modified
- `path/to/existing/file.py` — [what changed]

### Tests
- X tests written, Y passing

### Acceptance Criteria
- [x] Criterion 1
- [x] Criterion 2
- [ ] Criterion 3 — [why not met, if applicable]

### Notes
- [Any decisions made during implementation]
- [Any spec ambiguities encountered]
```

---

## Rules

- Never skip reading the spec refs — they contain critical details
- If a Story depends on another Story's output that doesn't exist yet, report the blocker immediately
- If the spec is ambiguous, make the simplest reasonable choice and document it
- Don't implement beyond the Story's scope, even if you see opportunities
- Every Story must end with passing tests
