# OPS Game Design Principles

**How the Open Paradigm System thinks about game design.**

This document describes the design philosophy behind OPS — the recurring patterns that make OPS games feel coherent, extensible, and surprising. It's written for game designers, not engineers. If you want the technical architecture, read `ops.md`. This document is about *why* these patterns produce good games and how to apply them when designing new ones.

The principles here are distilled from multiple OPS projects across different media: tabletop RPGs, auto-battlers, narrative engines, and idle games. The specific mechanics differ wildly. The underlying design thinking is the same.

---

## The Core Bet

OPS makes one foundational bet: **a game is more interesting when its complexity emerges from simple rules interacting than when it's authored interaction by interaction.**

This means designing small, composable pieces — entity types, tags, triggers, meters — and letting them combine into behavior you didn't explicitly program. A character dies in a magical zone, their sword drops, the zone transforms it into a cursed blade, nearby NPCs sense the curse and flee. No one scripted that sequence. Three independent rules, each simple on its own, composed it automatically.

The designer's job in OPS is to create the vocabulary (the pieces and the rules for how they connect) and then curate the emergent results. Not to hand-author every possible interaction.

---

## Structural Foundations

These aren't design choices — they're the load-bearing structure that makes everything else work. Violating them doesn't just break a feature; it undermines the entire system's ability to produce emergent, trustworthy behavior.

### Everything Is a Record

All state changes are recorded as immutable events in an append-only log. This isn't a technical detail — it's a design affordance. Because everything is recorded:

- Players (and GMs, and the system) can always answer "why did this happen?" by tracing the event chain.
- The game can be rewound, replayed, or audited.
- Consequences feel real because they're permanent and traceable, not silently overwritten.

When designing mechanics, think about the events they produce. A well-designed mechanic tells a story through its event trail. If you can't describe what events a mechanic produces, the mechanic isn't well-defined yet.

### Authored State vs. Derived State

Every entity in OPS splits its data into two layers: **spec** (what was deliberately set) and **status** (what's computed from it). A character's constitution is spec — someone set it. Their max HP is status — it's calculated from constitution.

This matters for game design because it forces clarity about what's a *decision* and what's a *consequence*. When a player levels up constitution, max HP follows automatically. When an equipment bonus changes, every derived stat updates. The designer never has to worry about values drifting out of sync because the system makes desync structurally impossible.

**Design rule**: If a value can be computed from other values, it should be. The only things in spec should be genuine inputs — the irreducible decisions that everything else flows from.

### Eight Ways to Change the World

All mutations in OPS reduce to eight atomic operations: create an entity, update a field, archive an entity, adjust a meter, set a meter, set a reference, add a reference, remove a reference. That's it. Every mechanic — combat, crafting, dialogue, faction politics, weather — decomposes into some combination of these eight.

This constraint is liberating, not limiting. Because the operation vocabulary is fixed, adding a new entity type, event type, or mechanic never requires new mutation code. The designer can invent freely knowing that the system can already execute whatever they dream up, as long as it can be expressed as "change these numbers, set these references, create these things."

### Propose, Then Commit

State changes go through a proposal-and-approval workflow before they become permanent. In a tabletop RPG, this is the GM reviewing a player's action. In a video game, it might be the server validating a client's input. In a single-player game, it might be instant auto-approval.

The pattern is identical at every speed. Design mechanics with gating in mind — who approves this? Under what conditions? What can be modified before approval? The answers change per medium, but the question is always relevant.

---

## Composition: Building Blocks, Not Blueprints

### Entities Are What They Contain

In OPS, you don't define a "Warrior" class that inherits from "Character" with a "MeleeAttack" method. You define a character who has warrior-like traits, equipment, and perks. A "temple" and a "tavern" are both groups — one offers healing services, the other offers rumors. A "potion" and a "permanent perk" use the same effect component model; they differ only in duration.

This means new content is always *new data*, never *new code*. Want a new species? It's a new trait with new tags. Want a new building type? It's a group with new services. Want an entirely new category of magic? It's new perks referencing a new resource type, which the system discovers automatically when a character first acquires one.

**Design rule**: If adding a new variety of something requires anything other than new data entries in existing structures, the structure needs to be more general.

### One Effect Model to Rule Them All

All game effects — whether they come from perks, equipment, consumables, environmental hazards, or status conditions — share a single typed format. A fire damage effect works the same way whether it comes from a sword enchantment, a spell, a potion, or a trap. The source differs; the mechanical expression is identical.

This eliminates an entire class of design bugs: "this effect works when it comes from a perk but not from equipment" can't happen because both use the same model. It also means cross-source synergies work automatically: a "+10% fire damage" bonus applies to ALL fire damage, regardless of origin.

### Tags: The Shared Vocabulary

Tags are simple keywords that live on game content — [Fire], [Healing], [Stealth], [Divine], [Undead]. They form a shared vocabulary across all systems. When a perk grants "+10% damage to [Fire] Actions," it doesn't know or care where those fire actions come from. Any new content tagged [Fire] automatically benefits.

This is how OPS achieves combinatorial complexity without combinatorial authoring effort. The designer creates individual pieces. Tags make those pieces interact with everything that shares their vocabulary. Adding ten new [Fire]-tagged abilities doesn't require updating any existing bonuses — they just work.

**Design rule**: When two systems need to interact, prefer tag-based coupling over explicit references. "Bonus against [Undead]" is better than "bonus against Skeleton, Zombie, Vampire, Ghost" because it automatically includes every [Undead] creature that will ever exist.

---

## Emergent Mechanics: Rules That Surprise You

### Meters as Boundaries, Not Bookkeeping

A meter isn't just a number with a min and max. It's a *reactive boundary*. When a meter hits its limit — HP reaches 0, a progress clock fills up, stress overflows — the system broadcasts an event. That event can trigger anything.

This turns what would normally be special-case code ("if HP <= 0, die") into general-purpose reactive design. Death is just a trigger on meter underflow. Clock completion is just a trigger on meter overflow. "Convert excess healing to shields" is just a trigger on HP overflow. The designer defines the boundary; the trigger system handles the consequences.

**Design rule**: Whenever something interesting should happen at a numeric threshold, use a meter. Don't check the value manually — let the overflow/underflow system detect it and react.

### Triggers and Cascades: Cause and Effect Chains

A trigger is a simple rule: "When X happens, if Y is true, do Z." Triggers are data, not code. They watch for events and emit new events in response. When a trigger fires and produces a new event, *that* event can match *other* triggers, creating a cascade.

One player action ("start downtime") might produce a cascade: downtime passes → factions advance → one faction completes a project → a new territory is created → nearby NPCs react. The player pressed one button. The world responded systemically.

This is where OPS games get their feeling of being *alive*. The designer didn't script that downtime sequence. They wrote five independent rules, each trivially simple, and the cascade composed them into a narrative event that surprised even the designer.

**Design rule**: Prefer many small, independent triggers over few large, complex ones. Complexity should emerge from composition, not from individual rule complexity. And always cap cascade depth — unbounded chains are infinite loops.

### Signal Events: Announce, Then React

Sometimes you need to broadcast that something happened without prescribing what should follow. A signal event carries no operations — it changes nothing directly. Its sole purpose is to be heard by triggers.

"A character entered a sacred grove" is a signal. The signal itself does nothing. But a trigger that watches for grove entry might grant a blessing. Another might alert the grove's guardian. Another might log it in the character's journal. Each reaction is independent. The signal decouples the announcement from every possible consequence.

**Design rule**: When an event's consequences are context-dependent or likely to expand over time, make it a signal. Let triggers handle the specifics. This way, adding new reactions never requires modifying the original mechanic.

---

## Player Experience Patterns

### One Scale, One Language

OPS games use a universal rating scale across all systems. Whether you're looking at a character's star rating, an equipment quality tier, a faction's power level, or a quest's difficulty, the same 0-5 scale (or whatever range the project chooses) applies everywhere with consistent visual language.

Players learn one vocabulary and can immediately reason about relative quality in any context. "That's a 3-star sword from a 4-star blacksmith for my 2-star character" is instantly parseable because the scale means the same thing everywhere.

**Design rule**: Resist the temptation to create system-specific scales. When a new system needs a quality dimension, adopt the existing universal scale. If you find yourself writing a custom 1-100 quality range for equipment while using a 1-5 scale for characters, stop — unify them.

### One Formula, Many Prices

Where possible, a single valuation formula should drive multiple economic systems. A character's "value" determines their hiring cost, their upkeep, their retirement payout, and their contribution to services. Change the formula once; every economic system adjusts automatically.

This creates economic coherence that players can intuit even if they never see the formula. "Expensive characters are expensive to maintain but valuable when retired" makes holistic sense. It also prevents the common design bug where one economic system is exploitable because it uses a different valuation than the others.

### Power Plateaus, Not Power Creep

OPS games prefer lateral growth (new capabilities, new options, new roles) over vertical power creep (ever-increasing numbers). Power should plateau, and cycling mechanics (retirement, turnover, prestige resets) should keep the game fresh.

When a character reaches their mechanical ceiling, the interesting question shifts from "how do I get stronger?" to "what do I do with this strength?" and eventually "when do I gracefully transition out to make room for something new?" This keeps long-running games healthy by preventing the numbers from spiraling beyond the system's ability to challenge.

### Narrative Gates

Mechanical progress alone shouldn't be sufficient for advancement. The story has to make sense too. A character shouldn't become a guild leader just because they ground enough XP — they should also have done something in the narrative that earns it. Progress clocks fill through play, but completion requires narrative resolution.

This is the "narrative gate" pattern: both the numbers AND the story must align for advancement. It prevents the feeling of a game being purely mechanical while still giving the mechanical systems meaningful weight.

### Mystery Through Ambiguity

OPS supports leaving state intentionally ambiguous until it's narratively observed. A character's exact location might be fuzzy — they're "somewhere near the docks" based on their relationships, not pinned to a specific coordinate. A group project's outcome is undefined until the progress clock fills and someone narrates the result.

This "deferred resolution" pattern turns the system's uncertainty into a design feature. Mystery, suspense, and surprise become possible because the game genuinely doesn't know the answer until the moment of observation. For tabletop RPGs, this means the GM isn't lying when they say "we'll find out" — the system is honestly in a superposition until collapsed.

---

## Information and Relationships

### Relationships Are Data

Relationships between entities — who belongs to what faction, who owns which items, which locations are connected — are first-class data in OPS, not implicit associations buried in code. Every relationship is a typed reference with optional metadata tags.

This means the game can query, traverse, and reason about its own relationship graph. "Who are the allies of my faction's leader?" is a data query, not a special-case function. Relationships can be created, modified, and removed through the same event system as everything else.

### The Bond Graph as Universal Index

When a single relationship structure (the "bond graph") connects all entities, it becomes a universal index for computing derived systems:

- **Visibility**: Who can see what? Traverse bonds to determine information flow.
- **Presence**: Who is where? Compute from bond distances to locations.
- **Membership**: Who belongs to what? Read from refs with role tags.
- **Influence**: Who affects whom? Weighted bond traversal.

One graph, many queries. The designer defines the bonds; the system derives everything else. This means changes to relationships automatically ripple through visibility, presence, membership, and influence without anyone updating each system individually.

### Context-Dependent Flavor

The same mechanical effect can have different narrative meaning depending on its source. "+2 to intimidation" from a warrior trait means physical menace. "+2 to intimidation" from a noble trait means social authority. "+2 to intimidation" from a cursed item means supernatural dread. The mechanic is identical. The flavor varies per attachment.

This is a content efficiency pattern. Design the mechanic once. Write the narrative wrapper per source. The combinatorial space of mechanics × sources produces far more variety than authoring unique mechanics for each source would.

---

## Resolution and Simulation

### Simulate, Then Present

When a game mechanic needs to resolve an outcome — combat, crafting, a heist — resolve it as a complete deterministic simulation first, producing structured data. Then present the results from that data.

This separation has cascading benefits:
- The same resolution can be presented multiple ways (detailed log, dramatic summary, statistical overview, animation sequence)
- Results are testable and reproducible
- The presentation layer can be redesigned without touching game logic
- Replays are free — you have the data

**Design rule**: If your resolution logic depends on how results will be displayed, the coupling is wrong. The simulation should produce data that is meaningful independent of any particular presentation.

### Hybrid Resolution

Not everything needs the same fidelity. OPS supports mixing abstracted checks (fast, attribute-driven, resolve in one step) with fully simulated encounters (deep, engine-driven, multi-step). A tavern brawl might be one abstracted check. A boss fight might be a full tactical simulation. Both are valid resolution modes within the same system.

**Design rule**: Match resolution depth to narrative importance. Don't simulate what doesn't matter. Don't abstract what the player cares about deeply.

### Template + Procedural

Handcrafted content provides the narrative backbone — the major story beats, the memorable locations, the key NPCs. Procedural generation provides variety and replayability — random encounters, equipment affixes, background NPCs, side quests.

The best OPS designs use templates as anchors and procedural generation as connective tissue. The player gets authored quality for the things that matter most and procedural freshness for everything else.

---

## Content as Portable Packages

### Paradigms

An OPS paradigm is a self-contained package of game content: entity schemas (Kinds), event types, action definitions, triggers, and seed data. It's the unit of portability — you can take a paradigm from one OPS project and drop it into another (assuming compatible schemas).

This means game systems can be designed, tested, and shared independently. A "faction politics" paradigm, a "crafting" paradigm, a "weather" paradigm — each is a module. The game is the combination of its paradigms plus project-specific customization.

### Configurable Event Templates

Event types differ by data configuration, not by code paths. A "combat encounter" and a "social encounter" and a "exploration encounter" might all be the same event template with different phases enabled, different profiles flagged, and different resolution parameters.

**Design rule**: When you find yourself creating a new event type that's structurally similar to an existing one, make the existing one configurable instead. The fewer distinct code paths, the fewer bugs — and the easier it is for content authors to create new event varieties.

---

## Anti-Patterns

These indicate the design is drifting away from OPS principles.

**Hardcoded mechanics** — A special code path for one specific ability, item, or event type. If one fireball spell needs custom logic, the effect model isn't general enough.

**Implicit mutation** — State changes that bypass the event log. If something changes and there's no event record, causality tracking breaks and the audit trail has gaps.

**Inheritance taxonomies** — Deep class hierarchies for entity types. If you need "MeleeWarrior extends Warrior extends Fighter extends Character," you're inheriting when you should be composing.

**Orphan mechanics** — Systems that don't use the tag vocabulary or participate in triggers. If a new system can't interact with existing systems through tags and events, it's an island that adds complexity without emergent potential.

**Vertical power spiral** — Stats that grow without bound and no retirement or cycling pressure. Eventually the numbers become meaningless and the system can't challenge the player.

**Opaque resolution** — Outcomes that players can't trace back to causes. If something happens and no one can explain why, the event trail is insufficient or the presentation isn't exposing causality clearly enough.

**Presentation-coupled logic** — Game rules that depend on how results are displayed. If changing the UI would change the game's behavior, the separation between simulation and presentation is broken.
