---
description: Generate scaffolding — new domain specs, Epic breakdowns, or Story stubs
argument-hint: <type> <name>
allowed-tools: [Read, Write, Edit, Glob, Grep]
---

# Scaffold

Generate scaffolding files for new project artifacts.

**Arguments**: `$ARGUMENTS` — expects `<type> <name>` where type is `domain`, `epic`, or `story`

---

## Scaffold Types

### `domain <name>`

Create a new domain specification document.

**Creates**: `spec/domains/<slug>.md`

1. Read `spec/domains/_TEMPLATE.md` if it exists; otherwise use the standard format below.
2. Generate slug from name (lowercase, hyphenated).
3. Create the spec file with placeholder sections.
4. Add entry to `spec/MASTER.md` domain table with 🔴 status.
5. Optionally add term to `spec/glossary.md`.

**Template structure**:
```markdown
# [Display Name]

**Status**: 🔴 Not started
**Last interrogated**: —
**Last verified**: —
**Depends on**: TBD
**Depended on by**: TBD

---

## Overview

[Brief description of this domain]

---

## Core Concepts

### [Concept 1]
[To be defined during interrogation]

---

## Decisions

[No decisions recorded yet]

---

## Open Questions

- [Initial questions to explore]

---

## Implications for Other Specs

[None yet]
```

### `epic <phase>.<number> <title>`

Create a new Epic specification in the implementation directory.

**Creates**: `spec/implementation/phase<N>-<slug>.md`

1. Read existing Epics to match the format.
2. Generate the Epic file with Story stubs.
3. Add entry to `spec/implementation/README.md`.

**Template structure**:
```markdown
# Epic [X.Y] — [Title]

**Phase**: [N] — [Phase Name]
**Depends on**: [Epic dependencies]
**Blocks**: [What this Epic blocks]
**Parallel with**: [Concurrent Epics]

---

## Overview

[Brief description of what this Epic delivers]

---

## Stories

### Story [X.Y.1] — [Title]

**Files to create**:
- [file paths]

**Spec refs**: [links to relevant spec documents]

**Acceptance criteria**:
- [criterion 1]
- [criterion 2]

---

## Definition of Done

- All Stories complete with passing tests
- Code review passed
- Relevant specs verified
```

### `story <epic-number> <story-title>`

Add a new Story stub to an existing Epic file.

**Modifies**: The Epic file containing the specified Epic number.

1. Read the existing Epic file.
2. Determine the next Story number.
3. Append the Story stub following the existing format.

---

## Post-Scaffold

After creating any scaffold:
1. Report what was created/modified.
2. Suggest the logical next step:
   - Domain: `/ops-question spec/domains/<slug>`
   - Epic: Fill in Story details from spec documents
   - Story: Assign spec refs and acceptance criteria
