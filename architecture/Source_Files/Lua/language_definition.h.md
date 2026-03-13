# Source_Files/Lua/language_definition.h

## File Purpose
A compile-time constant definition file that maps human-readable symbolic names to internal numeric codes for game engine entities. It serves as the "language definition" for script writers, enabling them to reference items, monsters, sounds, damage types, and game mechanics by mnemonic names rather than raw hex values.

## Core Responsibilities
- Define symbolic constants for weapons, items, and powerups with hexadecimal IDs
- Map monster/creature type mnemonics to engine-internal codes
- Define damage source type constants for hit/death attribution
- Enumerate monster behavioral states and action codes
- Map audio effect IDs to sound event names
- Define projectile/attack type mnemonics
- Categorize level geometry (polygon) types and properties
- Enumerate game mode constants (deathmatch, CTF, king-of-the-hill, etc.)
- Provide backwards-compatible naming aliases (underscore-prefixed and non-prefixed variants)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This file is a pure data definition file included directly into another compilation unit. It organizes constants into logical sections:
1. **Items** (0x00ΓÇô0x23): weapons, ammo, powerups, balls, keys
2. **Monsters** (0x00ΓÇô0x2E): creature type IDs  
3. **Damage Types** (0x00ΓÇô0x17): sources of harm for attribution
4. **Monster Classes** (0x0001ΓÇô0x8000): bitmask flags for monster grouping
5. **Player Actions** (0x00ΓÇô0x0D): movement, trigger, queue controls
6. **Monster Actions** (0x00ΓÇô0x0B): stationary, attacking, dying, teleporting states
7. **Monster Modes** (0x00ΓÇô0x04): lock-on behavior states
8. **Faders** (0x00ΓÇô0x20): screen effects and cinematic transitions
9. **Network Compass** (0x00ΓÇô0x10): directional beacons
10. **Sounds** (0ΓÇô214): extensive SFX library (weapons, enemies, ambience, UI)
11. **Projectiles** (0x00ΓÇô0x26): attack type IDs
12. **Polygons** (0x00ΓÇô0x17): level geometry properties (teleporters, triggers, damage zones)
13. **Powerups** (0x00ΓÇô0x03): enhancement types
14. **Game Types** (0x00ΓÇô0x08): multiplayer modes
15. **Netscript bindings**: references to undefined symbols (appears non-functional)

## External Dependencies
- **Undefined symbol references** (netscript section): `_panel_is_oxygen_refuel`, `_panel_is_shield_refuel`, `_panel_is_double_shield_refuel`, `_panel_is_triple_shield_refuel`, `_weapon_fist`, `_weapon_pistol`, `_weapon_plasma_pistol`, `_weapon_shotgun`, `_weapon_assault_rifle`, `_weapon_smg`, `_weapon_flamethrower`, `_weapon_missile_launcher`, `_weapon_alien_shotgun`, `_weapon_ball`, `_game_of_most_points`, `_game_of_most_time`, `_game_of_least_points`, `_game_of_least_time` ΓÇö defined elsewhere.

## Notes
- Dual naming convention: entries are duplicated with/without underscore prefix for backwards compatibility (e.g., `_item_knife` / `knife`).
- Explicit backwards-compatibility note in header: the first `shotgun` mnemonic entry (0x16) was removed due to naming conflict (0x17 also claimed "shotgun"), replaced with a comment to avoid script breakage.
- Netscript section ("useless in this file?") suggests these constant mappings may be unused or represent a legacy scripting interface.
