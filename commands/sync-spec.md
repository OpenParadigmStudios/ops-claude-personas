---
description: Full spec reconciliation — update spec documents to match code reality
argument-hint: [spec-path|domain]
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
---

# Sync Spec

Perform a comprehensive reconciliation between specification documents and the current codebase. This is the "heavy" counterpart to `/check-spec` — instead of just flagging divergences, it resolves them by updating specs to match code reality.

**Arguments**: `$ARGUMENTS` — optional scope filter: a spec path (e.g., `spec/domains/characters.md`), a domain name (e.g., `characters`), or omit for full-project sync.

---

## Workflow

### Step 1: Identify Code Changes

Determine what changed since specs were last verified:

```bash
git diff --name-only HEAD~5
```

If a specific spec/domain is targeted, filter to files in that domain. If no filter, scan all changed source files.

Categorize changed files by domain (models, services, routes, schemas, tests).

### Step 2: Map Changes to Spec Documents

For each changed source file:
1. Identify which domain it belongs to
2. Find the corresponding spec document in `spec/domains/`
3. Note any spec that references the changed file's entities or endpoints

Also load:
- `spec/glossary.md` — current term definitions
- `spec/MASTER.md` — current status tracking
- `CLAUDE.md` — project conventions

### Step 3: Deep Compare

For each spec-to-code pairing, compare in detail:

**Entity/Model Claims**
- Fields: name, type, constraints, defaults
- Relationships: foreign keys, refs, containment
- Indexes and unique constraints

**Behavioral Claims**
- Service methods: signatures, business logic, error conditions
- Event types: which events are emitted, their payloads
- Trigger rules: what fires and when

**API Claims**
- Endpoints: method, path, request/response schemas
- Status codes and error formats
- Auth requirements

**Constraint Claims**
- Validation rules
- Meter boundaries and overflow/underflow behavior
- Invariants and preconditions

### Step 4: Update Spec Documents

For each divergence where code is authoritative (code ahead of spec):

1. **Update the relevant section** — modify field lists, behavior descriptions, endpoint docs to match code
2. **Add decision blocks** for new implementation choices:
   ```markdown
   ### [Decision Area]
   - **Decision**: [What was implemented]
   - **Rationale**: [Why, if discernible from code/commits]
   - **Implications**: [Downstream effects]
   ```
3. **Preserve existing decisions** — don't overwrite rationale blocks that are still valid
4. **Mark "Last verified"** with today's date on updated sections

For divergences where spec should be authoritative (spec ahead of code), leave the spec as-is and note these as "not yet implemented."

### Step 5: Update Glossary

Scan code for terms that should be in the glossary:
- New model/entity names
- New event type names
- New domain concepts introduced in services or schemas

Add missing terms to `spec/glossary.md`. Update definitions of existing terms if their meaning has shifted.

### Step 6: Update MASTER.md

- Update status indicators for affected specs (🔴 → 🟡 → 🟢 or 🔄)
- Record which specs were synced and when
- Note any cross-spec implications discovered during reconciliation

### Step 7: Report

```markdown
## Spec Sync: [scope]

### Summary
| Metric | Count |
|--------|-------|
| Specs updated | X |
| Divergences resolved | X |
| Glossary terms added/updated | X |
| Not-yet-implemented (spec ahead) | X |

### Specs Updated
- `spec/domains/[name].md` — [summary of changes]

### Decision Blocks Added
- **[Spec:section]**: [New decision recorded]

### Glossary Changes
- **[term]**: [added/updated] — [definition]

### Spec Ahead (Left As-Is)
- **[Spec:section]**: [What's specified but not implemented]

### Cross-Spec Implications
- [Changes in domain X that affect domain Y's spec]

### MASTER.md Updates
- [Status changes made]
```

---

## Rules

- Never delete spec content that represents intentional design decisions — only update factual claims
- When unsure whether code or spec is authoritative, flag it for human decision rather than auto-resolving
- Preserve document structure — update sections in place, don't reorganize
- Always update glossary alongside spec changes to keep terminology consistent
- If a sync reveals that a large section of spec needs rewriting, mark it 🔄 and report rather than attempting a full rewrite
