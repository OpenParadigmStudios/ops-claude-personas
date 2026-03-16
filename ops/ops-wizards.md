# OPS Patterns: Wizards Engine

## What This Document Is

An extraction of Open Paradigm System (OPS) game-design patterns from the Wizards Engine project — a narrative tabletop RPG state tracker. This captures the design philosophy, mechanical patterns, and structural abstractions that make the system work, independent of implementation details.

This document is self-contained. All OPS concepts are defined inline.

---

## Project Context: Wizards Engine

**What it is**: A backend state tracker for a single narrative-heavy, low-crunch tabletop RPG campaign. Tracks character sheets, game world state, and provides a proposal workflow for player actions.

**Medium**: Tabletop — physical dice rolled at a real table. Not a virtual tabletop or dice roller.

**Scale**: Small fixed group (4-6 players + 1 GM), single campaign.

**Architecture**: API-first REST backend with a mobile-friendly web UI.

### Medium-Specific Traits That Shape the OPS Instantiation

- **GM as human approval gate**: Proposals are reviewed by a person, not auto-resolved. A Draft sits pending while the GM reads narrative and rolls physical dice at the table.
- **Physical dice**: The system calculates dice pools but never rolls or tracks roll results. The narrative IS the outcome.
- **Single campaign**: No DSL, no configuration language, no expression evaluator. All game logic is hardcoded in Python. One game, one set of rules.
- **Deferred resolution is natural**: The GM decides outcomes at the table. The system tracks the mechanical consequences, not the decision process.
- **Session-based play with downtime**: Play happens in discrete sessions with real-world gaps between them. Downtime activities happen asynchronously.
- **Bond graph models social/geographic information flow**: Who knows what, who is where, who can see which events — all derived from the same relationship graph.

### Contrast with Video Game OPS

In a video game, Drafts auto-approve in milliseconds, triggers cascade programmatically, projections update every frame. Here, a Draft sits pending while the GM reads narrative and rolls physical dice. The patterns are identical; the tick rate is human-speed.

---

## Core Philosophy

**Mutable state + append-only event log.** State (the current values of all game objects) is the source of truth. The event log records every state change as immutable history. State is what the world IS; events are what HAPPENED.

This is not event sourcing — you don't replay events to derive state. State is written directly. Events provide audit trail, activity feeds, session timelines, and future undo capability.

**No DSL — all game logic hardcoded.** No configuration language, no expression evaluator, no YAML-driven schemas. Game rules live in code. This keeps the system simple and debuggable for a single campaign.

**Deferred Narrative Resolution.** Game state is intentionally left ambiguous until narratively observed. Characters have bond-distance presence at Locations (not a pinned position) — their actual location is resolved at the table when someone looks for them. Group projects have mechanical progress (clock segments) but their outcomes are defined retroactively when the clock completes. The system supports potential/fuzzy state alongside concrete state.

**API-first.** Players propose, GM approves, system enforces. The API is the game interface. The UI is a view layer.

---

## Game Objects

### Three Narrative Node Types

Everything that exists "in the fiction" is one of three types:

- **Characters**: Beings in the world — both player characters (PCs) and non-player characters (NPCs). A unified entity with a `detail_level` discriminator: `full` (PC) with complete mechanical depth, or `simplified` (NPC) with descriptive-only fields.
- **Groups**: Organizations, crews, families, guilds. Have a power tier, project clocks, traits, and bonds to other Groups (Relations), Locations (Holdings), and Characters (Members — derived from the bond graph).
- **Locations**: Places in the world with nestable hierarchy. Have Feature Traits and unlimited bonds to any Game Object. Presence ("who's here") is computed from the bond graph, not curated lists.

### System Entities

Tracking tools that exist outside the fiction:

- **Clocks**: Progress trackers (Blades in the Dark-style). Configurable segment count. Completion is computed on read and auto-generates a resolution proposal for the GM.
- **Sessions**: Records of play sessions with a forward-only lifecycle (Draft, Active, Ended). Track campaign time, participant lists, and resource distribution.
- **Stories/Arcs**: Narrative threads with owners, status, freeform tags, and embedded narrative entries. Players contribute entries; GM controls the arc.

System Entities do not participate in the bond graph.

### Shared Structure

All Game Objects share: id, name, description, soft delete flag. Plus type-specific fields. Soft-deleted objects are hidden from lists but remain accessible via direct lookup, and references to them stay valid.

### The Detail Level Discriminator

Characters use a single entity with varying depth rather than separate PC/NPC types. A `detail_level` field (`full` or `simplified`) determines which mechanical fields are active. NPCs skip resource meters, skills, magic, and Core/Role traits — they participate in the bond graph with descriptive-only bonds but carry no mechanical weight. This keeps the conceptual model clean: in the fiction, PCs and NPCs are the same kind of thing. The detail difference is about player interaction, not ontology.

---

## Bond System

### The Unified Relationship Primitive

ALL relationships between Game Objects are Bonds. Every connection in the system — between Characters, Groups, Locations, or any combination — uses the same bond model. One concept, varying mechanical depth.

### Varying Mechanical Depth by Context

| Bond Category | Slots | Mechanics |
|---------------|-------|-----------|
| PC Bonds (Full Characters) | 8 | Stress (0-5), degradation, +1d modifier on proposals, Trauma on character stress max |
| NPC Bonds (Simplified Characters) | 7 | Descriptive only (active/retired, no mechanics) |
| Group Relations (Group-Group) | 7 | Descriptive, bidirectional with source/target labels |
| Group Holdings (Group-Location) | Unlimited | Descriptive, directional |
| Location Bonds | Unlimited | Descriptive, to any Game Object |

The 8th PC slot accounts for the expected party Group bond (a Character's bond to a Group IS their membership).

### Bond Graph as Universal Index

The bond graph does triple duty from a single traversal algorithm:

1. **Visibility**: Who can see which events and story entries (7-level model from `silent` to `global`, computed via hop distance through the bond graph).
2. **Presence**: Who is present at which Locations (1-hop = commonly present, 2-hop = often present, 3-hop = sometimes present).
3. **Membership**: A Character's bond to a Group IS their Group membership. No separate membership record — computed by querying inbound Character bonds.

### Character-Intermediary Traversal Rule

The bond graph traversal enforces a constraint: after a non-Character node (Group or Location), the next hop must go through a Character (PC or NPC). You can't traverse through two Groups or two Locations consecutively. Characters are the social connective tissue — information and presence flow through people, not through abstract organizational chains.

### Bidirectionality Defaults by Type Pairing

Character-Character, Character-Group, and Group-Group bonds default to bidirectional (one record, both sides see it with their respective labels). All Location-involved bonds default to directional. GM can override at creation.

---

## Trait System

### Three Tiers with Varying Mechanical Depth

**Core Traits** (2 slots, PC only): A character's defining qualities. Each has a charge meter (0-5). Spending 1 charge grants +1d to a proposal's dice pool. Linked to a Trait Template from the GM-created catalog — multiple characters can share the same template (e.g., "Brave").

**Role Traits** (3 slots, PC only): Learned abilities and professional expertise. Same mechanics as Core Traits (charge 0-5, +1d on spend). Different narrative category, same mechanical treatment.

**Group Traits** (10 slots): Freeform name + description. No categories enforced, no charges, no dice bonuses. Represent culture, training, assets, reputation — whatever the GM wants.

**Feature Traits** (5 slots, Location only): Freeform descriptive traits representing physical characteristics, atmosphere, dangers. All interchangeable — categories are naming conventions, not system distinctions.

### Template/Instance Two-Layer Pattern

PC traits use a two-layer architecture:

1. **Trait Template** (catalog layer): A GM-created entry with name, description, and type (core/role). Exists independently of characters. Editing a template propagates to all characters referencing it.
2. **Trait Instance** (character layer): A per-character record linking to a template. Holds character-specific state: charges, active/retired status, event history.

Players can propose new templates via the "New Trait" downtime action — on GM approval, the system auto-creates a Trait Template in the catalog and links the instance. The catalog grows organically from play.

### Modifier Stacking

A single proposal can include at most 1 Core Trait (+1d), 1 Role Trait (+1d), and 1 Bond (+1d), for a maximum of +3d on top of the base skill dice pool. This forces meaningful choice about which modifiers to activate.

### Past/Retired Lifecycle

When a PC trait is replaced (via downtime action, sacrifice, or GM action), the old instance is marked inactive. It remains on the character sheet in a "Past" section with full event history preserved and viewable, but cannot be used mechanically. This creates a narrative record of character evolution.

---

## Resource Meters

### Four PC Resources with Distinct Roles

**Stress (0-9)**: Accumulated harm and pressure. Effective max decreases by 1 per Trauma. When Stress hits max, the character gains a Trauma (a Bond slot is consumed), and Stress resets to 0. Healed via the "Rest" downtime action (3 base + up to +3 from modifiers).

**Free Time (0-20)**: The downtime currency. Gained automatically at Session Start via Time Now delta (an abstract campaign-time counter the GM controls). Spent on downtime activities (all cost 1 FT). Carries over between sessions. Capped at 20.

**Plot (0-5)**: Guaranteed successes. Each Plot spent places a guaranteed 6 before rolling — not an extra die, but a guaranteed result. Gained at Session Start (+1 per session, +2 for "Additional Contribution" meta-game reward). Can temporarily exceed 5, but clamped to 5 at Session End, giving players the active session window to convert excess via Find Time (3 Plot to 1 FT).

**Gnosis (0-23)**: The magical fuel. Spent as sacrifice in Magic Actions. Converted to additional dice via a tiered table with diminishing returns (N dice costs N*(N+1)/2 Gnosis). Regained via downtime activity. The unusual max of 23 is a deliberate lore-driven design choice.

### Meter Boundary Pattern

Certain meters have hardcoded side effects when they hit their limits:

- Character Stress at max: triggers Trauma (compound consequence — bond retired, trauma bond created, stress reset, all in one event)
- Bond Stress at max: stress resets to 0, effective max decreases by 1 (degradation)
- Free Time at max 20: excess from Time Now delta is lost
- Plot exceeds 5: clamped to 5 at Session End
- Clock Progress reaches segments: auto-generates a resolution proposal

These boundary behaviors are hardcoded in application logic — not a configurable trigger system. When a boundary is hit, the change entry in the event log gains a `clamped` annotation.

### Compound Consequences

A single game action can produce multiple mutations recorded within one event. The canonical example is Trauma: when Stress hits max, the system retires a bond, creates a trauma bond, and resets stress — all recorded as a single event. This preserves the "one event per action" invariant while allowing complex cascading effects.

---

## Magic System

### Freeform and Sacrifice-Driven

Magic is not a spell list. Players describe their magical intention, explain its symbolism, and sacrifice resources to power it. The GM reviews and creates a Magic Effect on the character's sheet as the outcome.

### Five Hardcoded Schools

Being, Wyrding, Summoning, Enchanting, Dreaming. Each has a level (0-5) that serves as the base dice pool for magical actions using that school. XP progression (flat 5 XP per level) is entirely GM-awarded — no automatic gain.

### Sacrifice as Combined Resources

A single Magic Action can combine any mix of sacrifice types:

- Gnosis (1:1 conversion)
- Stress (1 Stress = 2 Gnosis equivalent; standard Stress rules apply — can trigger Trauma mid-action)
- Free Time (1 FT = 3 + lowest Magic Stat level Gnosis equivalent)
- Bond/Trait sacrifice (10 Gnosis equivalent; the bond/trait goes to Past — destroyed)
- Other (freeform text; GM assigns Gnosis value during review)

This creates dramatic, multi-resource magical expenditure. Sacrificing a Bond is permanent — a meaningful choice between modest empowerment (+1d from Using a bond) and dramatic sacrifice (10 Gnosis from destroying it).

### Three Effect Types

- **Instant**: One-time, not tracked on the sheet. Don't count toward the effect cap.
- **Charged**: Persistent on the sheet. Power level (1-5), current/max charges. Each use costs 1 charge — player-initiated directly, no proposal needed. Rechargeable via Charge Actions.
- **Permanent**: Always-active, no charges. Power level (1-5). Created primarily via Enchanting. Boostable via Charge Actions.

### Effect Cap and Self-Retirement

Max 9 active effects (charged + permanent). Players can self-retire effects without GM approval — removing power requires no gate. Retired effects move to Past with full history preserved.

### Style Bonus

A hidden GM-only Gnosis modifier added during Magic Action review. Rewards creative narrative and good symbolism. Not visible to the player — the GM's subjective reward for quality roleplaying.

### Charge Actions

A variant of Magic Action used to recharge a charged effect (restore/increase charges) or boost a permanent effect's power level. Same workflow (intention, symbolism, sacrifice) but targeting an existing effect. Charged effects grow in capacity; permanent effects grow in strength — different investment paths.

---

## Action System

### Three Resolution Paths

**Player Proposals** (approval gate): The core workflow. Player submits a typed action with narrative, the system validates and calculates the mechanical effect, the GM reviews with full visibility into what will happen, then approves (with optional overrides and rider events) or rejects. On approval, the system auto-applies all consequences. On rejection, the player revises in place and resubmits.

**GM Actions** (direct, no queue): The GM can change any game state directly. Integrity-only validation (valid types, valid references, correct data types) — no game-logic range enforcement. The GM is trusted. GM actions reuse domain event types (e.g., `character.stress_changed`, not `gm.modify_character`) — distinguished by the actor type.

**Player Direct Actions** (no approval): Low-stakes actions that bypass proposals: Find Time (3 Plot to 1 FT), Use Effect (decrement 1 charge), Retire Effect (move to Past). These create events immediately.

### 11 Action Types

**Session actions** (3): `use_skill`, `use_magic`, `charge_magic` — dice-based actions during active play.

**Downtime actions** (7): `regain_gnosis`, `recharge_trait`, `maintain_bond`, `work_on_project`, `rest`, `new_trait`, `new_bond` — all cost 1 FT automatically.

**System proposals** (1): `resolve_clock` — auto-generated when a clock completes.

### Calculated Effects with GM Override

The system pre-computes the full mechanical effect of each proposal (dice pool, resource costs, Gnosis gains, stress healed — everything). The GM sees exactly what will happen and can override any field. Overrides REPLACE calculated values, not add to them. This gives the GM full narrative authority while keeping the math transparent.

### Rider Events

When approving a proposal, the GM can attach a rider event — a bundled direct-action event that fires atomically in the same transaction. Used for side effects: "your skill check succeeds AND the clock advances AND this NPC reacts." One player action, one approval, multiple world-state changes.

### Resource Validation

Resources are validated on submit (reject immediately if unaffordable) and deducted on approval. If resources become insufficient between submit and approval (another action was approved in between), the system returns a conflict — the GM can force-approve to acknowledge the gap.

---

## Event System

### Append-Only, Immutable Log

Every state change produces an event record. Events are never modified or deleted (except visibility, which the GM can override). One event per state-changing action, even compound changes — all mutations go into a single event's changes field.

### Three Operation Types

Each entry in an event's changes carries a classification tag:

- `field.set`: Non-numeric or unbounded fields (name, status, is_active)
- `meter.delta`: Bounded numeric adjusted by a signed amount (stress +2, charge -1)
- `meter.set`: Bounded numeric set to an absolute value (stress reset to 0)

These make the event log queryable by mutation kind without introducing a DSL.

### Compound Consequences Within Single Events

When a meter boundary triggers additional mutations, all changes are recorded in the same event. A stress increase that triggers Trauma produces one event containing: the stress delta, the old bond retirement, the new trauma bond creation, and the stress reset. No separate "boundary triggered" event.

### 7-Level Visibility Model

Events are visible to players based on bond-graph proximity:

| Level | Who Sees | Mechanism |
|-------|----------|-----------|
| `silent` | GM only (audit log) | Bookkeeping |
| `gm_only` | GM only (normal feed) | Sensitive actions |
| `private` | Owner-scoped + GM | Actor's character + primary target's owner |
| `bonded` | 1-hop bond + GM | Direct bond to any target |
| `familiar` | 2-hop bond + GM | Through Character intermediary |
| `public` | 3-hop bond + GM | Through two Character intermediaries |
| `global` | All players + GM | System announcements |

Visibility is computed on read using Character-intermediary bond-graph traversal. No pre-computed cache — computed at query time for the small player count.

### GM Can Override Visibility Post-Creation

The GM can change any event's visibility after the fact — making a secret group event `gm_only` for spoilers, or elevating a private event to `global` for dramatic reveals.

---

## Session & Downtime

### Forward-Only Lifecycle

Sessions progress through three states: Draft (editable, deletable), Active (play in progress), Ended (read-only, permanent). No undo — reversing Start would require clawing back distributed resources from all participants. Mistakes are corrected via GM direct actions.

### Time Now: Abstract Campaign Time

An integer counter set by the GM on each Session. The delta between a character's last session Time Now and the current session's Time Now determines Free Time gained. Higher delta = more FT. The GM controls pacing — a larger gap means more downtime resources.

### Resource Distribution at Session Start

On Start, the system automatically:
- Distributes Free Time via Time Now delta (capped at 20)
- Awards Plot (+1 per participant, +2 if "Additional Contribution" flagged — a meta-game reward for writing recaps, bringing props, helping organize)

### Plot Overflow and Session-End Clamp

Plot can temporarily exceed 5 from any source. It's clamped to 5 when the session ends. This gives players the active-session window to convert excess to FT via Find Time, adding meaningful resource management without long-term hoarding.

### Downtime Is Not a Mode

There's no "downtime phase" toggle. Players can submit downtime proposals whenever they have Free Time — before, during, or after sessions. The 7 downtime activities (all costing 1 FT) are always available. FT is the natural limiter.

---

## Feed

### Merged Events + Story Entries

The Feed is a unified, visibility-filtered chronological stream. It merges Events (state changes) and Story entries (narrative contributions) into a single view, using a discriminated union response shape (`type: "event"` or `type: "story_entry"`) with common fields at the top level.

This is a query pattern, not a stored entity — no new data model.

### Visibility-Filtered Per Player via Bond Graph

Every feed item is filtered through the 7-level visibility model. A player only sees events and stories reachable through their bond graph at the appropriate hop distance.

### Multiple View Scopes

- **Per-object feed**: Activity on a specific Character, Group, or Location
- **Personal feed**: Everything visible to the authenticated player across all bonds (own actions flagged with `is_own`)
- **Starred feed**: Personal feed filtered to Game Objects the player has explicitly starred

### See = Write for Stories

If a player can see a Story (via the visibility model), they can add entries to it. Visibility IS write access for narrative contribution. This encourages collaborative storytelling within the natural access boundaries of the bond graph.

---

## Design Pattern Catalog

### Soft Delete / Past/Retired

Game Objects use soft delete (hidden from lists, accessible by ID, references remain valid). Traits, Bonds, and Magic Effects use a Past/Retired pattern — marked inactive with full event history preserved, displayed in a separate "Past" section. The game world remembers what was.

### Deferred Narrative Resolution

Game state is intentionally left ambiguous until narratively observed. NPC location is a probability smear across bonded locations. Group project outcomes are undefined until the clock completes. The system supports fuzzy state because that's how GMs actually run games — not everything is decided upfront.

### Bond Graph as Universal Index

One data structure (the bond graph) drives three computed systems via a single traversal algorithm: event/story visibility, location presence, and group membership. Adding a bond doesn't just create a relationship — it changes what you can see, where you appear to be, and which groups you belong to.

### Character-Intermediary Traversal

Bond-graph traversal enforces that after a non-Character node, the next hop must go through a Character. Information flows through people, not through organizational abstractions. This models real-world social dynamics — you learn about a Guild's activities through a Guild member, not by traversing Guild-to-Guild political connections.

### Meter Boundary into Compound Consequence

Meters with hardcoded boundary behaviors trigger cascading mutations within a single event. Stress hits max, Trauma fires: bond retired, trauma bond created, stress reset — all atomic, all one event. The boundary is the trigger; the consequence is compound.

### Count-Based Slots (Not Indexed Positions)

Bond and Trait slots are count-based — the system enforces a maximum number of active items per owner type but assigns no fixed ordinal positions. Items are referenced by ID, not by slot index. Display ordering is by creation time. This simplifies the model and avoids the complexity of managing indexed positions.

### Template/Instance (Shared Catalog + Per-Entity State)

PC traits separate the definition (what the trait IS — name, description, type) from the instance (what the trait DOES for this character — charges, active status, history). Multiple characters can share a Trait Template. Editing a template propagates to all characters. The catalog grows organically when players propose new traits and the GM approves.

### Detail Level Discriminator (One Entity, Varying Depth)

Rather than separate PC and NPC entity types, a single Character entity with a `detail_level` field controls which mechanical fields are active. NPCs and PCs are the same kind of thing in the fiction — the system reflects this ontological unity. The discriminator determines depth, not identity.

### Override Replacement (GM Replaces Calculated, Not Adds)

When the GM overrides a calculated effect, the override REPLACES the calculated value — it's not additive. If the system calculates 4 Gnosis gained and the GM overrides to 6, the result is 6, not 10. This gives the GM clean authority over outcomes without having to do mental arithmetic.

### See = Write (Story Visibility = Write Access)

If a player can see a Story through the visibility model, they can add narrative entries to it. This collapses the read/write permission boundary for collaborative fiction — the bond graph already gates access appropriately, so no separate write permission is needed.

### Hidden Modifier (Style Bonus, Invisible to Player)

The GM can add a hidden Gnosis bonus to Magic Actions, rewarding creative narrative and good symbolism. The player never sees the modifier — they only see the outcome. This lets the GM incentivize quality roleplaying without making the reward mechanical knowledge.

### Union Visibility (Multiple Owners = Broader Access)

When a Story has multiple owners of different types (a PC and a Group, say), visibility is the UNION of all owner rules. The PC can see it (owner), AND anyone bonded to the Group can see it (familiar traversal). Multiple ownership broadens access, never restricts it.
