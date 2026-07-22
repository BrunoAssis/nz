# Neutral Zone Game Design Document

## Game Overview

Neutral Zone is a real-time strategy game set in space. Players control territories on a hexagonal grid representing the universe, managing resources, exploring unknown space, improving their planets and engaging in tactical combat with starships.

## Core Mechanics

### 0. Time Model

The game runs in **continuous real time** with a periodic **economic tick**:

- Movement, exploration, colonization, construction and combat all progress continuously in real time.
- Every **5 seconds** (tunable), an economic tick fires: planets produce Credits and territory influence is recalculated. Wherever this document says "per turn", read "per economic tick".
- The simulation is deterministic: same seed + same commands = same outcome.

### 1. Hexagonal Grid System

- **Grid Type**: Hexagonal with axial coordinates (q, r), pointy-top orientation
- **Grid Size**: we must test with the grid size, but the game can be played up to 12 players; the grid size will be proportional to the number of players. **v0**: 2 players (1 human vs 1 simple AI, no networking), hexagonal map of radius 5 (91 hexes), ~12 neutral planets placed randomly (minimum distance of 2 hexes between planets) plus 2 home planets.
- **Hex Types**:
  - **Neutral**: Unclaimed hexes
  - **Player Territory**: Contiguous areas controlled by players (colored borders)
  - **Fog of War**: Unexplored hexes
- **Occupancy**: A hex holds at most **1 unit**. Hexes occupied by allied units are obstacles for pathfinding; ordering a move into an enemy-occupied hex is an attack order.

### 2. Units (Starships)

All units have these core stats:

- **Speed**: Inverse to the time required to move between adjacent hexes. Time per hex = `6s / Speed` (tunable): Scout 2s, Fighter 3s, Battlecruiser 6s.
- **Attack**: Damage dealing capability
- **Defense**: Damage reduction capability
- **Health**: Hit points before destruction; might be recovered, but always slower than recovering shields. (Recovery is not implemented in v0; rates TBD for v1.)
- **Shield**: Absorbs damage before Health is affected; might be recovered. (Recovery is not implemented in v0; rates TBD for v1.)

Movement is **click-to-move**: the player selects a unit and clicks a destination; the unit follows an A* path through explored hexes, hex by hex, at its own speed. An unexplored destination adjacent to the explored path is allowed — the unit starts exploring upon entering it.

#### Unit Types:
 
**Scout Ship**
- Speed: 3
- Attack: 2
- Defense: 1
- Health: 10
- Shield: 5
- Cost: 5 $ | Build time: 10s

- Role: Fast exploration and reconnaissance

**Fighter**
- Speed: 2
- Attack: 3
- Defense: 2
- Health: 10
- Shield: 10
- Cost: 10 $ | Build time: 15s

- Role: Combat and maneuvarability

**Battlecruiser**

- Speed: 1
- Attack: 5
- Defense: 3
- Health: 20
- Shield: 10
- Cost: 20 $ | Build time: 25s

- Role: Heavy combat and territorial control

Costs and build times are initial values for balancing during playtesting.

##### Ideas for future units (won't be implemented now)

**Stealth Ship**
- Speed: 2
- Attack: 2
- Defense: 2
- Health: 10
- Shield: 5

- Role: Stealth and covert operations
- Special power: is invisible and can only be seen in special conditions

### 3. Combat System

#### Engagement:

- Combat starts **only when a unit attempts to enter a hex occupied by an enemy**. Adjacent enemies do not automatically fight (but adjacency matters for support).
- The attacker stops in the adjacent hex and both units become **locked in combat**: neither moves until the combat ends. The attacker may be explicitly ordered to retreat, which cancels the combat.
- If the defender is destroyed, the attacker completes its move into the contested hex. If the attacker is destroyed (or retreats), the defender stays.

#### Combat Resolution:

- Combat proceeds in **rounds of 1 second** (tunable). Each round, both units deal damage **simultaneously**.
- **Damage Formula**: `Damage = Attacker.Attack - Defender.Defense` (minimum 1), computed with effective (support- and territory-modified) stats

- **Damage Application**:
  1. Damage depletes Shield first
  2. When Shield = 0, damage depletes Health
  3. When Health = 0, unit is destroyed

#### Support Mechanics:
- Allied units in adjacent hexes (not in combat) provide support
- **Support Bonus**: 1.1x multiplier to supported unit's Attack and Defense
- Multiple support units stack (1.1 × 1.1 = 1.21x for 2 supporters, etc.)
- Multipliers are combined first and rounding happens once, on the final damage value.
- Support applies to both sides of a combat.

### 4. Locations (Planets)

#### Planet Properties:
- **One per hex maximum**
- **Base Production**: 1 Credit ($) per economic tick
- **Improvements**: Up to 2 types per planet
- **Ship Construction**: Ships are built through a per-planet **build queue**:
  - Enqueueing an item charges its full cost **instantly**.
  - Any queued or in-progress item can be **canceled at any time**; the refund is inversely proportional to construction progress (`refund = cost × (1 − progress)`; items still waiting in queue refund 100%).
  - Construction of the next item can only **start** while the planet's hex is unoccupied. If a ship is parked on the planet, the queue waits and the planet shows an **indicator icon** ("queued but blocked"). As soon as the ship leaves, construction starts automatically.
  - While a ship is under construction, it **occupies the planet's hex** (no other ship may enter until it's done).
  - The finished ship spawns on the planet's hex.

#### Planet Improvements:

**Shipyard**
- Effect: +20% ship construction speed
- Cost: 30 $
- Limit: One per planet

**Market**
- Effect: +20% $ production (1.2 $/turn)
- Cost: 20 $
- Limit: One per planet

More improvement types will be added in the future — including defensive ones that boost a planet's statblock.

#### Planet Defense (Statblocks):

- Every planet has its own statblock (Attack, Defense, Health, Shield — no Speed). More "military" planets soak more damage and fight back harder.
- Archetypes (initial values, tunable):

| Archetype | Attack | Defense | Health | Shield | Notes |
|---|---|---|---|---|---|
| Pacific | 0 | 0 | 10 | 0 | Does not resist; when neutral, can be colonized directly |
| Standard | 2 | 2 | 15 | 5 | Fights back |
| Fortress | 4 | 3 | 25 | 10 | Rare; heavy defense |
| Home | 3 | 2 | 20 | 10 | Player starting planet |

- Neutral planet distribution at map generation: ~50% Pacific, ~35% Standard, ~15% Fortress (tunable).
- A planet keeps its archetype statblock regardless of owner. In the future, new improvements will boost a planet's defensive stats.
- Planets never initiate combat; they only fight when attacked.
- Player-owned planets always defend when attacked, whatever their archetype (a Pacific one just falls quickly). Neutral planets: only Standard/Fortress resist.

#### Attacking a Planet:

- Ordering a unit into a hex containing an enemy (or resisting neutral) planet engages combat **against the planet**: the attacker locks in the adjacent hex and rounds proceed exactly like unit combat (same cadence, damage formula and application).
- If an enemy unit stands on the planet's hex, the attacker fights that unit first; a planet with Attack > 0 counts as one supporter (1.1x) for its defending unit. Once the unit is destroyed, renewing the attack engages the planet itself.
- **Support**: when a planet is attacked, **all** allied units adjacent to it — including a unit on its own hex — act as supporters (1.1x each to the planet's Attack and Defense), as long as they are not locked in a combat of their own.

#### Conquest (Planet at Health 0):

- The moment a planet's Health reaches 0 it can no longer defend itself, and:
  - One improvement is randomly destroyed.
  - Any ship under construction is destroyed (no refund); remaining queue items are refunded 100% to the owner.
- **Neutral planet** at Health 0 (or Pacific neutral, which never resists): the attacking unit enters the hex and **colonization starts automatically**, following the normal colonization cost rules (if the player cannot afford it, colonization auto-starts as soon as they can).
- **Enemy-owned planet** at Health 0: the attacker enters the hex and **de-colonization** starts automatically. Duration = colonization time / 2 (15s, tunable), with a progress bar; free of charge. When it completes, the planet becomes neutral; if the occupant stays, colonization then starts automatically.
- If the occupying unit leaves or is destroyed during de-colonization, it is fully canceled: the planet remains owned by its original player — defenseless at Health 0, so any enemy unit entering later restarts de-colonization from zero.
- A planet at Health 0 still produces credits for its owner (defenseless, not disabled).
- A planet is restored to full Health/Shield when a colonization completes (new owner takes an intact planet). There is no other planet regeneration in v0.

### 5. Economy System

#### Resources:

- **Credits ($)**: Base currency gathered from planets

#### Resource Generation:

- **Base Rate**: 1$ per economic tick (5s, tunable) per owned planet
- **Market Bonus**: +20% $ production

#### Starting Resources:

- Each player starts with 10$
- Home planet begins producing Credits immediately

### 6. Exploration System

Exploration determines visibility on the hex grid. Hexes are either Explored or Unexplored. Unexplored hexes are covered by fog of war and hide their contents.

#### Fog of War

- Only explored hexes are visible to the player.
- Exploring a hex reveals that hex only; neighbors remain fogged.
- Starting visibility: The player begins with some hexes already explored — radius 2 from the home planet (seed state only).
- Units do not automatically reveal fog by proximity.

#### How Exploring Works

- Eligibility: A hex can be explored only if at least one friendly unit "moves into" that hex.
- Action: When the player moves a unit to an eligible hex, it moves into it and begins exploring.
- Duration: Exploring takes N seconds (let's start with 10). A progress bar is shown on the hex while exploring (style: a progress bar that takes all the hex perimeter and fills up as the unit explores).
- Completion: When the bar fills, the hex becomes Explored and fog is removed for that hex only.
- Cancellation: If the exploring unit leaves the hex before completion, exploration is fully canceled and must be restarted.

#### Enemy Visibility

- In v0, enemy units are visible whenever they stand on a hex the player has explored. There are no hidden units yet.

#### Hidden Enemies (Design Note — v1+, depends on stealth)

- If a unit starts exploring a hex that contains a hidden enemy unit, combat should begin immediately upon entry. The "blind" unit has some disadvantage as it was surprised by the attack. TBD the disadvantage itself.

#### Colonization

- Colonization requires the target hex to be explored, and it is revealed that the hex contains a planet.
- Requirements: the planet is neutral and a friendly unit stands on its hex.
- A resisting neutral planet (Standard/Fortress archetype) must first be reduced to Health 0 in combat; Pacific neutral planets can be colonized directly (see Planet Defense).
- Duration: 30 seconds (tunable), with a progress bar. If the colonizing unit leaves or is destroyed, colonization is canceled.

##### Cost and Refunds

- Cost: Colonization cost scales with the number of planets the player has already colonized: $1 for the first, doubling for each next one ($1, $2, $4, $8...). The home planet does not count. The cost is charged when the Colonize action is started.
- Refunds on cancel: If Colonization is canceled, refund is proportional to remaining colonization time.

### 8. Territory Control

#### Ownership Rules:

- **Planet influence**: Whenever a planet is colonized, its hex automatically becomes part of a player's territory. A planet "emanates" influence outwards from its center, in all directions. This influence travels through the player's territory (contiguous areas owned by the player) until it hits a non-owned hex. After hitting that hex, that hex is influenced by the planet's influence. That hex might "turn" into player territory if the player's influence is larger than all other player's (including the "neutral" influence that represents unclaimed hexes). After turning a hex, other influences drop to zero. The influence strength is proportional to the planet's "influence" value and weakens as it moves away from the planet's center. If at any time a hex is not contiguous with the player's territory, its influence drops to zero.
- **Influence values (initial, tunable)**: each planet has `influence = 5`; the pressure applied on a frontier hex is `influence − distance` (minimum 0), accumulated per economic tick. The "neutral" influence of an unclaimed hex is a fixed 2. Influence and territory conversion are recalculated on each economic tick.
- **Contiguous Areas**: Adjacent hexes of same player form territories. The player's units have advantage on their territory: **+10% Defense** (initial value, tunable).

- **Neutral Zones**: Unclaimed hexes between player territories

#### Home Territory:

- Each player starts with one "home" hex containing a planet
- Home hex is always owned
- Starting unit: One Scout ship on home planet

### 9. Victory Conditions

#### Primary Objectives:

- **Territorial Control**: Control majority (50% + 1) of hexes
- **Military Victory**: Eliminate all enemy units and planets
- **Economic Dominance** (v1+): Achieve resource generation threshold (TBD)

In v0, only Territorial Control and Military Victory are implemented, checked on each economic tick.

#### Secondary Objectives (v1+):

- Explore all hexes on the map
- Build maximum improvements on all owned planets
- Achieve technological/production superiority

### 10. Neutral units (v1+)

- Neutral units are units that are not controlled by any player. They are always hostile towards all players.
- Some planets have neutral units on them which will defend that planet.
- Random events might trigger neutral units to appear at the edge of the map (always in a neutral zone hex). That neutral unit might be of any unit type. It will always have one objective: 1 - to take control of the nearest planet; 2 - to traverse through the map and exit on the other side (triggering a combat if any unit comes too close to it); 3 - to fortify the defense of a neutral planet.

## Game Flow

### Setup Phase:

1. Generate hex grid with random planet placement (seeded RNG)
2. Assign home hexes to players (always opposite sides of map, but not in the same hex every time to avoid predictability - the starting hex might be one of, let's say, 10 possible hexes).
3. Place starting units (Scout ship on home planet)
4. Initialize fog of war (only home territories visible)

All numeric values in this document (costs, durations, tick length, influence values) are initial defaults for playtesting and live in a single tunable config file.


## UI/UX Design

### Main Game View:

- Although the game mechanics are 2D, its visuals will be 3D, with simple and clean 3D models, which will be geometric low poly shapes, with 2-3 colors each, maximum (the predominant color will be the color of the player, with details in a "fixed" color - example: a ship hull will be blue or red depending on the player color, but it will have a "front window glass" which is always yellow and textured as glass).
- Planets will always be spheres (sometimes with rings, sometimes asteroidic, sometimes a "space station" format...) and might have particle effects on them.
- Hexagonal battlefield with clear unit/planet indicators
- Resource bar showing credits, number of colonized planets
- Selected unit/planet information panel
- The main scenario will be the deep space (blackish/purplish/galaxish/black-holish) background. The hexes will be mostly transparent, with a translucid fill of the color of that player when the hex is theirs. Fog of war for explorable hexes will have a greyish/cloudy tint. Unexplorable hexes (hexes which are not neighbouring any explored hex) will have a pitch-black cloudy tint.

### Selection System:

- The game must be fully playable using mouse only, mouse/keyboard, or touch controls.
- Click/tap to select hexes, units, or planets
- Visual feedback for valid moves/actions
- Context-sensitive action buttons

### Planet Interface:

- Build ships: enqueue items, see queue with progress, cancel items (with refund)
- "Queued but blocked" indicator icon on the planet when construction is waiting for a parked ship to leave
- Construct improvements (if not already built)
- View production status and bonuses

### Unit Interface:

- Unit status and health
- Colonize command (when on top of a planet)
