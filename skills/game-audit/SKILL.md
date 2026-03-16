---
name: game-audit
description: Audit implemented game mechanics for balance, elegance, and player experience alignment with ops.md
argument-hint: <domain>
allowed-tools: [Read, Glob, Grep]
---

# Game Audit

Analyze implemented game mechanics for a specific domain — checking balance, design elegance, and alignment with ops.md patterns.

**Arguments**: `$ARGUMENTS` — domain name or area to audit (e.g., `magic`, `bonds`, `downtime`, `proposals`)

---

## Workflow

### Step 1: Load Design Context

Read these documents:
1. `ops/ops.md` — the portable game state patterns. This is the technical north star.
2. `ops/ops-principles.md` — OPS game design philosophy. This is the design north star — why these patterns produce good games and how to evaluate mechanics against them.
3. `spec/glossary.md` — canonical term definitions
4. The relevant domain spec (e.g., `spec/domains/magic.md` for a magic audit)
5. Your game's design document, if present (e.g., `engine.md`, `game-design.md`)

### Step 2: Load Implementation

Find and read the code for this domain:
- Models: How are the game objects stored?
- Services: Where is the game logic?
- API routes: How do players interact with it?
- Tests: What scenarios are covered?

### Step 3: Mechanical Analysis

Check the math and logic:

**Resource Flows**
- Where does this resource come from? (Sources)
- Where does it go? (Sinks)
- Is there a steady state, or does it accumulate/deplete over time?
- Are conversion rates balanced? (e.g., currency exchanges, resource-to-upgrade costs)

**Meter Behavior**
- Are boundaries enforced? What happens at min/max?
- Do boundary events trigger the right consequences? (e.g., death on HP underflow, level-up on XP overflow)
- Is clamping behavior correct?

**Resolution Mechanics**
- What inputs feed the resolution? (stats, modifiers, resources spent)
- Are modifier sources distinct and capped to prevent runaway stacking?
- Are there resource-spending options that create meaningful risk/reward tradeoffs?
- Does the math produce interesting decisions?

**Event Chain Integrity**
- Do actions produce the right event types?
- Do compound consequences record all mutations in a single event?
- Are cascades bounded (max depth, self-trigger prevention)?

### Step 4: Player Experience Evaluation

Think from the table:

- **Friction check**: How many steps does this take? Could it be simpler?
- **Decision quality**: Does this mechanic create interesting choices, or is there an obvious "best move"?
- **Narrative integration**: Does the mechanic support fiction, or fight it?
- **Frustration test**: Could a player feel cheated or confused by this mechanic?
- **Information clarity**: Can players understand their options without reading the spec?

### Step 5: OPS Pattern Alignment

Check against portable patterns (ops.md for technical invariants, ops-principles.md for design philosophy):
- Are GameObjects schema-driven with clean Spec/Status separation?
- Are Events immutable with proper dual-layer design (semantic + operations)?
- Do Triggers use data-driven rules (not hardcoded if-else chains)?
- Is the Draft/Approve/Commit flow clean?
- Are Projections derived correctly?

### Step 6: Report

```markdown
## Game Audit: [Domain]

### Mechanical Findings
| Area | Status | Notes |
|------|--------|-------|
| Resource flow | [Balanced/Imbalanced] | [details] |
| Meter boundaries | [Correct/Issues] | [details] |
| Dice pools | [Fair/Skewed] | [details] |
| Event chains | [Clean/Issues] | [details] |

### Player Experience
- **Strengths**: [What works well at the table]
- **Friction points**: [Where players might struggle]
- **Missing feedback**: [Where players need more information]

### OPS Pattern Alignment
- [Pattern] — [aligned/diverged] — [details]

### Recommendations
1. [Highest priority change]
2. [Second priority]
3. [Nice to have]

### Overall Assessment
[Summary: Does this domain serve the intended play experience? Is it mechanically sound?]
```
