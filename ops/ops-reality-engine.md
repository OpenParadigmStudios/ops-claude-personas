# The Open Paradigm System — Reality Engine Reference

This document captures the complete Open Paradigm System (OPS) as expressed through Reality Engine, a backend platform for running narrative-heavy tabletop RPG campaigns. It is designed to be self-contained: an agent reading only this document should understand OPS well enough to compare Reality Engine's implementation with OPS implementations in other domains (video games, board games, real-time simulations).

---

## 1. What OPS Is

The Open Paradigm System is a **narrative-first, system-agnostic game design framework**. It provides a set of interlocking patterns for managing game state through immutable events and schema-driven entities. The patterns are engine-agnostic — they work in tabletop tools, turn-based systems, real-time simulations, and video games. Adapt the vocabulary; the structure holds.

### Core Invariant

> Only committed, immutable Events change game state. Everything else exists to propose, gate, or explain those events.

No system — UI, AI, network sync, scripting — may mutate a game entity directly. All mutations flow through the event log. This single constraint gives you: replay, undo, audit trails, deterministic networking, and a clean separation between "what the game world is" and "how players interact with it."

### Three Pillars

1. **Everything is a GameObject.** Characters, factions, locations, clocks, items, stories, the game itself — all are entities defined by schema (Kinds), carrying authored data (spec) and derived data (status).

2. **All change flows through Events.** Every mutation is an immutable record in an append-only log. Events carry both semantic meaning (what kind of thing happened) and atomic operations (what changed). The event log is the source of truth; all other state is a projection of it.

3. **Content is modular.** Game rules, entity schemas, available actions, and reactive triggers are all defined in data packages called Paradigms. No code changes are needed to add a new type of character, a new action, or a new trigger. Rules are data, not code.

### Reality Engine Context

Reality Engine is a specific OPS implementation targeting **tabletop RPGs** — campaigns with GMs, players, characters, factions, and long-running narrative continuity. Its primary concerns:

- Replace a GM's scattered notes and index cards with structured, flexible state management
- Support narrative-heavy play focused on intrigue, factions, and long-running continuity
- GMs have universal visibility; players see filtered information tied to their characters and bonds
- Event-sourced history means every change is traceable and the timeline can be reviewed or replayed
- YAML-driven configuration means game systems are portable and extensible

The validation heuristic: **if the news feed reflects the right events for the right people at the right time, the system is working.**

---

## 2. GameObjects — The Entity Model

A **GameObject** is any entity in the game world. Think of it as an index card in a well-organized GM binder — it has a type, some information, and references to other cards. What makes GameObjects narrative nodes rather than dumb data bags is the Kind system, the spec/status split, and the special field types.

### 2.1 Well-Known Types

While any Kind can be defined in a Paradigm, most OPS games will have variations of these:

| Kind | Purpose |
|------|---------|
| **Game** | The campaign instance itself. Required — every Paradigm must define this. Targets for signal Events. |
| **Character** | PCs and NPCs. A single Kind with optional PC-specific fields (player ref is nullable for NPCs). |
| **Faction** | Organizations, gangs, guilds, governments. Hold influence, pursue goals, have project clocks. |
| **Location** | Places in the world. Recursively nestable (Room in Building in City in Region). |
| **Clock** | Progress trackers, countdowns, project timelines. Fill and trigger effects at thresholds. |
| **Item** | Equipment, artifacts, resources. |
| **Story** | Narrative arcs, quests, projects, goals. Recursively nestable. |
| **Session** | Play session metadata — attendance, touched objects, chronological event grouping. |

### 2.2 The Kind System (Schema as Contract)

Every GameObject has a **Kind** — a schema that declares what fields it carries, what operations are legal on it, and what actions it can participate in. Kinds are defined in Paradigms as YAML.

```yaml
kinds:
  Character:
    description: "A person in the game world — PC or NPC"

    fields:
      # PC-only (null for NPCs)
      player:
        type: ref
        target_kind: Player

      # Numeric meters with bounds
      hp:
        type: meter
        min: 0
        max: self.status.max_hp
        default: 10

      stress:
        type: meter
        min: 0
        max: 9
        default: 0

      # Attributes
      constitution:
        type: number
        default: 2

      # Relationships
      faction:
        type: ref
        target_kind: Faction

      bonds:
        type: ref[]
        target_kind: [Character, Faction, Location]

      # Nested structured data
      appearance:
        type: object
        fields:
          height: string
          build: string
          notable_features: string[]

    # Computed values (derived into status)
    computed:
      max_hp:
        expr: self.spec.constitution * 3 + 5
      bond_count:
        expr: len(self.spec.bonds)

    # Actions available on Characters
    actions:
      - HealCharacter
      - TakeDamage
      - FormBond
```

**Why Kinds matter**: They are the contract between your content (game design) and your engine (runtime). A Kind says "a Character *is* this shape" and the engine enforces it. When you add a new Kind, you don't write new code — you declare new schema, and the existing event/operation machinery handles it.

For MVP, Kinds are standalone — no inheritance. Shared structure is achieved through similar field definitions. Future: mixins/traits or single inheritance.

### 2.3 Spec vs Status (Authored vs Derived)

Every GameObject's data splits into two layers:

| Layer | Written by | Example |
|-------|-----------|---------|
| **Spec** | Events only | `constitution: 3`, `faction: faction_042` |
| **Status** | Computed from spec | `max_hp: 14`, `bond_count: 3` |

**Spec** is the source of truth — the deliberately authored state that only changes through Events. **Status** is a cache of derived values, recomputed when spec changes. You never write to status directly; you write events that change spec, and status follows.

This split eliminates an entire class of bugs: desync between "the real value" and "the displayed value." There is no displayed value — there is spec (ground truth) and status (always derived from it).

Status recomputation is synchronous on first read after invalidation — guarantees freshness, simpler than async background jobs.

### 2.4 Reserved Fields

These fields exist on every GameObject regardless of Kind:

| Field | Type | Description |
|-------|------|-------------|
| `id` | ULID | Unique identifier (system-generated) |
| `game_id` | ULID | Parent Game (isolation boundary) |
| `kind` | string | Kind name (e.g., "Character") |
| `name` | string | Display name |
| `description` | string | Optional description/flavor text |
| `parent` | ref | Optional hierarchical parent |
| `tags` | string[] | Filtering labels |
| `owner` | selector | Who owns this object |
| `acl` | ACLEntry[] | Access control |
| `archived` | boolean | Soft-delete flag |
| `version` | integer | Modification counter |
| `created_at` | datetime | Creation timestamp |
| `updated_at` | datetime | Last modification timestamp |

GameObjects are never hard-deleted. Archiving marks them as no longer active while preserving all history.

### 2.5 Hierarchical Containment

Any GameObject can have a `parent` pointer to another GameObject, enabling recursive nesting:

```
Region (Location)
  +-- City (Location)
       +-- Building (Location)
            +-- Room (Location)

Campaign Arc (Story)
  +-- Faction Arc (Story)
       +-- Character Arc (Story)
            +-- Scene Beat (Story)
```

This is not a fixed tree structure — it's a pattern available to any Kind that constrains the `parent` field to its own Kind. A quest system, a skill tree, an inventory, an organizational chart — all the same containment pattern with different Kinds.

### 2.6 Schema Flexibility

- **All fields optional at schema level** — validation happens at use time (when Actions/Triggers need values)
- **Multi-layer defaults** — Kind definition provides defaults, EventTypes can override (EventType > Kind > null)
- **Lenient schema drift** — new fields default to null on old objects; removed fields preserved but ignored. No migration scripts needed.
- **No enum/choice fields** in MVP — use strings; enums deferred to later.

---

## 3. Special Field Types

Three field types go beyond primitives and unlock most gameplay patterns.

### 3.1 Meters — Bounded Numerics

A Meter is a special numeric field type with enforced bounds, specialized operations, and trigger potential.

```yaml
fields:
  hp:
    type: meter
    min: 0
    max: self.status.max_hp    # Expression — dynamic bound
    default: 10

  stress:
    type: meter
    min: 0
    max: 9                      # Static bound
    default: 0

  project_progress:
    type: meter
    min: 0
    max: 8
    default: 0
```

Meter bounds can be static numbers or expressions referencing other fields.

**Overflow/underflow behavior:**
When a meter operation would push past bounds:
1. **Clamp** — the value stops at the limit
2. **Emit Event** — the system generates a `MeterOverflow` or `MeterUnderflow` system event recording how much was lost
3. **Trigger Potential** — triggers can react to these events

This means "when HP hits 0, trigger death sequence" isn't special-cased code — it's a trigger on MeterUnderflow. "Convert excess healing to temporary shields" is a trigger on MeterOverflow. The mechanic emerges from the general pattern.

### 3.2 Refs — Typed Pointers

A Ref is a typed pointer to another GameObject. Can be singular or a list, with optional Kind constraints.

```yaml
fields:
  # Singular ref with Kind constraint
  faction:
    type: ref
    target_kind: Faction

  # List ref with multiple allowed Kinds
  bonds:
    type: ref[]
    target_kind: [Character, Faction, Location]

  # Unconstrained ref list
  allies:
    type: ref[]
```

Refs make relationships first-class. You don't store a faction_id string and hope it's valid — the system enforces referential integrity and detects dangling references when targets are archived.

**Ref traversal behavior:**
- Archived objects are excluded from traversal by default
- Null refs are skipped silently
- Cross-object traversal: `self.spec.bonds.*.spec.loyalty` returns a list of values

**Dangling reference handling:**
When a GameObject is archived, refs pointing to it become dangling. The system alerts the GM, who can: re-assign the ref, clear it, create a replacement, or ignore it for historical record. Archived objects remain queryable with an `archived: true` flag.

### 3.3 Tagged Refs — Relationship Metadata

All refs can carry optional **tags** — simple string arrays that annotate the reference with metadata.

```yaml
# Simple ref (no tags)
faction: char_123

# Tagged ref
faction:
  target: char_123
  tags: ['primary', 'founder']

# Ref list with mixed tagging
owners:
  - target: char_001
    tags: ['primary']
  - target: faction_002
    tags: ['supporting', 'sponsor']
  - target: char_003    # No tags
```

**Use cases:**
- **Role annotation**: `owners` refs tagged with `['primary']`, `['antagonist']`
- **Relationship types**: `bonds` refs tagged with `['ally']`, `['rival']`, `['family']`
- **Provenance**: `related_events` refs tagged with `['trigger']`, `['manual']`

Tags let you express relationship *roles* without needing a separate join entity. "Who is the primary owner?" is a query filter, not a separate data model.

**Query access:**
```
# Find owners with 'primary' tag
self.spec.owners[tags contains 'primary']

# Check if any owner is tagged as antagonist
self.spec.owners.any(o => o.tags contains 'antagonist')
```

**Operations on tags:**
- `ref.add` and `ref.set` accept an optional `tags` parameter
- `ref.tag` modifies tags on existing refs (add/remove tags)

### 3.4 Nested Objects

Kinds can contain structured nested data:

```yaml
fields:
  appearance:
    type: object
    fields:
      height: string
      hair_color: string
      distinguishing_marks: string[]

  background:
    type: object
    fields:
      origin:
        type: ref
        target_kind: Location
      family: string
```

---

## 4. Traits and Perks

Traits and Perks are OPS concepts for attaching narrative-mechanical descriptors and conditional special rules to entities. They originate from Reality Engine's game design layer and represent the vocabulary a game designer uses to give characters (and other entities) distinct identities and capabilities.

### 4.1 What Traits Are

Traits describe the important things that make something special in the story. They are narrative-mechanical descriptors that can be attached to **any** entity — characters, factions, locations, or any other GameObject. They are useful bits of description, noting something important to track in the narrative profile of a thing.

### 4.2 Varieties of Traits

Traits come in several natural varieties, though OPS does not impose a rigid taxonomy. Each OPS project defines its own categories. The common varieties are:

**Personal attributes** — personality, physical structure, mental habits, or core background:
- Strong, Clever, Confident, Observant, Proud, Empath, Hot Headed, Royal Blood, Graceful, Handsome, Calculating, Cautious

**Training and specialization** — skills, careers, roles, and initiatory paths:
- Special training: Scientist, Doctor, Tactician, Swordmaster
- Party roles: Leader, Support, Face, Guardian
- Career backgrounds: Merchant, Smith, Farmer, Academic
- Initiatory paths: a particular priesthood training, magical tradition, secret martial art school

**Bonds to other entities** — deep connections to important things in the game:
- A character's mentor, lover, family member, guild, apprentice
- These create relationships that carry narrative weight and mechanical significance

### 4.3 Trait Structure

Every Trait has:
- **Name** — a short-form label (e.g., "Swordmaster", "Strong", "Royal Blood")
- **Description** — a paragraph explaining what the Trait represents

In addition, some Traits carry:
- **Meters** — measuring progress, power level, uses remaining, or other tracked values
- **Perks** — conditional special rules (see below)

### 4.4 What Perks Are

Perks are small, reusable mechanical special rules attached to Traits. They let entities do things they normally wouldn't be able to — granting extra dice on rolls, giving resource refunds in certain conditions, unlocking special moves, introducing new rules as conditional special cases.

### 4.5 Basic vs Advanced Perks

Each Trait can have two categories of Perks:

**Basic Perks** — anyone with the Trait gets these automatically. They represent the fundamental capabilities that come with the Trait.

**Advanced Perks** — a list of upgrade options attached to the Trait. Not all entities with the Trait will have these. Instead, an entity may acquire Advanced Perks through character advancement mechanisms. This creates a progression path within each Trait.

### 4.6 Context-Dependent Perk Flavor

A key OPS pattern: the **same Perk** can appear on multiple Traits with **different narrative context**. The Perk's mechanical effect is settings-agnostic, but each Trait provides a narrative context line describing how and when it applies.

Example — the **Confident** Perk (grants +1d on a roll with variable Stamina cost):

| Trait | Context |
|-------|---------|
| **Introspective** | "Due to long time spent analyzing your own strengths and weaknesses, you may be Confident when you already feel you have a strong chance of success — when you have 3d or more in your dice pool." |
| **Reckless** | "You are Confident in your chances of success when you probably shouldn't be — when you have 1d or less in your dice pool." |

Example — the **Specialty** Perk (Push Yourself costs 1 Stamina instead of 2):

| Trait | Context |
|-------|---------|
| **Swordmaster** | "You may apply your Specialty whenever you are fighting with, evaluating, crafting, or otherwise doing anything involving a sword." |
| **Scientist** | "Your Specialty applies whenever you are doing anything that your scientific knowledge would help in." |
| **Strong** | "Your Specialty is in engaging your muscles to move, push, pull, crush, or smash things or otherwise do heroic acts of great strength." |

Example — the **Backup** Perk (when aiding an ally, they may treat Mixed as Success):

| Trait | Context |
|-------|---------|
| **Motivator** | "You easily fall into work harmony with anyone you share any Bond or Trait with." |
| **Bond between two Characters** | "Because of the close relationship between the two of you, you may always act as Backup for each other when you work together." |

This pattern means Perks are designed once as mechanics, then given narrative meaning by each Trait that grants them. The same mechanical effect can feel completely different depending on which Trait provides it.

### 4.7 Perk Examples

Perks cover a wide range of mechanical effects:

**Action enhancement:**
- **Confident** — +1d on a roll, Stamina cost varies by result (Botch: lose 3+, Crit: free)
- **Careful** — spend 1 Stamina before rolling; Mixed results treated as Success
- **Practiced** — on a Botch, spend Stamina to downgrade to Failure
- **Second Wind** — spend 1 Stamina before rolling; recover Stamina on Success/Crit

**Support and teamwork:**
- **Supporter** — if you Aid and they Succeed/Crit, refund the Stamina cost
- **Helpful** — spend extra Stamina when Aiding to give +1d
- **Backup** — if you Aid, they may treat Mixed as Success
- **Bodyguard** — pay 1 Stamina to make a defensive roll on behalf of an ally

**Resource management:**
- **Specialty** — Push Yourself costs 1 Stamina instead of 2
- **Effortless** — recover 1 Stamina when Push Yourself results in Success/Crit
- **Expanded Inventory** — gain +2 Gear slots

**Camp and downtime:**
- **Boost Morale** — spend 1 Supply to recover 1 Stamina for every character
- **Forager** — spend 1 Stamina to gather 3 Supply charges
- **Maintenance** — spend 1 Stamina to give a piece of Gear +1d to all rolls next day

**Crafting:**
- **Make Healing Potions** — spend resources to create healing consumables
- **Make Fire Bombs** — spend resources to create offensive consumables
- **Make Poison Flasks** — spend resources to create debuff consumables

**Challenge-specific (Momentum/Tension system):**
- **Press Your Advantage** — spend Momentum before a roll; gain Momentum on Success and reduce Tension
- **Turn the Tables** — spend all Momentum to roll Tension as dice, potentially clearing large amounts of Tension on Success

### 4.8 Abstract vs Instantiated

Traits exist in two places simultaneously:

**Abstract (template)**: A Trait definition in the game system describes all potential options — the full list of Perks, the meter definitions, the description template. This is the Platonic form of the Trait.

**Instantiated (on an entity)**: A particular Character having a Trait, with tracked state — which Advanced Perks they've purchased, current meter values, their personal narrative context. The instance references the abstract template but carries its own state.

This mirrors the Kind/GameObject pattern: Kinds are templates, GameObjects are instances. Traits are templates, instantiated Traits are tracked state on specific entities.

---

## 5. Events — Immutable State Records

Events are the atomic primitive of OPS. Every mutation to the game world is recorded as an immutable Event, forming a complete, ordered history that can be replayed deterministically.

### 5.1 Dual-Layer Design

Every Event carries two layers of information:

```yaml
Event:
  # Semantic layer — WHAT kind of thing happened
  type: "DowntimePasses"
  parameters: { days: 7 }

  # Operations layer — WHAT mutations it performs
  operations:
    - op: meter.delta
      target: faction_001
      payload: { meter: progress, delta: +1 }
    - op: meter.delta
      target: faction_002
      payload: { meter: progress, delta: +2 }
```

**Semantic layer** (the EventType name): Used for trigger matching and human reasoning. "A downtime event happened" is meaningful to game logic and narrative.

**Operations layer** (the mutation list): Used for actually updating state. These are the atomic, mechanical changes. Projections consume this layer to compute current state.

**Why both?** One event type can produce different operations depending on context, and different event types can produce the same operation. A "HealCharacter" and a "RestAtInn" might both emit `meter.delta` on HP, but triggers care about *why* the healing happened, not just that HP changed.

### 5.2 The Eight Operations

All state mutations reduce to combinations of eight atomic primitives:

| Operation | What it does | Payload |
|-----------|-------------|---------|
| `object.create` | Instantiate a new GameObject | `kind`, `initial_data` |
| `object.update` | Modify a data field (path-based) | `path`, `value` |
| `object.archive` | Soft-delete a GameObject | (none) |
| `meter.delta` | Adjust a meter by a signed amount (clamped) | `meter`, `delta` |
| `meter.set` | Set a meter to an absolute value (clamped) | `meter`, `value` |
| `ref.set` | Set a singular ref | `field`, `target_id`, `tags?` |
| `ref.add` | Add to a ref list | `field`, `target_id`, `tags?` |
| `ref.remove` | Remove from a ref list | `field`, `target_id` |

Additionally, `ref.tag` modifies tags on existing refs (`add_tags`, `remove_tags`).

Every gameplay mechanic — combat, crafting, dialogue consequences, weather changes, faction politics, quest progression — decomposes into sequences of these primitives. Fewer operation types means fewer code paths, fewer bugs, and total replay fidelity.

### 5.3 EventType Definitions

EventTypes are named schemas defined in Paradigms. They specify what targets are required, what parameters are accepted, and what operations are performed when instantiated.

```yaml
event_types:
  DowntimePasses:
    description: "Signals that a downtime phase has begun"
    targets:
      primary: Game
    parameters: {}
    operations: []           # Signal only — no mutations
    narrative_template: "Downtime begins"
    tags: [downtime, phase]

  ProgressFaction:
    description: "Advances a faction's project progress"
    targets:
      primary: Faction
    parameters:
      amount:
        type: integer
        default: 1
      reason:
        type: string
        optional: true
    operations:
      - op: meter.delta
        target: $primary
        payload:
          meter: project_progress
          delta: $amount
    narrative_template: "{{primary.name}} advances project by {{amount}}"
    tags: [faction, progress]
```

### 5.4 Event Metadata and Causality

Events carry rich metadata for debugging, auditing, and causal reasoning:

```yaml
Event:
  id: ULID                          # globally unique, sortable
  game_id: ...                      # isolation boundary
  seq: 42                           # strict per-game ordering

  scope:
    primary: char_001               # main target
    refs: [faction_002, loc_003]    # other involved objects

  causality:
    parent_event_id: evt_041        # what triggered this (null if root)
    root_draft_id: draft_012        # originating player action
    depth: 1                        # cascade depth (0 = player-initiated)

  actor:
    type: user | system | trigger
    id: ...

  timestamp: ...
  session_id: ...                   # optional grouping by play session
  narrative: "The Fog Hounds gain ground while the crew rests."
  tags: [faction, progress]
```

The **causality** block lets you answer:
- "What player action ultimately caused this?" — follow `root_draft_id`
- "What was the chain of events?" — follow `parent_event_id` up
- "How deep is this cascade?" — check `depth`

### 5.5 Signal Events

Events with no operations that target the Game object itself. Used to broadcast that something happened without directly mutating state.

`DowntimePasses` is the canonical example — it doesn't reduce anyone's HP or advance any projects. It just announces that downtime is happening. Triggers listen for the signal and emit their own Events with actual mutations.

The signal itself does nothing. The reactions do everything.

### 5.6 System Events

The engine automatically emits system events during lifecycle moments. These are normal EventTypes defined in a Core Paradigm, implicitly included in all games.

| System Event | When | Purpose |
|-------------|------|---------|
| **MeterOverflow** | A meter operation clamps at maximum | Payload: meter name, requested value, overflow amount |
| **MeterUnderflow** | A meter operation clamps at minimum | Payload: meter name, requested value, underflow amount |
| **ObjectCreated** | A new GameObject commits | Payload: Kind name |
| **ObjectArchived** | A GameObject is soft-deleted | Payload: Kind name |
| **RefDangling** | An archived object leaves dangling refs | Payload: field name, archived object ID/Kind |

System events:
- Emit as child events (depth+1) after the parent event commits
- Inherit actor from parent event (maintains audit trail)
- Always emit — cannot be suppressed
- Participate in the same trigger/cascade system as any other event

**Cascade order when an event commits:**
1. Parent event commits
2. System events emit (MeterOverflow/Underflow, ObjectCreated, ObjectArchived, RefDangling)
3. Normal triggers evaluate and fire
4. Cascade continues

---

## 6. Stories and Narrative Structure

Stories are OPS's mechanism for tracking narrative arcs — quests, character goals, faction schemes, ongoing plots. They are GameObjects of Kind "Story" with special patterns that make them powerful narrative tools.

### 6.1 Recursive Nesting

Stories are **recursively nestable** — a Story can contain child Stories using the standard `parent` field. Whether something is a "beat" or an "arc" depends on where you're standing:

```
Campaign Arc: "The Fall of Doskvol" (Story)
  +-- Faction Arc: "The Lampblacks' Rise" (Story)
       +-- Personal Arc: "Marcus's Revenge" (Story)
            +-- Scene Beat: "The Warehouse Confrontation" (Story)
```

From the campaign level, "Marcus's Revenge" is just a beat. From Marcus's perspective, it's an entire arc with its own structure. There is no separate StoryBeat Kind — the distinction is purely contextual.

### 6.2 Story Fields

```yaml
kinds:
  Story:
    description: "A narrative arc, quest, project, or goal (recursively nestable)"
    fields:
      summary:
        type: string              # Short text for Feed display

      owners:
        type: ref[]
        target_kind: [Character, Faction]
        # Tagged refs: 'primary', 'founder', 'antagonist', 'lead', 'saboteur'

      involved_characters:
        type: ref[]
        target_kind: Character

      related_stories:
        type: ref[]
        target_kind: Story

      related_events:
        type: ref[]
        # Auto-populated when Events target this Story

    computed:
      active_children:
        expr: find('Story').where(parent == self).where(tags contains 'active')
```

### 6.3 Polymorphic Ownership

A Story can have multiple owners of different types (Characters, Factions) via tagged refs. The Lampblacks' faction project might have:
- The faction as `['primary']` owner
- Three characters tagged as `['lead']`
- One character tagged as `['saboteur']`

This captures *who* is involved and *how*, not just that they're connected.

### 6.4 Auto-Linked Event History

Events targeting a Story automatically populate `related_events` via `ref.add`. This means every Story accumulates a complete history of events that shaped it without manual bookkeeping.

EventTypes can opt out with `link_to_story: false` for administrative or high-frequency events that would clutter history. System events do not auto-link.

### 6.5 Lifecycle via Tags

Stories use tags rather than formal states for lifecycle management:
- `active` — in progress
- `completed` — finished successfully
- `abandoned` — gave up
- `on-hold` — paused

This keeps the model simple and extensible — any tag vocabulary works.

### 6.6 Narrative Gate Pattern

Stories enable a progression pattern that requires both mechanical progress AND narrative completion:

```yaml
actions:
  AdvanceFactionTier:
    preconditions:
      - expr: self.spec.project_progress >= 8
      - expr: find('Story').where(owners contains self).where(tags contains 'completed').any()
```

This enforces that players must both accumulate mechanical progress (fill the clock) AND wrap up the narrative (complete a Story about the advancement) before advancing. Pure numbers aren't enough; the story has to make sense.

---

## 7. Triggers and Cascades

Triggers are declarative rules that watch for specific event types and automatically emit new events in response. They are how OPS gets emergent, systemic gameplay without writing procedural code for every interaction.

### 7.1 Trigger Structure

A trigger says: "When *this kind of thing* happens, if *these conditions hold*, then *do this other thing*."

```yaml
FactionAdvanceOnDowntime:
  watches: DowntimePasses              # semantic event type to match
  condition: target.spec.active == true # optional guard
  emits: ProgressFaction               # what to fire
  for_each: find('Faction').where(active == true)  # iteration
```

**Match** — the primary condition, usually an EventType name. Can also filter by scope, tags, parameters, or custom expressions.

**Guard** — additional conditions that must be true for the trigger to fire. Prevents triggers from activating inappropriately. Has access to game state (e.g., "only if meter X > 5").

**Emission** — what to produce. Can emit direct Events (immediate, no approval) or Drafts (GM reviews before Events are created). Configurable per trigger.

### 7.2 Cascades

When a trigger fires and emits a new event, that event can itself match other triggers. This chain is a **cascade**:

```
GM invokes "Start Downtime"
  -> DowntimePasses signal event (depth 0)
    -> Trigger: FactionAdvanceOnDowntime
      -> ProgressFaction for Fog Hounds (depth 1)
      -> ProgressFaction for Red Sashes (depth 1)
        -> Trigger: FactionGoalComplete
          -> CompleteFactionProject for Red Sashes (depth 2)
            -> ObjectCreated: new Turf GameObject (depth 3)
              -> [no more triggers match — cascade ends]
```

One player action ("start downtime") produced a cascade of five events across three depth levels. The player pressed one button. The game world responded systemically.

### 7.3 Cascade Safety

Unbounded cascades are infinite loops. OPS enforces limits:

- **Max depth** — cascades cannot exceed N levels deep (e.g., 10)
- **Max emissions** — a single trigger firing cannot emit more than M events
- **Self-trigger prevention** — a trigger cannot re-trigger itself in the same cascade

When a limit is hit, the cascade halts and the system logs a warning.

### 7.4 Compositional Game Mechanics

Triggers make game mechanics **compositional**. You define small, focused rules:
- "When a character dies, drop their inventory"
- "When an item is dropped in a magic zone, it transforms"
- "When a transformed item appears, nearby NPCs react"

Each rule is simple. Together, they produce emergent behavior that no single rule anticipated. A character dies in a magic zone -> their sword drops -> it transforms into a cursed blade -> NPCs flee. You didn't script that sequence. The system composed it from three independent rules.

---

## 8. Actions, Drafts, and Approval

Actions are how users express intent to change the world. The Draft/Approve/Commit flow ensures that changes are vetted before becoming permanent Events.

### 8.1 ActionDefs — Declared Intents

An ActionDef is a declarative definition of what an action *is*, defined in Paradigms:

- **Targets** — which Kind(s) this action applies to (bidirectional: Actions target Kinds AND Kinds list Actions)
- **Preconditions** — conditions that must be true (using the Expression DSL)
- **Parameters** — user-editable values with types, defaults, and validation
- **Effects** — EventType templates to propose
- **Approval policy** — how approval is determined

**Action Offers** (what actions are available on a given object right now) are computed at runtime from ActionDefs + current state, never stored.

### 8.2 Drafts — Editable Proposals

When an action is invoked, a Draft is created. Drafts are Workflow Domain entities — they capture user intent, hold editable parameters, and manage approval workflow. Key properties:

- **Editable** — parameters can be adjusted before approval
- **Never affect projections** — drafts don't change world state
- **Contain proposed events** — what will happen if approved
- **Have lifecycle** — `pending` -> `approved`/`rejected` -> `committed`

### 8.3 The Full Loop

```
Action invoked (player or GM clicks something)
  -> Draft created (editable, cancellable)
    -> Approved (by GM, by game rules, or auto-policy)
      -> Events committed (immutable, permanent)
        -> Operations applied (spec fields change, status marked dirty)
          -> Triggers fire (matching events produce new events)
            -> Cascade until quiescence or safety limit
              -> Projections updated (status recomputed)
```

**Auto-approval** is the degenerate case: the draft is created and immediately approved in the same tick. You still get the event log, the causality chain, and the replay — you just skip the human wait.

Different games need different levels of control:
- **Tabletop RPG**: GM approves or rejects player actions
- **Turn-based**: Actions queue during planning, commit on turn resolution
- **Real-time**: Most actions auto-approve; some (trades, guild actions) require confirmation

The pattern is identical; the tick rate and approval speed differ.

---

## 9. Projections — Derived Read Models

A Projection is a read-optimized view computed entirely from the event log. The current state of any GameObject is a projection.

### 9.1 How Projections Work

```
Event Log (source of truth):
  evt_001: object.create Character { name: "Vex", hp: 10, constitution: 3 }
  evt_002: meter.delta Character/Vex { meter: hp, delta: -4 }
  evt_003: object.update Character/Vex { path: spec.name, value: "Vex'ahlia" }

Current State Projection:
  Character/Vex'ahlia: { hp: 6, constitution: 3, max_hp: 14 }
```

### 9.2 Key Properties

- **Always rebuildable** from events — projections are caches, not sources of truth
- **Can be optimized per use case** — full state, activity feed, aggregate counts, relationship graphs
- **Status fields recompute lazily** — marked dirty when underlying spec changes, recomputed on next read
- **Never stale** — synchronous recomputation on first read after invalidation

### 9.3 Feeds

Feeds are a special projection — filtered lists of activity updates for specific audiences:
- **GM Feed**: comprehensive view of all campaign events
- **Player Feed**: filtered to global events, bonds, characters, groups, and sessions the player is part of
- **Object Feed**: history of all events affecting a specific entity

Feeds are the primary player-facing view of game activity. Players typically interact with the world through Feeds rather than directly inspecting GameObject data.

---

## 10. Paradigms — Content Packaging

Paradigms are modular, shareable packages that define complete game systems. They are the mechanism by which OPS achieves system-agnosticism — the engine is generic, and Paradigms supply the game-specific content.

### 10.1 What a Paradigm Contains

| Component | Purpose |
|-----------|---------|
| **Kinds** | GameObject schemas (Character, Faction, Location, etc.) |
| **EventTypes** | Named event schemas (TakeDamage, DowntimePasses, etc.) |
| **ActionDefs** | Available actions with targets, preconditions, effects |
| **Triggers** | Reactive rules (when X happens, do Y) |
| **Default Objects** | Optional seed data created when a Game starts |

Every Paradigm must define a **Game Kind** — the schema for the campaign instance itself.

### 10.2 YAML-Defined

Paradigms are defined entirely in YAML. No code changes needed to:
- Add a new type of character, faction, or entity
- Define a new action or event type
- Create reactive triggers
- Seed a game with starting objects

```yaml
# Example: default objects in a Paradigm
default_objects:
  - kind: Game
    id: $GAME_ID
    spec:
      name: "New Campaign"
      paradigms: [blades-in-the-dark]

  - kind: Faction
    spec:
      name: "The Bluecoats"
      tier: 3
      hold: strong

  - kind: Faction
    spec:
      name: "The Hive"
      tier: 2
      hold: weak
```

### 10.3 Composition

- **Multiple paradigms per game** — combine paradigms if their definitions don't conflict
- **Dependencies** — paradigms can depend on other paradigms
- **Conflict detection** — same Kind name, same ActionDef ID, etc. detected at load time

### 10.4 House Rules as Overlays

House rules are implemented as a Paradigm overlay — local modifications to an existing Paradigm. A house rule Paradigm can modify Kinds, add new EventTypes or Triggers, or override defaults. House rule Paradigms can be published as new Paradigms for others to use.

### 10.5 Core Paradigm

A required Paradigm that defines system EventTypes (MeterOverflow, MeterUnderflow, ObjectCreated, ObjectArchived, RefDangling). Implicitly included in all Games.

---

## 11. Expression and Query DSL

OPS includes a simple domain-specific language used throughout the system for computed values, conditions, and queries.

### 11.1 Expression Features

```
# Path references
self.spec.constitution
self.status.max_hp
target.spec.hp

# Arithmetic
self.spec.constitution * 3 + 5

# Comparisons
self.spec.hp < self.status.max_hp
self.spec.stress >= 6

# Logical operators
x and y
x or y
not x

# Conditionals
if self.spec.hp > 0 then 'alive' else 'dead'

# Functions
len(self.spec.bonds)
min(self.spec.hp, 10)
max(x, y)
floor(x)
ceil(x)
sum(self.spec.bonds.*.spec.loyalty)

# List membership
target in self.spec.allies

# Cross-object traversal
self.spec.bonds.*.spec.loyalty           # list of values
sum(self.spec.faction.spec.members.*.spec.influence)
```

### 11.2 Query DSL

Projection-style chainable queries for finding and filtering GameObjects. Available in all expression contexts.

```
find('<Kind>').where(<condition>).where(<condition>)...
```

**Examples:**
```
# Find all Stories owned by the current object
find('Story').where(owners contains self)

# Find active Stories for a character
find('Story').where(owners contains self).where(tags contains 'active')

# Find Locations with a specific parent
find('Location').where(parent == some_city)

# Find Characters in a faction
find('Character').where(faction == target)
```

**Operators:** `==`, `!=`, `<`, `>`, `<=`, `>=`, `contains`, `in`

**List predicates** (`any`, `all`, `none`) for complex filtering:
```
owners.any(o => o.tags contains 'primary')
bonds.all(b => b.kind == 'Character')
tags.any(t => t startsWith 'role-')
```

**Query results** can be:
- Checked for existence: `.any()` returns boolean
- Counted: `len(find(...))` returns integer
- Iterated: `.*.field` traverses into matched objects

**Example computed status fields:**
```yaml
computed:
  active_story_count:
    expr: len(find('Story').where(parent == self).where(tags contains 'active'))

  has_completed_project:
    expr: find('Story').where(owners contains self).where(tags contains 'completed').any()

  member_names:
    expr: find('Character').where(faction == self).*.name
```

### 11.3 Game Isolation Invariant

**All queries are scoped to the current Game.** Cross-game queries are forbidden. Games are strict isolation boundaries. This is a permanent invariant, not a limitation to be relaxed later.

### 11.4 Expression Constraints

- **No time references** — expressions cannot access `now` or timestamps; time-based logic is handled by Events/Triggers
- **Null handling** — return null when paths don't resolve; GM is notified
- **Archived refs** — skipped in traversal by default
- **Deterministic** — same inputs always produce same outputs

### 11.5 Where Expressions Are Used

- **Meter bounds** — `max: self.status.max_hp`
- **Computed status fields** — `expr: self.spec.constitution * 3 + 5`
- **Trigger conditions** — `condition: target.spec.active == true`
- **Action preconditions** — `expr: self.spec.project_progress >= 8`
- **Conditional EventType operations** — choose which operations to perform based on state

---

## 12. Key Design Patterns

These patterns recur across OPS implementations. They emerge naturally from the primitives described above.

### 12.1 Narrative Gate

Require both mechanical progress AND narrative completion for advancement:

```yaml
preconditions:
  - expr: self.spec.project_progress >= 8           # mechanical
  - expr: find('Story').where(owners contains self)
          .where(tags contains 'completed').any()    # narrative
```

Pure numbers aren't enough; the story has to make sense. This pattern bridges the gap between crunchy mechanics and narrative fiction.

### 12.2 Nestable Kinds

Any Kind can enable recursive hierarchies by constraining `parent` to itself:

```yaml
# Story nests within Story
Story:
  fields:
    # parent: {type: ref, target_kind: [Story]}

# Location nests within Location
Location:
  fields:
    # parent: {type: ref, target_kind: [Location]}
```

Uses the reserved `parent` field — no special flag required. Nestability is a pattern, not a feature.

### 12.3 Computed Collections

Use Query DSL in status fields to dynamically find related objects:

```yaml
computed:
  active_children:
    expr: find('Story').where(parent == self).where(tags contains 'active')
  faction_members:
    expr: find('Character').where(faction == self)
```

These provide live views of related data that update automatically as the world changes — no manual bookkeeping.

### 12.4 Tagged Refs for Relationships

Use tags on refs to capture relationship roles without needing separate join entities:

```yaml
owners:
  - target: char_001
    tags: ['primary', 'founder']
  - target: faction_002
    tags: ['supporting']
```

"Who is the primary owner?" becomes a tag filter. Relationship types, role annotations, and provenance tracking all use the same mechanism.

### 12.5 Signal Events

Broadcast that something happened without directly changing anything. Target the Game object (since it's a GameObject too):

```yaml
DowntimePasses:
  targets:
    primary: Game
  operations: []    # Signal only
```

Triggers listen for the signal and produce the actual effects. This decouples the announcement from the consequences, making each independently modifiable.

### 12.6 Overflow/Underflow as Triggers

Instead of special-casing boundary conditions in code, use the meter system's automatic events:

- "When HP hits 0" = trigger on MeterUnderflow where meter is `hp`
- "Convert excess healing to shields" = trigger on MeterOverflow where meter is `hp`
- "When project clock fills" = trigger on MeterOverflow where meter is `project_progress`

Boundary reactions are just triggers, not special cases.

### 12.7 Bidirectional Action Binding

Actions target Kinds AND Kinds list Actions. Available actions on an object = the union of both:

```yaml
# ActionDef targets Kinds
HealCharacter:
  targets:
    primary: Character

# Kind lists Actions
Character:
  actions:
    - HealCharacter
    - TakeDamage
```

This gives flexibility — either side can declare the relationship.

---

## Appendix A: Glossary of OPS Terms

| Term | Definition |
|------|-----------|
| **GameObject** | Any entity in the game world, defined by a Kind |
| **Kind** | Schema declaring a GameObject's shape, field types, computed properties, and available actions |
| **Spec** | The authored, event-sourced fields of a GameObject (source of truth) |
| **Status** | Derived/computed fields, recalculated from spec (cache, not truth) |
| **Meter** | A bounded numeric field that emits system events on overflow/underflow |
| **Ref** | A typed pointer from one GameObject to another (singular or list) |
| **Tagged Ref** | A ref with optional string tags for role annotation and relationship metadata |
| **Event** | An immutable record of something that happened, carrying a semantic type and a list of operations |
| **EventType** | A named schema for a category of events (e.g., "TakeDamage", "DowntimePasses") |
| **Operation** | One of eight atomic mutation primitives (create, update, archive, meter.delta/set, ref.set/add/remove) |
| **Signal Event** | An Event with no operations, used for broadcasting (e.g., DowntimePasses targeting Game) |
| **Trigger** | A data-driven rule: "when EventType X occurs, if condition Y holds, emit EventType Z" |
| **Cascade** | A chain of trigger firings from a single root event, with depth tracking and safety limits |
| **ActionDef** | A declarative definition of an available action with targets, preconditions, and effects |
| **Draft** | A pending, editable proposal that becomes an Event upon approval |
| **Projection** | A read-optimized view derived entirely from the event log (the "current state") |
| **Feed** | A filtered projection of activity updates for a specific audience (GM, player, object) |
| **Paradigm** | A modular package of Kinds, EventTypes, ActionDefs, Triggers, and seed data |
| **House Rules** | Local modifications to a Paradigm, implemented as a Paradigm overlay |
| **Core Paradigm** | Required Paradigm defining system EventTypes (MeterOverflow, etc.) |
| **Trait** | A narrative-mechanical descriptor on an entity — personal attribute, training, or bond |
| **Perk** | A conditional special rule attached to a Trait (Basic: automatic; Advanced: purchasable) |
| **Narrative Gate** | A progression pattern requiring both mechanical progress and narrative completion |
| **Nestable Kind** | A Kind where objects can parent other objects of the same Kind (recursive hierarchy) |
| **Computed Collection** | A status field using Query DSL to dynamically find related objects |
| **Expression DSL** | The domain-specific language used for computed values, conditions, and queries |
| **Query DSL** | The `find().where()` chainable query syntax within the Expression DSL |
| **ULID** | Universally Unique Lexicographically Sortable Identifier — used for Event and GameObject IDs |
| **Causality** | Metadata on each event tracking its parent event, root action, and cascade depth |
| **Dangling Reference** | A ref pointing to an archived GameObject — flagged for GM resolution |

---

## Appendix B: Comparing OPS Across Domains

OPS patterns are engine-agnostic. Here's how they map to other implementations:

| OPS Pattern | Tabletop RPG (Reality Engine) | Video Game | Board Game |
|-------------|------------------------------|------------|------------|
| Kind | YAML schema in Paradigm | Prefab / ScriptableObject / Entity archetype | Card type definition |
| GameObject | Campaign entity (character, faction, etc.) | Entity / Actor / Node | Token / Card instance |
| Spec fields | Authored data on game objects | Component data | Printed stats |
| Status fields | Computed properties, updated on read | Cached computed properties, updated per tick | Derived values |
| Meter | HP, stress, project progress | Health, mana, stamina, cooldown | Track / counter |
| Ref | Faction membership, bonds | Entity reference, target lock | Attached / assigned cards |
| Event | Immutable log entry | Command object, replay buffer entry | Game log entry |
| Trigger | Reactive rule in Paradigm YAML | Observer/subscriber (data-driven, logged) | Triggered ability |
| Cascade | Downtime -> faction progress -> project complete | Damage -> death -> inventory drop -> transform | Chain reaction |
| Draft | GM approval queue | Input buffer / server validation | Pending action |
| Projection | Current game state view | Live ECS world snapshot | Board state |
| Paradigm | Game system YAML (Blades in the Dark, etc.) | Mod / data pack / content module | Expansion set |
| Trait | Character descriptor with Perks | Ability / talent tree node | Card effect |
| Perk | Conditional special rule on a Trait | Passive / active skill | Special ability |

---

_This document was extracted from the Reality Engine project specifications and archived game design documents. It captures the Open Paradigm System as expressed through a tabletop RPG campaign management platform._
