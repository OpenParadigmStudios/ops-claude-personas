---
name: check-spec
description: Lightweight spec verification — check that spec claims match code reality, update "Last verified" if aligned
argument-hint: <spec-path>
allowed-tools: [Read, Write, Edit, Glob, Grep]
---

# Check Spec

Verify that a specification document accurately reflects the current implementation.

**Arguments**: `$ARGUMENTS` — path to the spec file (e.g., `spec/domains/characters.md`)

---

## Workflow

### Step 1: Load the Spec

Read the target spec file completely. Extract:
- All entity/field claims
- Behavioral assertions (what the system does)
- Constraints and invariants
- API endpoint descriptions
- Relationships between entities

### Step 2: Load Context

- `spec/glossary.md` — term definitions
- `spec/MASTER.md` — current status

### Step 3: Find Implementation

Search for corresponding code:
- Models: your project's models directory
- Services: your project's services directory
- API routes: your project's API routes directory
- Schemas: your project's schemas directory
- Tests: your project's test directory

Match by domain name, entity names, and glossary terms.

### Step 4: Quick Compare

For each major spec claim, check:

| Claim Type | Verification |
|-----------|-------------|
| Entity exists | Model class defined? |
| Field exists | Column/attribute present? |
| Field type | Type matches spec? |
| Constraint | Validation/check present? |
| Endpoint | Route registered? |
| Behavior | Service method implements it? |

### Step 5: Classify Results

- **Aligned**: Spec matches code.
- **Spec ahead**: Spec describes something not yet implemented. (Expected during phased build.)
- **Code ahead**: Code has something not in the spec. (Needs spec update.)
- **Diverged**: Both exist but disagree. (Needs resolution.)

### Step 6: Update or Report

**If aligned**:
- Update the spec's "Last verified" date to today
- Update MASTER.md if the status should change

**If divergences found**:
- List each divergence with the spec claim and code reality
- Don't auto-fix — report for human decision
- Don't update "Last verified" — it's not verified yet

### Step 7: Output

```markdown
## Spec Check: [spec-path]

### Status: [VERIFIED / DIVERGENCES FOUND / NO IMPLEMENTATION YET]

### Verified Claims
- [Claim] — matches [file:line]

### Divergences
| # | Spec Says | Code Does | File |
|---|-----------|-----------|------|
| 1 | [claim] | [reality] | [file:line] |

### Spec Ahead (Not Yet Implemented)
- [Feature described in spec but no code exists]

### Code Ahead (Not in Spec)
- [Code feature not documented] — [file:line]

### Updated
- Last verified: [date, if aligned]
- MASTER.md: [any status changes]
```

---

## Lightweight by Design

This is a quick check, not a deep audit. For comprehensive reconciliation, use `/sync-spec` instead. The goal here is to confirm alignment and stamp the date, or flag divergences for deeper investigation.
