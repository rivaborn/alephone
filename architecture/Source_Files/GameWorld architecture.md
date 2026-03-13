# Subsystem Overview: GameWorld

## Purpose

The GameWorld subsystem is the core simulation engine for the Marathon game world, managing all entities (player, monsters, projectiles, items, effects), world geometry (polygon BSP, lines, sides), physics simulation, AI pathfinding, interactive elements (platforms, devices, lights), and the main 30 FPS game loop that orchestrates frame-by-frame updates to all game state.

## Key Files

| File | Role |
|------|------|
| `map.h` / `map.cpp` | Core map geometry (polygons, lines, sides, endpoints), entity lists, spatial queries, object lifecycle |
| `marathon2.cpp` | Main game loop (`update_world`), world state manager, level transitions, trigger logic, completion detection |
| `player.h` / `player.cpp` | Player entity, physics state, inventory, damage/death, powerup management, serialization |
| `monsters.h` / `monsters.cpp` | Monster AI, behavior state machines, pathfinding, targeting, combat, activation, damage handling |
| `projectiles.h` / `projectiles.cpp` | Projectile simulation, ballistics, collision detection, detonation, damage application |
| `items.h` / `items.cpp` | Item lifecycle, placement, pickup mechanics, inventory updates, animation |
| `effects.h` / `effects.cpp` | Visual/audio effects (explosions, splashes, teleportation), animation, serialization |
| `platforms.h` / `platforms.cpp` | Dynamic platforms/doors, state transitions, collision handling, geometry updates, adjacent cascades |
| `devices.cpp` | Interactive control panels (switches, terminals, refuel stations, save points), action key targeting |
| `lightsource.h` / `lightsource.cpp` | Dynamic lighting, six-phase animation state machines, intensity curves, activation triggers |
| `media.h` / `media.cpp` | In-world liquids (water, lava, goo, sewage, Jjaro), height calculation, damage, effects |
| `physics.cpp` / `physics_models.h` | Character movement, gravity, collision resolution, aim input processing |
| `world.h` / `world.cpp` | Trigonometric tables, point transformations, distance calculations, deterministic RNG |
| `flood_map.h` / `flood_map.cpp` | Polygon-based graph search for pathfinding, path retrieval via backtracking |
| `pathfinding.cpp` | Waypoint generation, path traversal, random biased paths for unreachable destinations |
| `interpolated_world.h` / `interpolated_world.cpp` | Double-buffered world snapshots, per-frame interpolation (>30 FPS rendering) |
| `map_constructors.cpp` | Map geometry precalculation, side type derivation, serialization/deserialization |
| `ephemera.h` / `ephemera.cpp` | Temporary visual objects (particles, sprites), object pool allocation, polygon-based spatial indexing |
| `scenery.h` / `scenery.cpp` | Static/decorative objects, animation, destruction effects, MML configuration |
| `weapons.h` / `weapons.cpp` | Player weapon state machines, firing, reloading, ammo management, shell casings |

## Core Responsibilities

- **Entity management**: Create, update, and destroy all game world entities (player, monsters, projectiles, items, effects) with per-entity state structures
- **Physics simulation**: Character movement with gravity/collision, polygon height changes, media interactions, 30 FPS tick-based deterministic calculations
- **Monster AI**: Pathfinding via flood-fill, target selection and line-of-sight, state machines (locked/lost/unlocked), melee/ranged combat, difficulty scaling
- **Game loop orchestration**: Main `update_world()` frame handler; coordinate all subsystem ticks; manage action queues from player input, Lua, and prediction
- **Interactive world elements**: Platforms (doors, elevators), control panels (terminals, switches), dynamic lights, adjacent geometry cascading on state changes
- **Collision detection and spatial queries**: Polygon boundary crossing, line intersections, media containment, entity visibility, terrain obstruction
- **Pathfinding and movement**: Breadth-first flood-fill of polygon graph; waypoint generation for monster navigation; random biased destination selection
- **Environmental effects**: Liquids (height, current, damage), dynamic lighting (intensity animation curves, state transitions), visual/audio effect spawning

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- `update_world()` ΓÇö main game loop entry point (called by shell/main loop)
- Entity creation/removal functions (`new_monster()`, `new_projectile()`, `new_item()`, `new_effect()`, `new_light()`, `new_platform()`)
- Player/entity query and modification functions (`get_player_data()`, `get_monster_data()`, `get_object_data()`, `translate_map_object()`)
- Collision/geometry queries (`find_adjacent_polygon()`, `point_in_polygon()`, `find_line_crossed()`)
- Pathfinding entry point (`flood_map()` for cost-based path planning)

**Consumes from other subsystems:**
- **Rendering** (interface.h, render.h): Collection loading/unloading, shape animation data, camera updates
- **Audio** (SoundManager.h): Play world sounds, 3D audio positioning, ambient sound updates
- **Input** (ActionQueues.h): Player action flags (movement, aim, weapon input), Lua action injection
- **Networking** (network.h, network_games.h): World state synchronization, client-side prediction rollback, netgame type queries
- **Scripting** (lua_script.h): Event callbacks (entity damaged/killed, platform activated, item created), custom behavior injection
- **Configuration** (InfoTree.h): MML/XML parsing for entity, weapon, item, platform, light customization

## Runtime Role

**Initialization (`entering_map`):**
- Allocate or reuse entity storage arrays (respecting `dynamic_limits`)
- Load map geometry, triggers, platforms, lights, media from WAD/map data
- Place initial monsters and items via `placement.cpp`
- Initialize player state and spawn point

**Per-frame update (`update_world`, 30 FPS tick):**
- Process action queues (real player, Lua, prediction systems); merge and dequeue
- Update physics: player movement, gravity, collision (via `physics.cpp`)
- Update entities: monsters AI/pathfinding, projectile ballistics, item animation
- Detect and handle collisions, damage application, death/respawn
- Evaluate polygon-based trigger logic (lights, platforms, monsters)
- Execute Lua event callbacks (pre/post-update idle)
- Manage level completion conditions (extermination, exploration, retrieval, repair, rescue)

**Shutdown (`leaving_map`):**
- Serialize all entity state if saving game
- Clear entity arrays and dealloc resources
- Unload collections and sounds

**High-frequency rendering (>30 FPS):**
- `interpolated_world.cpp` maintains double-buffered snapshots at each 30 FPS tick boundary
- Per render frame, interpolate entity positions, camera, weapon display based on elapsed machine ticks

## Notable Implementation Details

- **Fixed-point arithmetic**: All positions, distances, angles use fixed-point (`_fixed`, `world_distance`) or int16 for deterministic, replay-compatible calculations across platforms
- **Entity pools with dynamic limits**: Storage arrays (ObjectList, MonsterList, ProjectileList, EffectList) respect runtime-adjustable limits per game mode/compatibility level; limits can be queried or overridden via MML
- **BSP spatial organization**: Polygons form the primary spatial structure; entities belong to polygons; flood-fill pathfinding uses polygon graph adjacency
- **Deferred operations**: Map object addition/removal use deferred queues to support networking prediction/rollback without immediate state mutation
- **Object pool allocators**: Ephemera use linked-list free-list design for O(1) reuse; lookup tables cache animation/shape data
- **Deterministic RNG**: Global and local LFSR seeds enable reproducible randomness for film playback and network synchronization
- **MML/XML configuration**: All entity types (monsters, weapons, items, lights, platforms, effects) support MML-driven property overrides at runtime
- **Lua integration**: Extensive event hooks (`L_Call_Init`, `L_Call_Idle`, `L_Call_PostIdle`, `L_Call_Player_Damaged`, `L_Call_Projectile_Created`, etc.) allow scripted extensions and custom behavior without engine recompilation
