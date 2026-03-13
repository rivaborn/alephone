# Source_Files/GameWorld/media.h

## File Purpose
Header defining the media (liquid) system for the game engine. Media represents in-world liquids (water, lava, goo, etc.) with configurable properties including height, current, damage, and audio. Supports flexible media heights via light intensity mapping and extensible per-map media definitions via MML (Marathon Markup Language).

## Core Responsibilities
- Define media type enums (`_media_water`, `_media_lava`, `_media_goo`, `_media_sewage`, `_media_jjaro`)
- Define media flags (sound obstruction) and associated macros
- Declare `media_data` structure encoding all persistent media properties (32 bytes)
- Manage global `MediaList` vector of all active media instances
- Declare accessors and query functions for media properties
- Provide MML parsing and runtime update routines

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `media_data` | struct | Stores all properties of a single media instance: type, flags, light index (for height), current direction/magnitude, bounds, origin, height, texture, transfer mode |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MediaList` | `vector<media_data>` | global | Dynamic collection of all media in current map; replaces fixed array |

## Key Functions / Methods

### new_media
- Signature: `size_t new_media(struct media_data *data)`
- Purpose: Create and register a new media instance in the map
- Inputs: Pointer to initialized `media_data` structure
- Outputs/Return: Size/index of newly created media
- Side effects: Appends to `MediaList`
- Calls: (not visible in this file)
- Notes: Uses dynamic allocation via vector

### update_medias
- Signature: `void update_medias(void)`
- Purpose: Advance simulation of all active media (currents, animations, effects)
- Inputs: None; reads global `MediaList`
- Outputs/Return: None
- Side effects: Updates each `media_data` in-place; may trigger audio/visual effects
- Calls: (implementation in media.c)
- Notes: Called once per frame; integrates with game time

### get_media_data
- Signature: `media_data *get_media_data(const size_t media_index)`
- Purpose: Accessor for media instance; returns null for invalid indices
- Inputs: Media index
- Outputs/Return: Pointer to `media_data` or `NULL`
- Side effects: None (read-only)
- Calls: None visible
- Notes: Safe accessor (null-check capable)

### Query Functions
- `get_media_detonation_effect()` ΓÇö Fetch visual effect for media explosions
- `get_media_sound()` ΓÇö Fetch sound index for event (entering/leaving/splashing)
- `get_media_submerged_fade_effect()` ΓÇö Fetch fade effect for submersion
- `get_media_damage()` ΓÇö Fetch damage taken by entities in media (scaled)
- `get_media_collection()` ΓÇö Retrieve texture collection for media
- `IsMediaDangerous()` ΓÇö Test if media type deals damage
- `count_number_of_medias_used()` ΓÇö Count distinct media types in use

### MML Parsing
- `parse_mml_liquids()` ΓÇö Parse XML-based liquid definitions from InfoTree
- `reset_mml_liquids()` ΓÇö Reset liquid definitions to defaults

### Serialization
- `pack_media_data()` / `unpack_media_data()` ΓÇö Binary serialization for save/network

## Control Flow Notes
Media integrates into the frame loop via `update_medias()`, called during `update_world()`. Media height is dynamically computed from a light intensity curve (`light_index`), allowing flexible per-polygon water levels without per-frame recalculation. Damage and sound effects are polled on demand by entity systems (monsters, projectiles, players).

## External Dependencies
- **Includes**: `<vector>` (C++ STL), `map.h` (provides `SLOT_IS_USED` macros and world types)
- **Types from elsewhere**: `damage_definition` (map.h), `shape_descriptor`, `world_distance`, `world_point2d`, `world_point3d`, `angle`, `_fixed` (world.h / type system)
- **Classes declared**: `InfoTree` (XML tree; defined elsewhere, used for MML parsing)
- **Macros used**: `TEST_FLAG16`, `SET_FLAG16`, `UNDER_MEDIA` (height test)
