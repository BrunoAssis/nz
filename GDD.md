# Neutral Zone Game Design Document

## Game Overview

Neutral Zone is a real-time strategy game set in space. Players control territories on a hexagonal grid representing the universe, managing resources, exploring unknown space, improving their planets and engaging in tactical combat with starships.

## Core Mechanics

### 1. Hexagonal Grid System

- **Grid Type**: Hexagonal with axial coordinates (q, r)
- **Grid Size**: we must test with the grid size, but the game can be played up to 12 players; the grid size will be proportional to the number of players.
- **Hex Types**:
  - **Neutral**: Unclaimed hexes
  - **Player Territory**: Contiguous areas controlled by players (colored borders)
  - **Fog of War**: Unexplored hexes

### 2. Units (Starships)

All units have these core stats:

- **Speed**: Inverse to the time required to move between adjacent hexes
- **Attack**: Damage dealing capability
- **Defense**: Damage reduction capability
- **Health**: Hit points before destruction; might be recovered, but always slower than recovering shields.
- **Shield**: Absorbs damage before Health is affected; might be recovered.

#### Unit Types:
 
**Scout Ship**
- Speed: 3
- Attack: 2
- Defense: 1
- Health: 10
- Shield: 5

- Role: Fast exploration and reconnaissance

**Fighter**
- Speed: 2
- Attack: 3
- Defense: 2
- Health: 10
- Shield: 10

- Role: Combat and maneuvarability

**Battlecruiser**

- Speed: 1
- Attack: 5
- Defense: 3
- Health: 20
- Shield: 10

- Role: Heavy combat and territorial control

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

#### Combat Resolution:

- **Damage Formula**: `Damage = Attacker.Attack - Defender.Defense` (minimum 1)

- **Damage Application**:
  1. Damage depletes Shield first
  2. When Shield = 0, damage depletes Health
  3. When Health = 0, unit is destroyed

#### Support Mechanics:
- Allied units in adjacent hexes (not in combat) provide support
- **Support Bonus**: 1.1x multiplier to supported unit's Attack or Defense
- Multiple support units stack (1.1 × 1.1 = 1.21x for 2 supporters, etc.)

### 4. Locations (Planets)

#### Planet Properties:
- **One per hex maximum**
- **Base Production**: 1 Credit ($) per turn
- **Improvements**: Up to 2 types per planet
- **Ship Construction**: Can build ships if its hex is unoccupied by another ship

#### Planet Improvements:

**Shipyard**
- Effect: +20% ship construction speed
- Cost: 30 $
- Limit: One per planet

**Market**
- Effect: +20% $ production (1.2 $/turn)
- Cost: 20 $
- Limit: One per planet

#### Conquest Rules:

- If a player successfully attacks (invades/conquers) an enemy's planet, one improvement is randomly destroyed and the planet will now be "neutral". That player is now in an advantage position to "colonize" that planet and transfer its ownership to them.

### 5. Economy System

#### Resources:

- **Credits ($)**: Base currency gathered from planets

#### Resource Generation:

- **Base Rate**: 1$/turn per owned planet
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

#### Hidden Enemies (Design Note)

- If a unit starts exploring a hex that contains a hidden enemy unit, combat should begin immediately upon entry. The "blind" unit has some disadvantage as it was surprised by the attack. TBD the disadvantage itself.

#### Colonization

- Colonization requires the target hex to be explored, and it is revealed that the hex contains a planet.

##### Cost and Refunds

- Cost: Colonization cost scales with the number of planets the player has already colonized: $1 for the first, doubling for each next one. The cost is charged when the Colonize action is started.
- Refunds on cancel: If Colonization is canceled, refund is proportional to remaining colonization time.

### 8. Territory Control

#### Ownership Rules:

- **Planet influence**: Whenever a planet is colonized, its hex automatically becomes part of a player's territory. A planet "emanates" influence outwards from its center, in all directions. This influence travels through the player's territory (contiguous areas owned by the player) until it hits a non-owned hex. After hitting that hex, that hex is influenced by the planet's influence. That hex might "turn" into player territory if the player's influence is larger than all other player's (including the "neutral" influence that represents unclaimed hexes). After turning a hex, other influences drop to zero. The influence strength is proportional to the planet's "influence" value and weakens as it moves away from the planet's center. If at any time a hex is not contiguous with the player's territory, its influence drops to zero.
- **Contiguous Areas**: Adjacent hexes of same player form territories. The player's units have advantage on their territory. (TBD mechanically).

- **Neutral Zones**: Unclaimed hexes between player territories

#### Home Territory:

- Each player starts with one "home" hex containing a planet
- Home hex is always owned
- Starting unit: One Scout ship on home planet

### 9. Victory Conditions

#### Primary Objectives:

- **Territorial Control**: Control majority (50% + 1) of hexes
- **Economic Dominance**: Achieve resource generation threshold (TBD)
- **Military Victory**: Eliminate all enemy units and planets

#### Secondary Objectives:

- Explore all hexes on the map
- Build maximum improvements on all owned planets
- Achieve technological/production superiority

### 10. Neutral units

- Neutral units are units that are not controlled by any player. They are always hostile towards all players.
- Some planets have neutral units on them which will defend that planet.
- Random events might trigger neutral units to appear at the edge of the map (always in a neutral zone hex). That neutral unit might be of any unit type. It will always have one objective: 1 - to take control of the nearest planet; 2 - to traverse through the map and exit on the other side (triggering a combat if any unit comes too close to it); 3 - to fortify the defense of a neutral planet.

## Game Flow

### Setup Phase:

1. Generate hex grid with random planet placement
2. Assign home hexes to players (always opposite sides of map, but not in the same hex every time to avoid predictability - the starting hex might be one of, let's say, 10 possible hexes).
3. Place starting units (Scout ship on home planet)
4. Initialize fog of war (only home territories visible)


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

- Build ships (if hex is empty)
- Construct improvements (if not already built)
- View production status and bonuses

### Unit Interface:

- Unit status and health
- Colonize command (when on top of a planet)
