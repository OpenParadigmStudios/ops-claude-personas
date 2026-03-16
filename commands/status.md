---
description: Quick project health check — specs, implementation progress, and next steps
argument-hint: [domain]
allowed-tools: [Read, Glob, Grep]
---

# OPS Status

Provide a quick health check of the project's current state — spec coverage, implementation progress, and recommended next steps.

**Arguments**: `$ARGUMENTS` — optional domain filter (e.g., `characters`, `magic`). If omitted, checks the full project.

---

## Workflow

### Step 1: Check Spec Coverage

Glob for spec files and assess completeness:

1. Read `spec/MASTER.md` for the status tracking table
2. Glob `spec/domains/*.md` — list all domain specs
3. Check each domain spec's status indicators (🔴 🟡 🟢 🔄)
4. Note which specs have been interrogated vs. still bare scaffolds
5. Check `spec/glossary.md` — is it populated?

### Step 2: Check Implementation Progress

1. Read `spec/implementation/README.md` for Epic/Story tracking
2. Glob `spec/implementation/*.md` — list all Epic files
3. For each Epic, count Stories and their status
4. Cross-reference with actual code — do expected files exist?

### Step 3: Check Documentation Health

1. Does `CLAUDE.md` exist and reference the project's actual stack?
2. Is `spec/MASTER.md` up to date with current status?
3. Are there specs marked 🔄 (needs revision) that haven't been addressed?

### Step 4: Report

```markdown
## Project Status

### Spec Coverage
| Domain | Status | Last Verified | Notes |
|--------|--------|--------------|-------|
| [name] | [emoji] | [date or —] | [notes] |

### Implementation Progress
| Phase | Epics | Stories Complete | Stories Remaining |
|-------|-------|----------------|-------------------|
| [N] | [count] | [X/Y] | [count] |

### Health Indicators
- **Specs needing attention**: [count with 🔴 or 🔄]
- **Stale specs**: [count not verified recently]
- **Implementation gaps**: [Stories ready but not started]

### Recommended Next Steps
1. [Most impactful action to take next]
2. [Second priority]
3. [Third priority]
```

---

## Rules

- This is a quick overview — don't deep-dive into any single area
- If spec files don't exist yet, report that clearly rather than erroring
- Focus on actionable next steps, not just status reporting
- If a domain filter is provided, show detailed status for that domain only
