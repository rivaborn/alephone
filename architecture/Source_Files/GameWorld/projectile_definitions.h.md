# Source_Files/GameWorld/projectile_definitions.h

## File Purpose

Defines the static data structures and configurations for all projectile types in the game engine. Contains the master `projectile_definition` struct and initializes a global array with concrete parameters (damage, speed, effects, sounds, flags) for every projectile type in the game.

## Core Responsibilities

- Define projectile behavior flags (guided, gravity-affected, rebounds, penetration, etc.) as bit-flags
- Declare the `projectile_definition` struct that aggregates all projectile properties
- Initialize `original_projectile_definitions[]` with complete configuration for ~40 projectile types
- Provide serialization/deserialization entry points for projectile definition data
- Support mod/customization by maintaining a mutable copy of projectile definitions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `projectile_definition` | struct | Core data structure holding all static properties of a projectile type: visuals, effects, damage, physics behavior, flags, and audio |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `projectile_definitions` | `projectile_definition[]` | static/global | Mutable array of projectile definitions, copied from originals at init for potential runtime modification |
| `original_projectile_definitions` | `const projectile_definition[]` | static/global | Read-only master list of all projectile configurations; ~40 entries covering player weapons, alien projectiles, special effects |

## Key Functions / Methods

### unpack_projectile_definition
- **Signature:** `uint8 *unpack_projectile_definition(uint8 *Stream, projectile_definition *Objects, size_t Count)`
- **Purpose:** Deserializes projectile definition data from a byte stream (for loading from maps/save files)
- **Inputs:** Byte stream pointer, target projectile definition array, count of definitions to unpack
- **Outputs/Return:** Advanced stream pointer (consumed bytes)
- **Side effects:** Populates the projectile definition objects in memory
- **Calls:** Not visible in this file (defined elsewhere)
- **Notes:** Used when loading map files or save games to restore projectile configurations

### pack_projectile_definition
- **Signature:** `uint8 *pack_projectile_definition(uint8 *Stream, projectile_definition *Objects, size_t Count)`
- **Purpose:** Serializes projectile definition data into a byte stream (for saving maps/games)
- **Inputs:** Byte stream pointer, projectile definition array, count of definitions to pack
- **Outputs/Return:** Advanced stream pointer (bytes written)
- **Side effects:** Writes binary data to stream
- **Calls:** Not visible in this file (defined elsewhere)
- **Notes:** Inverse of unpack; enables persistence across sessions

## Control Flow Notes

This file is a **configuration/initialization module**. It is not part of the per-frame gameplay loop. Instead:

1. **Initialization** ΓåÆ `original_projectile_definitions[]` is defined at compile-time with all 40+ projectile types pre-configured
2. **Load time** ΓåÆ Unpack/pack functions may restore or save modified definitions from/to disk
3. **Runtime** ΓåÆ Projectile creation code (in projectiles.c) reads from `projectile_definitions[]` to spawn projectiles with correct behavior
4. **Per-projectile** ΓåÆ Each active projectile references its definition by type index to determine damage, speed, visual effects, etc.

## External Dependencies

- **effects.h** ΓÇô Effect type constants (`_effect_rocket_explosion`, `_effect_grenade_explosion`, etc.)
- **map.h** ΓÇô `damage_definition` struct; world coordinate types; `WORLD_ONE` scale constant
- **media.h** ΓÇô Media detonation effect types
- **projectiles.h** ΓÇô Projectile type enum (`NUMBER_OF_PROJECTILE_TYPES`); `projectile_data` struct
- **SoundManagerEnums.h** ΓÇô Sound event constants (`_snd_rocket_flyby`, `_snd_fusion_flyby`, etc.); frequency constants (`_normal_frequency`, `_higher_frequency`)

---

**Notes on Projectile Flags:**
The file defines ~24 flag bits controlling projectile behavior: guidance, gravity interaction, media penetration, collision handling, detonation modes, and visual effects. Most flags are mutually exclusive or apply to specific projectile families (alien vs. player weapons, melee vs. ranged). Two specialized flags (`_penetrates_media_boundary`, `_passes_through_objects`) enable SMG bullets to enter/exit liquids without detonatingΓÇöa feature added in Feb 2000.
