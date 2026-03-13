# Source_Files/GameWorld/item_definitions.h

## File Purpose
Defines the static item metadata table for the game engine. Contains definitions for all itemsΓÇöweapons, ammunition, powerups, special items (keys, balls), and recharge stationsΓÇöused during inventory management and item spawning.

## Core Responsibilities
- Define the `item_definition` struct to hold item metadata (kind, names, shape, counts)
- Declare static array `item_definitions[]` containing ~50 item entries indexed by item type
- Support per-difficulty item count limits via extended array
- Provide method to query maximum carry count for a given difficulty level
- Prevent duplicate definitions in script-generated code via `DONT_REPEAT_DEFINITIONS` guard

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `item_definition` | struct | Metadata for a single item type: kind, localized name IDs, 3D shape, max carry count, invalid environments, and per-difficulty overrides |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `item_definitions[]` | `struct item_definition[]` | static | Indexed array of ~50 item definitions (weapons, ammo, powerups, balls, keys) initialized at compile time |

## Key Functions / Methods

### item_definition::get_maximum_count_per_player
- **Signature:** `int16 get_maximum_count_per_player(bool is_m1, int difficulty_level) const;`
- **Purpose:** Return the maximum number of this item a player can carry, accounting for difficulty level and Marathon 1 compatibility.
- **Inputs:** `is_m1` (Marathon 1 mode), `difficulty_level` (0ΓÇô4, _wuss_level to _total_carnage_level)
- **Outputs/Return:** `int16` max count (or `NONE` if not set)
- **Side effects:** None (const method)
- **Calls:** None visible in header
- **Notes:** Uses `extended_maximum_count[]` array if available; falls back to `maximum_count_per_player`.

## Control Flow Notes
This file is purely dataΓÇöit is compiled into the binary as a static array and accessed during:
- **Initialization:** Loaded once at game start to initialize item system
- **Runtime:** Queried when adding items to inventory, spawning items on map, or checking item capacity

No active control flow; acts as a lookup table for the items system (items.h).

## External Dependencies
- **items.h:** Enum definitions (`_weapon`, `_ammunition`, `_powerup`, `_item`, `_ball`; item type IDs like `_i_knife`, `_i_magnum`, etc.)
- **map.h:** Type definitions (`shape_descriptor`, `BUILD_DESCRIPTOR`, `UNONE`, game difficulty constants)
- **Conditional compilation guard:** `DONT_REPEAT_DEFINITIONS` (used by script_instructions.cpp to avoid duplication)
