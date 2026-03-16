---
name: review-changes
description: Review recent code changes for quality, patterns, security, and spec alignment
argument-hint: [path]
allowed-tools: [Read, Glob, Grep, Bash]
---

# Review Changes

You are reviewing code changes for quality and correctness.

**Arguments**: `$ARGUMENTS` — optional path filter (e.g., `src/my_project/services/`). If omitted, reviews all recent changes.

---

## Workflow

### Step 1: Identify Changes

If a path is provided, read files in that path:
```bash
git diff --name-only HEAD~1 -- <path>
```

If no path, check recent changes:
```bash
git diff --name-only HEAD~1
```

If no recent commits, check unstaged changes:
```bash
git diff --name-only
git diff --name-only --cached
```

### Step 2: Load Context

For each changed file:
1. Read the full file (not just the diff — context matters)
2. Identify which domain/spec area it belongs to
3. Read the relevant spec document
4. Read the glossary for term correctness

### Step 3: Review Checklist

Evaluate each changed file against:

**Correctness**
- [ ] Implements spec faithfully
- [ ] Edge cases handled (meter boundaries, soft deletes, null refs)
- [ ] Error responses match API conventions
- [ ] Primary keys generated correctly per project conventions

**Consistency**
- [ ] Follows existing codebase patterns
- [ ] Naming conventions match (snake_case, route prefixes)
- [ ] Code style uniform with surroundings

**Security**
- [ ] No SQL injection risk (parameterized queries via ORM)
- [ ] Auth checks on protected endpoints
- [ ] No sensitive data in error responses
- [ ] Input validation at boundaries

**Simplicity**
- [ ] No unnecessary abstractions
- [ ] No dead code or commented blocks
- [ ] Error handling is proportionate
- [ ] No over-engineering for hypothetical futures

**Spec Alignment**
- [ ] Field names match glossary
- [ ] Behavior matches decision blocks
- [ ] Event types follow project naming convention
- [ ] Visibility levels correct

### Step 4: Report

```markdown
## Code Review

### Scope
- Files reviewed: [count]
- Lines changed: [approximate]

### Critical Issues
- **[file:line]**: [Issue]. [Required fix].

### Moderate Issues
- **[file:line]**: [Issue]. [Suggestion].

### Minor Notes
- **[file:line]**: [Observation].

### Looks Good
- [Positive observations to reinforce good patterns]

### Verdict
[PASS / PASS WITH NOTES / NEEDS CHANGES]
```

---

## Severity Guide

- **Critical**: Must fix. Security holes, data corruption risk, fundamental spec violations.
- **Moderate**: Should fix. Pattern breaks, missing boundary validation, unclear spec interpretation.
- **Minor**: Nice to have. Style nits, naming improvements, optional refactors.
