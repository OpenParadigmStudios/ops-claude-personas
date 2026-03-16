---
name: verify-migration
description: Verify migration integrity — reversibility, spec alignment, and clean application
argument-hint: [range]
allowed-tools: [Read, Glob, Grep, Bash]
---

# Verify Migration

Check database migrations for integrity: reversibility, spec alignment, and clean upgrade/downgrade behavior.

**Arguments**: `$ARGUMENTS` — optional migration identifier or range (e.g., `head`, `abc123`, `last-3`). If omitted, verifies the full migration chain.

---

## Workflow

### Step 1: Discover Migrations

Find the project's migration directory and tool:
1. Check for Alembic: `alembic/versions/` or `migrations/versions/`
2. Check for other tools: Prisma, Knex, Django, etc.
3. List all migration files in order

```bash
alembic history --verbose
```

If a specific range is provided, filter to those migrations.

### Step 2: Check Reversibility

For each migration file:
1. Read the file completely
2. Verify both `upgrade()` and `downgrade()` are defined
3. Check that `downgrade()` is a true inverse of `upgrade()`:
   - Table created in upgrade → dropped in downgrade
   - Column added → column dropped
   - Index created → index dropped
   - Constraint added → constraint removed
4. Flag any migration with an empty or missing downgrade

### Step 3: Test Application

Run migrations forward and backward to verify they execute cleanly:

```bash
# Forward: apply all pending migrations
alembic upgrade head

# Backward: roll back to before the target migrations
alembic downgrade <base-or-target>

# Forward again: verify re-application works
alembic upgrade head
```

Report any errors encountered during migration execution.

### Step 4: Compare Against Data Model Spec

Read the project's data model specification (check `spec/domains/` for data model docs, `spec/architecture/` for schema docs).

For each table/model in the spec:
- Does a corresponding migration create it?
- Do column names, types, and constraints match?
- Are relationships (foreign keys) correctly defined?
- Are indexes present where the spec requires them?

For each migration:
- Does it align with a spec requirement, or is it ad-hoc?
- Are there migrations that contradict the spec?

### Step 5: Chain Health

Check the overall migration chain for issues:

**Ordering**
- Are migrations in a clean linear sequence, or are there branches/merges?
- Are dependency declarations correct?

**Idempotency**
- Would running `upgrade head` from a clean database produce the correct schema?
- Are there migrations that assume prior state not guaranteed by the chain?

**Conventions**
- Do migration messages follow project conventions (descriptive names)?
- Are unrelated changes separated into distinct migrations?
- Have any applied migrations been modified (dangerous — should never happen)?

### Step 6: Report

```markdown
## Migration Verification

### Summary
| Metric | Count |
|--------|-------|
| Total migrations | X |
| Reversible | X |
| Non-reversible | X |
| Spec-aligned | X |
| Spec-divergent | X |

### Reversibility Check
| Migration | Upgrade | Downgrade | Status |
|-----------|---------|-----------|--------|
| [revision] [message] | [operations] | [operations] | [OK/Missing/Incomplete] |

### Application Test
- **Forward**: [PASS/FAIL — details]
- **Backward**: [PASS/FAIL — details]
- **Re-forward**: [PASS/FAIL — details]

### Spec Alignment
| Spec Entity | Migration | Status | Notes |
|-------------|-----------|--------|-------|
| [table/model] | [revision] | [Aligned/Diverged/Missing] | [details] |

### Chain Health
- **Ordering**: [Clean/Branched — details]
- **Convention compliance**: [X/Y follow naming conventions]
- **Modified after apply**: [None found / list]

### Issues
1. [Highest priority issue]
2. [Second priority]

### Verdict
[HEALTHY / NEEDS ATTENTION / BROKEN — summary]
```

---

## Rules

- Never modify migration files — this is a read-only verification skill
- If migrations can't be run (missing database, missing dependencies), report what can be verified statically and note what couldn't be tested
- Always check both upgrade and downgrade — a migration that can't roll back is a deployment risk
- Flag any migration that was modified after being applied — this is always a bug
- Adapt the workflow to the project's migration tool — the examples use Alembic but the same checks apply to any migration system
