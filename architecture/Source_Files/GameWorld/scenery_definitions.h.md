# Source_Files/GameWorld/scenery_definitions.h

## File Purpose
Defines static scenery object types (environmental decorations and obstacles) for the game world. Contains a 61-entry array of pre-configured scenery definitions organized by environment theme (lava, water, sewage, alien, Jjaro), each specifying visual representation, physical properties, and destruction behavior.

## Core Responsibilities
- Define the `scenery_definition` struct describing individual scenery object properties
- Provide scenery behavior flags (solid, animated, destroyable)
- Maintain a static array of 61 pre-defined scenery instances for world placement
- Map scenery objects to graphical shape descriptors and collections
- Configure destruction effects and post-destruction replacement shapes

## Key Types / Data Structures
| Name | Kind | Purpose |
| ---- | ---- | ------- |
| scenery_definition | struct | Describes a scenery object type: visual shape, physical dimensions (radius/height), collision properties, and destruction effect |

## Global / File-Static State
| Name | Type | Scope | Purpose |
| ---- | ---- | ----- | ------- |
| scenery_definitions | array of `scenery_definition` (61 elements) | global | Pre-defined scenery objects indexed by scenario/map code for world instantiation |
| NUMBER_OF_SCENERY_DEFINITIONS | macro constant | global | Array size (61 objects) |

## Key Functions / Methods
None ΓÇö this is a pure data definitions header.

## Control Flow Notes
Not applicable; this file contains only static data declarations. The `scenery_definitions` array is indexed by game/map loading code to populate world scenery. At runtime, the engine consults these definitions to determine render shape, collision radius, height, and destruction behavior.

## External Dependencies
- `effects.h`: Effect type enums (`_effect_lava_lamp_breaking`, `_effect_water_lamp_breaking`, etc.)
- `shape_descriptors.h`: `shape_descriptor` typedef; `BUILD_DESCRIPTOR(collection, shape)` macro
- `world.h`: `world_distance` typedef; distance constants (`WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH`)

## Notes
- Scenery organized into five themed groups: lava (13 objects), water (11), sewage (10), alien/Pfhor (11), Jjaro (16)
- Many scenery objects are destructible (`_scenery_can_be_destroyed` flag), triggering specific effects on destruction
- Solid scenery (`_scenery_is_solid` flag) blocks player/entity movement
- Destroyed scenery may reveal replacement shapes (e.g., hanging lights break to expose bulbs)
- Conditional compilation: `#ifndef DONT_REPEAT_DEFINITIONS` allows optional exclusion of the array definition
- Comment at file top notes Jjaro scenery lamps are not yet made destroyable (could be toggled via effects)
