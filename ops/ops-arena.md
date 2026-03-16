# OPS Pattern Extraction: Arena

This document distills the **Open Paradigm System (OPS)** design patterns as implemented in the Arena project. OPS is a set of composable game design patterns for building deep, content-driven games with emergent complexity. The patterns emphasize modularity via tags, content-as-data, emergent synergies, and universal scaling systems.

**Purpose**: This extraction is intended for combination with similar extractions from other OPS projects. The combined document will generate an OPS-game-designer persona agent for reviewing and designing OPS-compatible games.

---

## 1. Source Game: Arena

**Genre**: Roguelike idle/auto-battler — a gladiatorial house management game.

**Core loop**: Recruit fighters into a gladiatorial household, optimize builds through deep Trait/Perk systems, manage equipment and economy, configure AI behavior, and watch automated tactical combat unfold.

**Design principle**: Deep build optimization with economic pressure, played in short sessions. Complexity lives in roster and build management, not moment-to-moment reflexes.

**Session model**: Idle-friendly, 2-5 minute productive sessions. Players configure between fights; combat executes automatically without player input.

**Tech**: Python, MUD-style architecture (headless server + TUI client). Deterministic simulation — combat resolves fully before presentation.

**Modes**: Single-player (tick-on-command, play at your own pace) with planned multiplayer expansion (persistent world, real-time ticks, PvP tournaments, cross-player economy).

**Player role**: House manager. Recruits fighters, optimizes builds, configures AI orders, manages economy. Spectates during combat; all decisions happen between fights.

---

## 2. Universal Star Rating System

A single 0-5 scale with consistent color coding applied across all entity types. One scale, one language, universal comprehension.

| Stars | Color  | Tier Name           |
|-------|--------|---------------------|
| 0     | Gray   | Broken / Degraded   |
| 1     | White  | Standard / Common   |
| 2     | Green  | Superior / Uncommon |
| 3     | Blue   | Rare                |
| 4     | Purple | Epic                |
| 5     | Orange | Legendary           |

**How it maps across systems**:

| System      | Star Rating Controls                                           |
|-------------|----------------------------------------------------------------|
| Characters  | Trait slot capacity (1-5 slots per category)                   |
| Traits      | Minimum purchase level; all Traits level to 5                  |
| Perks       | Minimum purchase level; all Perks level to 5                   |
| Equipment   | Quality ranges (0-99 = 0, 100 = 1, ... 500+ = 5)             |
| Groups      | Service quality ceiling, max recruit star, NPC trainer caps    |
| Events      | Difficulty/reward tiers, entry restrictions                    |
| Consumables | Effect magnitude (linear scaling with tier)                    |
| Quests      | Difficulty class defaults, enemy power, reward scaling         |

> **OPS Pattern — Universal Rating Scale**: Apply one numeric scale with consistent visual language across all game systems. Players learn a single quality vocabulary and instantly understand relative power in any context. New systems adopt the existing scale rather than inventing their own.

---

## 3. Game Objects

Entities in an OPS game are defined by **composition** (tags, traits, services) rather than rigid class inheritance.

### Characters — The Primitive Entity

The fundamental game object. Each character has:

- **Attributes**: 9 primary stats on a 0-200 scale (Might, Speed, Accuracy, Endurance, Charisma, Awareness, Intellect, Willpower, Luck). Each has a Current value (usable) and Potential value (trainable maximum). Loose inverse correlation between Current and Potential at generation — veterans start strong but have less growth headroom; raw recruits start weak with high ceilings.
- **Trait Slots**: Three categories (Core, Role, Bond) with slot count equal to Star Rating per category. Each slot holds one Trait of any star level — no budget math.
- **Derived Stats**: Calculated from weighted attribute blends (e.g., Physical Defense = 40% Speed + 35% Accuracy + 25% Awareness). Per-stat scaling multipliers convert blends to game-scale values.
- **State Machine**: Top-level states control availability (Available, In Combat, On Job, On Quest, Recovering, Dead). Mutually exclusive — characters can only do one thing at a time.
- **Anatomical Slots**: Physical equipment locations (Head, Torso, Hands, Feet, etc.). Modifiable by ancestry Traits (species can add/remove/change slot counts).
- **Resource Pools**: Emergent from Perk content. A pool activates when a character first owns a Perk that references that resource. Attribute-derived base capacity + Perk-granted bonuses.

### Groups — Unified Location/Service Entity

All named organizations (guilds, temples, taverns, factions, shops) are a single entity type with a composable service menu:

- **Services**: Vendor, Training, Recruitment, Spellcasting, Crafting, Quests — any combination per Group.
- **Tier**: 1-5 matching the universal scale. Controls service quality ceiling, max recruit star, NPC trainer caps.
- **Theme Tags**: Drive inventory, recruits, and quest content.
- **Public vs. Gated**: Public Groups accessible to all; Bond-gated Groups require a character Bond Trait.
- **Districts**: Groups cluster in hybrid tier + theme zones. Progressive discovery through Bond introductions and information mechanics (Tavern rumors).

### Equipment — Slot-Based Gear

- **Quality System**: 0-500+ numeric quality mapping to the universal star tiers.
- **Affixes**: Diablo-style tiered modifier pools using the unified component model.
- **Degradation**: Permanent wear from use. No quality floor — equipment degrades to 0 (broken). Primary long-term economic sink.
- **Implement Categories**: Parallel weapon (physical) and focus (magical) systems, each with distinct types and formulas.

### Consumables — Single-Use Items

- **Unified Component Model**: Consumables ARE mini-Perks bound to single-use items. Same data model as equipment affixes and Perk components.
- **Tag Integration**: Carry type tags ([Potion], [Bomb], [Scroll]) AND effect-based tags ([Healing], [Fire]). Cross-system synergies work automatically through shared tag vocabulary.

> **OPS Pattern — Composition over Inheritance**: Define entities by what they contain (tags, traits, services, components) rather than what class they belong to. A "temple" and a "tavern" are both Groups with different service menus. A "potion" and a "Perk" use the same effect component model. New entity varieties require only new data, not new code.

---

## 4. Trait System (Core / Role / Bond)

Three categories that partition character identity, capability, and affiliation.

| Category | Represents | Examples |
|----------|-----------|---------|
| **Core** | Who the character IS — innate identity | Species, personality, destiny |
| **Role** | What the character CAN DO — capabilities | Combat classes, crafting specializations |
| **Bond** | Who the character is CONNECTED TO — affiliations | Guild memberships, temple orders |

### Key Mechanics

- **Slot Capacity = Star Rating**: A 3-star character has 3 Core, 3 Role, and 3 Bond slots. Each slot holds one Trait of any star level.
- **Leveling**: All Traits level 1-5. XP cost per level: 100/300/600/1000/1500. Same schedule for Traits and Perks.
- **Trait Level Amplification**: Multiplier applied to all child Perks (1.0/1.2/1.4/1.7/2.0). Stacks multiplicatively with Perk level scaling — a 5-star Trait with a 5-star Perk produces a 4.0x total multiplier.
- **Acquisition-Only Gates**: Requirements (Attributes, other Traits/Perks, derived stats) are checked only at purchase time. Once owned, a Trait is never deactivated by stat changes. This prevents frustrating cascade failures.
- **Cross-Category Sharing**: Identical Perks can appear in different Traits across categories. No stacking — the second source provides redundant access (insurance against respec).

### Category-Specific Design

**Core Traits — Species as Content**: Species/ancestry are Core Traits, not a hardcoded species list. New species = new Core Trait. Hybrid ancestry = stacking multiple ancestry Traits. Personality archetypes are tags on Core Traits feeding the AI system.

**Role Traits — Dual Purpose**: Role Traits enable both combat Perks and Job-enabling Perks (via `job_role` flag). A Blacksmith Trait includes combat Perks AND forge capability. The player chooses whether the character fights or works — mutually exclusive.

**Bond Traits — Sole Relationship Axis**: Bond Trait level IS the character's relationship with a Group. No separate reputation/favor/standing system. Discount-only scaling with optional star-gated exclusive items (3-star) and unique services (5-star).

> **OPS Pattern — Tripartite Identity**: Partition character identity into three orthogonal axes: what they are, what they can do, and who they're connected to. Each axis has its own slot count, creating naturally constrained but deeply combinatorial build space.

---

## 5. Perk Component Model

Perks are multi-component packages that unify all character capabilities into one system.

### Three Component Types

1. **Actions**: Active abilities usable during combat. Defined by target tags, effect list, speed modifier, cooldown (per-combat only), and resource costs.
2. **Stat Adjustments**: Permanent bonuses applied while owned. Can be flat ("+5 Might"), tag-scoped flat ("+5 Soak vs [Fire]"), or tag-scoped percentage ("+10% damage to [Fire] Actions"). Negative values enable built-in tradeoffs.
3. **Triggers**: Conditional automatic effects that fire on specific combat events (OnHit, OnKill, OnFallen, OnTurnStart, etc.). Event type maps to effect list. All qualifying Triggers fire simultaneously with no ordering dependency.

### Shared Effect Component List

All three component types express outputs using a generic typed format:

| Type         | Parameters            | Example                          |
|--------------|-----------------------|----------------------------------|
| Damage       | damage_type, formula  | Deal 20 [Fire] damage            |
| Heal         | formula               | Restore 30 HP                    |
| ApplyStatus  | status_type, stacks   | Apply 3 stacks of Burning        |
| RemoveStatus | status_type, stacks   | Remove 5 stacks of Poison        |
| Move         | target, zones         | Push target 1 zone away          |
| Shield       | formula, duration     | Grant 50-point shield for 3 ticks|
| Summon       | template              | Summon a Fire Elemental          |

New effect types can be added without schema changes — the list format is inherently extensible.

### Level Scaling Rule

Output values (damage, healing, stat bonuses, trigger magnitudes) scale with the Perk/Trait level multiplier. Resource costs, cooldowns, and requirements stay **flat** at all levels. Universal rule — no per-component opt-in/opt-out.

> **OPS Pattern — Unified Component Model**: Express all game effects through a single typed component format shared across Perks, equipment affixes, and consumables. Every effect source uses the same data model, enabling universal tag synergies and eliminating special-case logic for different content types.

---

## 6. Tag System & Emergent Synergies

A shared keyword vocabulary connecting all content systems.

### How Tags Work

- Tags live on individual **Perks** and their component **Actions**.
- **Traits derive tags** from the union of all their Perks' tags (no independent Trait tags).
- **Auto-tags from resources**: Perks that reference a resource type automatically gain the corresponding tag (Mana -> Arcane, Faith -> Divine, Spirit -> Primal, Focus -> Psychic, Stamina -> Martial).
- Equipment and Consumables carry their own tags from the same vocabulary.

### Emergent Synergy Mechanism

Tag-scoped bonuses create cross-system interactions without explicit programming:

- A Perk granting "+10% damage to [Fire] Actions" boosts ANY [Fire]-tagged Action — regardless of which Trait, equipment, or consumable produced it.
- Equipment granting "+5 Soak vs [Poison]" works against any [Poison] source.
- A consumable tagged [Healing] triggers any Perk conditional on [Healing] effects.

### No Explicit Set Bonuses

There are no "if you have Trait X and Trait Y, gain bonus Z" mechanics. All cross-Trait synergies are tag-driven. New content automatically interacts with existing content through shared tags — no coordination between content authors required.

> **OPS Pattern — Tag-Driven Emergence**: Use a shared keyword vocabulary across all systems. Bonuses that reference tags create emergent synergies — any new content carrying existing tags automatically interacts with all existing tag-scoped bonuses. Complexity grows combinatorially with content volume, not with explicit interaction authoring.

---

## 7. Resource Pools

An open, extensible system where new resource types are content data, not schema changes.

### Core Mechanics

- **Pool activation is emergent**: A resource pool appears when a character first owns a Perk that references that resource. The pool disappears when no Perks reference it.
- **Attribute-derived base**: Each resource type defines a weighted attribute formula for base capacity (e.g., Mana = 60% Intellect + 25% Willpower + 15% Awareness).
- **Bonus capacity**: Perk Stat Adjustments can add bonus pool capacity, applied pre-multiplier for amplification.
- **Shared pool**: All Perks referencing the same resource type share one pool per character.
- **Multi-resource support**: A single Action can cost resources from multiple pools.

### Default Resource Types

Five ship as defaults, each with a canonical formula and auto-tag. Content authors can define additional types with custom formulas beyond this set.

| Resource | Primary Attribute | Auto-Tag |
|----------|------------------|----------|
| Mana     | Intellect        | Arcane   |
| Faith    | Willpower        | Divine   |
| Spirit   | Endurance        | Primal   |
| Focus    | Awareness        | Psychic  |
| Stamina  | Willpower/Endurance | Martial |

> **OPS Pattern — Emergent Resource Ownership**: Define resource types as content data with attribute-derived formulas. Pool activation emerges from Perk ownership — no explicit "this character has mana" assignment. The first Perk referencing a resource creates the pool; removing all such Perks removes it.

---

## 8. Combat Model

Tick-based deterministic simulation with structured output.

### Tick Resolution Order

Each combat tick resolves in a fixed five-phase sequence:

1. **Effects Phase**: Per-tick status effects resolve (DoT damage, stack decay, regen, shield expiry). All characters process simultaneously.
2. **Initiative Phase**: All characters add `sqrt(Speed) x global_multiplier` to their Initiative meter.
3. **Turns Phase**: Characters with Initiative >= 100 act in global Initiative order (highest first). Characters evaluate and act **sequentially** — board state updates after each action.
4. **Fallen Resolution**: Pure HP check — all characters below 1 HP enter Fallen state. Healing received during the tick can prevent this.
5. **Kill Triggers**: Batch-fire all kill-related triggers for characters who entered Fallen.

### Key Design Decisions

- **Diminishing returns on Speed**: Square root scaling means doubling Speed yields ~41% more Initiative, not 100%. Prevents speed from dominating.
- **Full-tick death deferral (Dying Blows)**: Characters reduced below 0 HP are NOT immediately removed. They finish the full tick — can accumulate Initiative and take their turn. Actions at <= 0 HP are "Dying Blows," explicitly tracked. Creates dramatic last-stand moments.
- **Zone-based spatial model**: Binary ranges (Short = same zone, Medium = adjacent, Long = non-adjacent). No damage falloff. Movement costs a full turn, making positioning genuinely strategic.
- **Action Speed modifier**: Flat modifier to base -100 Initiative cost. Fast actions (+30 = only -70 Initiative) let characters act sooner; slow actions (-40 = -140 Initiative) impose delay. Creates meaningful tradeoffs between power and tempo.
- **Deterministic simulation**: Seeded PRNG produces identical results from identical inputs. Enables replays, testing, and verification. Combat resolves entirely before presentation.
- **No in-combat healing by default**: All healing comes from Perks, consumables, or equipment effects. No natural regeneration. Makes healing a deliberate build investment.

### Stack-Based Status System

All temporary effects use a unified system:

- **Stacks = duration**: No separate duration field. Per-status decay formulas reduce stacks each tick. When stacks reach 0, status is removed.
- **Additive stacking**: New applications add stacks to existing count.
- **Minimum 1 stack**: Status effects always apply at least 1 stack after Resistance reduction. No true immunity — high Resistance + fast decay provides functional near-immunity.
- **Per-status decay rules**: Percentage-based, attribute-derived, flat, or special conditions. Each status type defines its own decay formula as content data.

### Structured Output

- **Combat Event Stream**: Typed events emitted during simulation (AttackEvent, DamageEvent, FallenEvent, etc.) form the authoritative combat log.
- **Combat Scoreboard**: Per-character stat block (damage dealt/taken/healed, kills, actions by type, ticks survived, dying blows). Feeds post-combat systems and meta-balance tracking.
- **Combat Context Flags**: Per-fight metadata (tournament round, exhibition status, PvE tier, team sizes, attrition parameters). Any system can query these flags to adjust behavior by context.

> **OPS Pattern — Simulation-First Combat**: Resolve combat as a deterministic simulation producing structured data. Present results from the data. This separates game logic from presentation, enables replays, supports multiple presentation modes (detailed logs, dramatic summaries, statistics), and makes the system testable.

---

## 9. AI System (Utility AI)

Content-authored AI behavior embedded in game data, not hardcoded.

### Gate + Scorer Model

Each turn, the AI evaluates every available Action:

1. **Gate check**: Binary prerequisites (resource cost, cooldown, range, valid targets). Any Gate returning 0 vetoes the Action.
2. **Score calculation**: Remaining Actions scored on two tracks.
3. **Selection**: Weighted random from top-scoring Actions with controlled sharpness.

### Two Score Tracks

- **Tactical Score**: Objective effectiveness — damage output, healing urgency, resource efficiency, AoE clustering, target vulnerability. What SHOULD be done.
- **Personality Score**: Character temperament — aggression preference, protective preference, showoff preference. What the character WANTS to do.

### Judgment Stat Blends the Tracks

A derived stat (40% Awareness + 35% Willpower + 25% Intellect) controls three parameters:

- **Tactical-vs-Personality blend**: Low Judgment (~0.3) = personality-driven, impulsive decisions. High Judgment (~0.95) = near-optimal tactical play.
- **Selection sharpness**: Low Judgment = more random selection from scored Actions. High Judgment = reliably picks the top scorer.
- **Lookahead depth**: High Judgment characters project future game state (Effects Phase, Initiative timing) to make better decisions.

### Content-Authored AI Configs

Each Action includes an `ai` block defining its Gates, Tactical Scorers, and Personality Scorers. Content authors select from a standard Consideration library. This means AI behavior is part of the game data, not the engine — new Actions bring their own AI evaluation logic.

> **OPS Pattern — Data-Driven AI**: Embed AI evaluation logic in content definitions alongside mechanical effects. Each Action knows how the AI should evaluate it. Intelligence is a character stat, not a system constant — low-intelligence characters make personality-driven mistakes; high-intelligence characters approach optimal play.

---

## 10. Event System & Post-Combat

Configurable event templates with per-event post-combat profiles.

### Events as Templates

All combat events (tournaments, PvE encounters, quests, exhibitions) are configurable templates sharing a common structure:

- **Post-combat profile**: Per-event flags controlling which phases fire (injury_enabled, discovery_enabled, recruitment_enabled, loot_enabled, xp_enabled).
- **Reward scaling by placement**: 1st = 100%, 2nd = 60%, 3rd = 40%, 4th+ = 25%. Injury and Perk Discovery are not scaled — consequences apply at full rate regardless of placement.
- **Context Flags**: Per-event combat configuration (attrition ramp onset, team sizes, consumable rules).

### Post-Combat Phased Presentation

A sequential dramatic flow, not a bulk summary:

1. **Combat Resolution**: Final blow, victory/defeat, summary statistics.
2. **Injury & Death Phase**: Fallen characters' fates revealed one by one. Tiered system: injury chance from overkill magnitude vs Injury Resistance, then weighted severity roll (Minor -> Major -> Critical -> Death).
3. **Perk Discovery Phase**: Per-qualifying-Trait end-of-combat roll. Accept/reject per discovery.
4. **Recruitment Phase**: Chance to recruit a newly generated NPC from the event's archetype pool. Declined recruits become persistent Free Agents.
5. **Loot Phase**: Equipment, gold, XP distributed by placement scaling.

Phases with no content are skipped silently. All teams go through post-combat — losers still progress, just slower.

### Injury as Content Data

Named injury types exist within structural severity tiers (Minor/Major/Critical/Death). Each named injury specifies affected stats, penalty magnitude, recovery time, and special rules. New injuries are added as content data without code changes.

> **OPS Pattern — Configurable Event Templates**: Define events as templates with toggleable post-combat phases. Different event types (training, exhibition, championship, quest) are the same system with different profile flags. New event types require only new data configurations, not new code paths.

---

## 11. Tick-Based Economy

Discrete ticks as the universal time unit for all passive systems.

### Time Model

- **Single-player**: Ticks advance on player command. Play at your own pace.
- **Multiplayer**: Server-configurable real-time schedule.
- **Each tick resolves**: Job production, store stocking, NPC buyer simulation, upkeep deduction, natural healing, energy recharge, event scheduling.
- **Between ticks**: Unlimited player actions (training, buying/selling, roster management, PvE encounters).

### Character Value Formula

One formula drives multiple economic systems:

- **Hiring cost**: Total character value (Star Rating exponential base + Attribute totals + Trait count + Perk count + equipment).
- **Upkeep**: `hiring_cost x upkeep_rate` (~10% per tick).
- **Service pricing**: Major services (resurrection, respec) priced as multiples of character upkeep (10x/50x/100x for resurrection tiers).
- **Retirement value**: Metacurrency earned on retirement scales with the same formula.

One formula, four uses — consistent scaling without separate tuning per system.

### Power Ceiling — Plateau + Laterals

Characters hit a power plateau relatively quickly (attributes at Potential, core Perks acquired, good equipment). Further investment goes into **lateral options**: new Traits, alternative builds, niche Perks, specialized equipment. Power doesn't grow infinitely; versatility does.

This creates a natural XP endpoint, making gold the primary long-term resource, and ensures low-star characters remain viable (they plateau faster and cheaper, filling niche roles).

### Economic Pressure Systems

- **Equipment degradation**: No quality floor — equipment degrades to 0 (broken). Primary long-term gold sink. Repair/replacement cycle drives ongoing demand.
- **Upkeep**: Every character costs gold per tick. Higher-star characters cost exponentially more. The sole modifier is Lifestyle Level (Poor/Standard/Luxury).
- **Emergent income sources**: No single dominant income. Arena, crafting, and quest income are all viable primary strategies based on build investment.
- **Job system**: Characters assigned to Jobs produce goods/services. Mutually exclusive with combat roster — forces meaningful allocation decisions.

> **OPS Pattern — Single Valuation Formula**: Derive hiring cost, upkeep, service pricing, and retirement value from one character value formula. Prices scale consistently across all economic interactions without per-system tuning. Changes to the formula automatically propagate to all dependent systems.

---

## 12. World Structure

### Districts — Hybrid Tier + Theme Zones

The world is divided into districts combining a tier range with a thematic focus (e.g., The Pits = 1-2 star basic/survival, Trade Quarter = 2-3 star commerce/crafting, High Quarter = 3-5 star elite guilds). Each district contains 6-10 Groups.

### Groups as the Sole World Interface

All interaction between player and world flows through Groups:

- **Economic activity**: Buy/sell through Group vendor services.
- **Character progression**: Train through Group trainer services (staffed by characters with personal knowledge).
- **Recruitment**: Hire from Group recruitment pools.
- **World exploration**: Discover new Groups and districts through Bond introductions and Tavern rumors.

### Progressive Discovery

- **Starting state**: One district fully unlocked, all public Groups accessible.
- **Bond introductions**: Recruiting a character with a Bond Trait reveals that Group and unlocks its district.
- **Tavern rumors**: Per-tick information generation (Group reveals, quest leads, events, recruitment tips). Higher-tier Taverns produce more rumors.
- **No dead ends**: Multiple discovery mechanisms ensure players can always find new content.

### Template-Driven World Generation

- **Handcrafted core** (~10-15 Groups): Narratively important locations with curated NPCs, services, loot tables, and Bond Traits.
- **Procedural extras** (~15-20 Groups): Generated from templates (theme tags + tier -> auto-generate services, NPCs, loot tables, recruitment archetypes, Bond Trait). Fill districts with variety and replayability.

> **OPS Pattern — Unified World Interface Entity**: Route all player-world interaction through a single entity type (Group) with composable services. The world is a collection of Groups in themed zones. Progressive discovery through multiple mechanisms prevents dead ends while rewarding exploration.

---

## 13. Content-as-Data Philosophy

Every game concept that could be expressed as data IS expressed as data. New content requires no schema changes.

| Concept | Data Representation |
|---------|-------------------|
| Species/ancestry | Core Traits (no hardcoded species list) |
| Personality archetypes | Tags on Core Traits feeding AI Personality Scorers |
| Injuries | Content data within structural severity tiers |
| Resources | Content-defined types with attribute-derived formulas |
| Status effects | Per-status decay formulas and per-stack effects |
| Effect components | Extensible typed format (type tag + key-value params) |
| Equipment affixes | Perk component model applied to gear |
| Consumable effects | Perk component model applied to single-use items |
| AI behavior | Gate + Scorer configs embedded in Action definitions |
| Event types | Post-combat profile flags + Context Flags |
| Quest challenges | Tagged attribute checks with consequence tables |
| Crafting recipes | Perk-granted (acquiring the Perk unlocks the recipe) |

The practical consequence: a content author adding a new species, resource type, injury, status effect, or AI behavior writes data — never code. The structural framework remains constant while content scales arbitrarily.

> **OPS Pattern — Content-as-Data**: Express all game concepts as data within structural frameworks. Species are Traits. Injuries are content entries within severity tiers. Resources are formulas with auto-tags. New content interacts with existing systems through shared data models — no per-content code changes, no schema migrations.

---

## 14. Generation & Procedural Patterns

### Weighted Probability Loot Tables

- **Nestable subtables**: Top-level table can branch into subtables with weighted probability.
- **Per-archetype definition**: Each Group's recruitment pool uses its own weighted tables.
- **Template-driven**: Theme tags + tier determine default weights. Curated signature items can be overlaid.

### Character Generation

- **Archetype-based**: Per-Group archetype tables define weighted Trait distributions for recruits.
- **Openness parameter**: Stacking percentage chance per star level to leave slots empty. Low openness = veteran with pre-filled Traits. High openness = open-potential character for player customization.
- **Per-roll overrides**: Different probability distributions for 1st vs 2nd vs 3rd slot roll within a category. Allows thematic weighting (1st Core roll favors ancestry, 2nd favors personality).

### Free Agent Ecosystem

Declined recruits become persistent Named NPCs that:
- Appear at other Groups' recruitment pools later.
- May be hired by NPC Groups as workers.
- Retain name, stats, and Traits permanently.
- Create a living NPC ecosystem where characters move between Groups.

### World Generation

- **Hybrid approach**: Handcrafted core Groups provide narrative backbone. Procedural Groups generated from templates fill districts with variety.
- **Template composition**: Theme tags + tier -> auto-generate service menu, NPC roster, loot tables, recruitment archetypes, Bond Trait definition.

> **OPS Pattern — Template + Procedural Generation**: Use handcrafted templates as narrative anchors combined with procedural generation for variety and replayability. Loot tables, character generation, and world generation all use the same nestable weighted probability system with archetype-driven customization.

---

## 15. Progression Patterns

### XP Per Character

- **Not pooled**: Each character earns and spends their own XP.
- **Spent on**: Attribute increases (cost per +1 = current value, creating natural deceleration) and Perk purchases from Trainers.
- **Training Speed**: Intellect-derived percentage bonus to ALL XP earned. Smart characters learn faster from every activity.

### Perk Discovery

- **Per-Trait end-of-combat roll**: Only "active" Traits (had at least one Action used or Trigger fire) qualify.
- **Formula**: (Base Rate + Luck Bonus + Perk Bonuses) x Trait Level Multiplier.
- **Accept/reject**: Player chooses whether to take the discovered Perk. Rejection keeps it in the pool for future rolls.
- **Rarity weighting**: Lower-minimum-star Perks are more likely to be discovered.

### Trainer System

Training is the primary acquisition channel, staffed by characters (not abstract services):

- **Personal knowledge**: A trainer can only teach Traits/Perks they personally own.
- **Level caps student**: A trainer teaches up to their own level (Mentor Perks can raise this cap).
- **Instant execution**: Pay XP + gold, receive the level immediately. No wait time.
- **Self-training**: Players can assign their own characters as trainers (locked out of combat).

### Promotion

- **Metacurrency cost**: Exponential with Star Rating (cheap early, extremely expensive late).
- **Additive bonuses**: +10% to all Attribute Potentials, +1 Trait slot per category. All existing progress retained.

### Retirement & Hall of Fame

- **Permanent removal**: Retired characters leave the game entirely (not the Free Agent pool).
- **Metacurrency reward**: Scales with total character value formula.
- **Career record**: Persistent Hall of Fame preserves fight history, tournament results, and character legacy.

### Metacurrency Sources

- **Achievements** (~75%): Mix of unique milestones (one-time) and repeatable tiered achievements (win 10/50/100/500 fights).
- **Retirement** (~25%): Meaningful planned cycle (invest in character -> plateau -> retire for metacurrency).

> **OPS Pattern — Progression Through Turnover**: Design progression so characters reach a power plateau, then become more valuable as retirement metacurrency than as active roster members. This creates natural roster turnover — new characters replace retired ones, keeping the game fresh. Expensive respec encourages replacing characters rather than endlessly rebuilding a single one.

---

## 16. Quest System — Garrison Missions

Multi-tick expeditions combining abstracted Travel Challenges with fully simulated combat.

### Structure

1. **Travel Phase**: 0+ Travel Challenges, one per tick. Each tests a party Attribute vs. a difficulty class using weighted top-3 party member scores.
2. **Encounter Phase**: 0-1 fully simulated combat encounter using the standard combat engine.
3. **Resolution Phase**: Quest rewards distributed. Instant return (no forced return travel).

### Key Mechanics

- **Weighted Top-3**: `party_score = 0.6 x best + 0.3 x second + 0.1 x third`. Encourages diverse party composition; parties under 3 are significantly penalized.
- **Graduated failure**: Minor (1-20 margin) hits one character lightly. Major (21-50) hits one character harshly. Catastrophic (51+) hits the whole party.
- **Fixed enemy composition**: Defined by template regardless of party size. Sending more characters is an advantage but costs more upkeep.
- **Unique rewards as primary draw**: Rare materials, Trait discoveries, unique equipment, special NPCs, Group introductions — things unavailable elsewhere.
- **Partial info**: Players see star rating and general challenge types before accepting but exact DCs and enemy composition are hidden.

> **OPS Pattern — Hybrid Resolution Systems**: Combine abstracted checks (fast, attribute-driven) with fully simulated encounters (deep, engine-driven) within a single event type. Travel Challenges test party breadth; final combat tests party depth. Two optimization axes in one system.

---

## Summary of OPS Core Principles

1. **Universal Rating Scale**: One 0-5 star system across all entities.
2. **Composition over Inheritance**: Entities defined by what they contain, not what class they are.
3. **Tripartite Identity**: Character identity split into orthogonal identity/capability/affiliation axes.
4. **Unified Component Model**: All effects share one typed data format across Perks, equipment, and consumables.
5. **Tag-Driven Emergence**: Shared tag vocabulary creates automatic cross-system synergies.
6. **Emergent Resource Ownership**: Resource pools activated by content references, not explicit assignment.
7. **Simulation-First Combat**: Deterministic simulation producing structured data, presented afterward.
8. **Data-Driven AI**: AI evaluation logic embedded in content alongside mechanical effects.
9. **Configurable Event Templates**: Event types differ by data configuration, not code paths.
10. **Single Valuation Formula**: One formula drives hiring, upkeep, services, and retirement.
11. **Unified World Interface**: All player-world interaction through one composable entity type.
12. **Content-as-Data**: Every concept expressible as data IS data. New content = new data, never new code.
13. **Template + Procedural Generation**: Handcrafted anchors + procedural variety from shared generation systems.
14. **Progression Through Turnover**: Power plateaus + retirement metacurrency create natural roster cycling.
15. **Hybrid Resolution**: Combine abstracted checks with full simulation for multi-axis optimization.
