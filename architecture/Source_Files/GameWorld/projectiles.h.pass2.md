# Source_Files/GameWorld/projectiles.h - Enhanced Analysis

## Architectural Role

This header anchors the **GameWorld projectile subsystem**, a critical component of the 30 FPS deterministic simulation loop. Projectiles are second-class entities (not full objects in the object pool) managed via a dynamic vector, distinct from monsters/items but equally essential to physics-based gameplay. The system bridges weapon/monster firing (upstream in GameWorld), collision detection (map.cpp), detonation effects (effects system), and network serialization (for multiplayer sync via pack/unpack functions). Its 40+ projectile type enum reflects Marathon's weapon-diverse design philosophy spanning weapons, enemy fire, and physics phenomena.

## Key Cross-References

### Incoming (who depends on this file)
- **weapons.cpp** *(GameWorld)* ΓåÆ calls `new_projectile()`, `preflight_projectile()` when player fires
- **monsters.cpp** *(GameWorld)* ΓåÆ calls `new_projectile()` for enemy fire; checks `ProjectileIsGuided()`
- **items.cpp** *(GameWorld)* ΓåÆ `drop_the_ball()` creates ball-type projectiles on item pickup
- **marathon2.cpp** *(GameWorld main loop)* ΓåÆ calls `move_projectiles()` once per tick; calls `remove_all_projectiles()` on level transitions
- **lua_map.cpp** *(Lua scripting)* ΓåÆ reads/queries projectile data for game logic callbacks
- **game_wad.cpp** *(Files)* ΓåÆ packs/unpacks projectile data for save/load and network map sync via `pack_projectile_data()`, `unpack_projectile_definition()`
- **Rendering/effects.cpp** *(presumed)* ΓåÆ reads projectile positions for visual effects (contrails, explosions)

### Outgoing (what this file depends on)
- **dynamic_limits.h** *(CSeries)* ΓåÆ `MAXIMUM_PROJECTILES_PER_MAP` fetches dynamic limit at runtime
- **world.h** *(GameWorld)* ΓåÆ angle type, trigonometric utilities, world_point3d, world_distance, _fixed (damage scaling)
- **map.cpp** *(GameWorld)* ΓåÆ collision detection, polygon queries (implied via `translate_projectile()` polygon_index parameters)
- **Effects system** *(GameWorld)* ΓåÆ detonation spawns visual/audio effects
- **Sounds** *(Sound subsystem)* ΓåÆ `load_projectile_sounds()`, `mark_projectile_collections()` trigger audio playback

## Design Patterns & Rationale

**Dynamic Limits Pattern**: Rather than hard-coded `#define MAXIMUM_PROJECTILES_PER_MAP 256`, the macro calls `get_dynamic_limit()` at runtime. This allows **MML/XML-driven customization** for scenario authorsΓÇöa key affordance for the mod-friendly Aleph One ecosystem. Contrasts with fixed limits in the original Marathon.

**Vector-over-Array**: The switch from a static array to `std::vector<projectile_data>` (circa 2000s refactoring) sacrifices minor determinism risk (reallocation order) for **unbounded projectile counts** and cleaner memory managementΓÇöessential for modded scenarios with heavy weapon fire.

**Bitfield State Flags**: Six macros pack flyby-sound, damage-infliction, and media-crossing status into a `uint16` flags field (13 bits unused per comment). This **space-conscious design** reflects 1990s memory constraints and ensures 32-byte `projectile_data` struct fits cache-efficiently.

**Guided vs. Unguided**: The `ProjectileIsGuided()` accessor and `target_index` field enable **AI-driven homing projectiles** (e.g., hunter, major defender), separating tracking logic from physics. Guided projectiles call `preflight_projectile()` to validate target legality before firing.

**Packing/Unpacking Abstraction**: Four serialization functions (`pack/unpack_projectile_data`, `pack/unpack_projectile_definition`) decouple wire format from memory layoutΓÇö**essential for save-game compatibility across engine versions** and network multiplayer sync (peers must agree on projectile state without exposing pointer values).

## Data Flow Through This File

**Creation Pipeline**:
1. Weapon/monster fire ΓåÆ `preflight_projectile()` (validates position, media, guided target)
2. If valid ΓåÆ `new_projectile()` allocates slot in `ProjectileList`, calls `mark_projectile_collections()` for audio
3. Projectile stored with owner/target/damage_scale metadata

**Per-Frame Simulation** (called from `marathon2.cpp` main loop):
- `move_projectiles()` iterates `ProjectileList`
- For each projectile: update position via physics (gravity, velocity), check contrail timer
- Call `translate_projectile()` to detect collisions (returns flags: hit_monster, hit_floor, hit_media, hit_landscape, hit_scenery)
- On collision: call `detonate_projectile()` (spawns effects, applies damage) or `remove_projectile()` (cleanup)

**Ownership & Cleanup**:
- Ownerless projectiles (owner=NONE) persist; `owner_type` still valid for damage attribution
- If owner dies: `orphan_projectiles()` cleans dangling references
- Level transition: `remove_all_projectiles()` clears all

**Serialization**:
- Save game / network sync: `pack_projectile_data()` writes binary blob
- Load / receive: `unpack_projectile_data()` restores state from blob

## Learning Notes

**Era-Specific Idioms**: The file's lineage (June 1994 original, updates through 2001 by Loren Petrich and Aleph One devs) reveals **incremental feature addition** (SMG bullet in Feb 2000, dynamic limits Feb 2010, guided accessor 2001) rather than architectural redesign. Comments in the header document each additionΓÇöa pattern that would be relegated to git commit messages in modern engines.

**Modding-Centric Design**: The dynamic limits, packing/unpacking, and `ProjectileIsGuided()` accessor all suggest **strong emphasis on scenario authorship customization**ΓÇöAleph One's primary extension point. Physics definitions unpack from WAD files, not hardcoded.

**Guided Projectiles as AI Feature**: Unlike modern engines (Unity/Unreal) that bind targeting to scriptable behaviors, Marathon bakes `target_index` into the core projectile struct, signaling that **homing was a first-class gameplay primitive**, not a scripted afterthought.

**Permutation for Item Drops**: The `permutation` field (item type if detonation creates a drop) ties projectile detonation to inventory mechanicsΓÇöa tight coupling reflecting older era constraints.

## Potential Issues

1. **Parameter Name Typo**: Cross-reference index shows `translate_projectile(... short projectile_indexx)` (double 'x'). Likely a typo in the header; check implementation in projectiles.cpp.

2. **Flag Space Exhaustion**: The `flags` uint16 has only 3 bits used (flyby, damage, media), leaving 13 unusedΓÇöbut near-full comment suggests future expansion was anticipated. If more flags are needed, a redesign to uint32 or a separate state struct may be required.

3. **Owner-Type Decoupling**: `owner_index` can be NONE while `owner_type` is valid (e.g., grenade bounces after monster dies). This dual representation is error-prone if not carefully maintained in `orphan_projectiles()` and damage attribution.

4. **Guided Target Validation**: `preflight_projectile()` validates target, but no lifetime check shownΓÇöif target dies after firing but before detonation, `translate_projectile()` may dereference a stale target_index. Likely handled in monsters.cpp's orphan logic, but not visible here.
