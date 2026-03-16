---
name: check-glossary
description: Verify consistent terminology across specs, code, and documentation against the project glossary
argument-hint: [scope]
allowed-tools: [Read, Glob, Grep]
context: fork
agent: Explore
---

# Check Glossary

Systematically verify that terminology is consistent across specification documents, source code, and documentation — using the project glossary as the canonical reference.

**Arguments**: `$ARGUMENTS` — optional scope: `specs` (specs only), `code` (code only), or omit for full check.

---

## Workflow

### Step 1: Load Glossary

Read `spec/glossary.md` completely. For each entry, extract:
- Canonical term
- Definition
- Any noted synonyms or abbreviations
- Related terms

### Step 2: Scan Specifications

Read all spec documents in `spec/domains/` and `spec/architecture/`. For each document:

**Synonym Detection**
- Search for known synonyms or informal variants of glossary terms
- Flag cases where a spec uses a different word for a glossary-defined concept (e.g., "character" vs. "player character" vs. "PC" when the glossary defines one canonical form)

**Undefined Terms**
- Identify domain-specific terms used frequently that have no glossary entry
- Focus on nouns that appear 3+ times and seem to name a concept, entity, or mechanic

**Consistency Within Documents**
- Flag documents that use multiple terms for the same concept internally

### Step 3: Scan Code

Search source files (models, services, schemas, routes) for terminology alignment:

**Naming Consistency**
- Do model/class names match glossary terms?
- Do database column names use glossary terminology?
- Do API endpoint paths and parameter names align?
- Do variable and function names reflect glossary definitions?

**Comment/Docstring Consistency**
- Do code comments use glossary terms or informal variants?

### Step 4: Scan Tests

Check test files for consistent terminology:
- Test function names
- Test descriptions/docstrings
- Fixture and factory names

### Step 5: Identify Stale Entries

Check for glossary entries that may be outdated:
- Terms defined in the glossary but not found in any spec, code, or test
- Terms whose definition no longer matches how they're used

### Step 6: Report

```markdown
## Glossary Check: [scope]

### Summary
| Metric | Count |
|--------|-------|
| Glossary terms checked | X |
| Consistent terms | X |
| Inconsistent terms | X |
| Undefined terms (candidates) | X |
| Stale entries | X |

### Inconsistent Usage
| Term | Canonical | Variant Found | Location |
|------|-----------|--------------|----------|
| [glossary term] | [correct form] | [variant used] | [file:line] |

### Undefined Term Candidates
| Term | Occurrences | Found In |
|------|-------------|----------|
| [term] | X | [files where it appears] |

**Proposed definitions**:
- **[term]**: [suggested definition based on usage context]

### Stale Glossary Entries
| Term | Last Found In | Status |
|------|--------------|--------|
| [term] | [nowhere / only in old specs] | [Candidate for removal / needs update] |

### Code Naming Mismatches
| Glossary Term | Code Name | Location | Suggestion |
|--------------|-----------|----------|------------|
| [canonical] | [what code uses] | [file:line] | [recommended change] |

### Recommendations
1. [Highest priority terminology fix]
2. [Second priority]
3. [Glossary additions needed]
```

---

## Rules

- The glossary is authoritative — when code and glossary disagree, flag the code (unless the glossary is clearly outdated)
- Don't flag common English words that happen to also be glossary terms (e.g., "action", "event") unless the usage clearly refers to the domain concept
- Focus on nouns and domain concepts, not verbs or generic programming terms
- A term appearing once in a comment isn't worth flagging — focus on systematic inconsistencies
- Propose new glossary entries for frequently-used undefined terms, but don't add them automatically
