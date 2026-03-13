# Source_Files/GameWorld/physics.cpp - Enhanced Analysis

## Architectural Role

Physics.cpp is the **input-to-simulation bridge** in the 30 FPS game loop, bridging the Input subsystem (raw hardware input) with GameWorld state (player entity, world geometry, AI/rendering). It implements the core physics simulation deterministically using fixed-point arithmetic to enable networked multiplayer sync. The file is also a **model repository**ΓÇöit houses physics_models[], a global array of tuned movement constants that MML configuration can override, making physics easily moddable without recompilation.

## Key Cross-References

### Incoming (who depends on this)
- **Input subsystem** ΓåÆ provides action_flags via `process_aim_input()` (mouse/gamepad deltas encoded into low-precision yaw/pitch deltas; fractional residuals tracked in vir_aim_delta)
- **GameWorld main loop** (`marathon2.cpp`) ΓåÆ calls `update_player_physics_variables()` every tick (30 Hz deterministic)
- **Rendering** ΓåÆ reads updated `player->variables.position/direction/elevation` to compute camera transforms each frame
- **Network layer** ΓåÆ serializes/deserializes physics_constants via `pack_physics_constants()` / `unpack_physics_constants()` for cross-machine sync and replay validation

### Outgoing (what this depends on)
- **Map subsystem** (`map.cpp`) ΓåÆ collision detection via `keep_line_segment_out_of_walls()`, `legal_player_move()`, polygon height queries, media (liquid) transitions
- **Monsters subsystem** (`monsters.cpp`) ΓåÆ `bump_monster()` when colliding with entities; `get_monster_definition_external()` for grenade-climb special case
- **Devices/Events** ΓåÆ `changed_polygon()` callback (marathon2.cpp) fired when player crosses polygon boundaries (platform activation, trigger zones)
- **Globals** ΓåÆ reads/modifies `physics_models[]` (walking/running variants); manages static `vir_aim_delta` (virtual aim offset for smooth input interpolation across network frames)

## Design Patterns & Rationale

**Dual-Phase Physics for Netplay Determinism**
- `physics_update()` is pure: computes new velocity/position/angles from constants and inputΓÇöno world side effects
- `instantiate_physics_variables()` applies computed state to world, performing collision detection and triggering callbacks
- The `predictive` parameter enables running physics twice (once for deterministic save/restore, once with side effects) to support client-side prediction rollback in networked play

**Fixed-Point Arithmetic Throughout** ΓÇö All positions/velocities stored as fixed-point (`_fixed` type) to guarantee deterministic results across CPUs/compilers. Conversion to world coordinates happens only at collision boundaries. This contrasts with modern engines using float32 + IEEE rounding guaranteesΓÇöhere, the approach is **explicitly deterministic by design**.

**Virtual Aim Buffering** ΓÇö `vir_aim_delta` (static global) reconstructs fractional aiming lost in low-precision network encoding (~8 bits for yaw/pitch over the wire). Input precision is ~16 bits internally; the residual is buffered frame-to-frame, ensuring smooth aiming despite quantized network transmission. Classic netcode optimization pattern.

**Physics Constants Indirection** ΓÇö `get_physics_constants_for_model()` selects from `physics_models[]` based on `_run_dont_walk` flag and level physics_model setting. Enables gameplay tuning via MML without recompilation; different maps/mods can have different gravity, acceleration, max speeds.

## Data Flow Through This File

```
Input Hardware (keyboard/mouse/gamepad)
  Γåô
Input subsystem ΓåÆ action_flags (packed bitfield of movement/aiming/action commands)
  Γåô
[Frame N] process_aim_input(action_flags, mouse_delta)
  ΓåÆ encodes high-precision delta into low-precision yaw/pitch payload
  ΓåÆ stores fractional residual in vir_aim_delta (carried to next frame)
  ΓåÆ returns updated action_flags with encoded aim
  Γåô
update_player_physics_variables(player_index, action_flags, predictive)
  ΓåÆ physics_update(): compute new velocity/position/direction (pure simulation)
  ΓåÆ instantiate_physics_variables(): apply to world (collision detection, polygon changes)
  Γåô
Updated player.variables.position/direction ΓåÆ Rendering (camera), Network (serialization)
Events (polygon changes, bumps) ΓåÆ GameWorld callbacks (trigger zones, platform activation)
```

## Learning Notes

**Soft Physics for Gameplay**: Code comments reveal intentional simplificationsΓÇö"physics model is too soft," sliding bugs with walls, issues with orthogonal collisions. The physics is **velocity-based, not force-based**, optimized for arcade-like feel over physical realism. The system privileges gameplay consistency and netplay determinism over accuracy.

**Trigonometric Lookup Tables**: All angles use `cosine_table[]` and `sine_table[]` pre-computed arrays, never calling `sin()`/`cos()`. Idiomatic to 90s/2000s game engines where CPU floating-point was expensive and determinism mattered. Modern engines use SIMD or GPU compute.

**Polygon-Centric World Model**: Movement is inherently polygon-relative via `supporting_polygon_index` tracking. When player crosses polygon boundaries, explicit `changed_polygon()` callbacks fire. This is characteristic of **BSP-tree-based engines**; modern mesh-based engines use continuous raycasting instead.

**Action Flags as Packed Bitfield**: The entire input state fits in a `uint32` with subfield macros (e.g., `SET_ABSOLUTE_YAW()`, `_run_dont_walk`). Efficient for netplay serialization in the 1990s/2000s; modern engines use struct-based messages, but competitive games still use similar bit-packing for bandwidth.

## Potential Issues

**Lingering Divergence Concerns**: The `DIVERGENCE_CHECK` debug code (conditional compilation, saved_points arrays, playback iteration tracking) suggests netplay had determinism bugs. The fact that this is still present in DEBUG mode indicates the developers didn't fully trust the physics + network + replay system, even after fixes.

**Dead Player Special Cases**: Dead players have simplified physics ("lose control, look down/up based on velocity"). This special-casing could mask bugs in velocity/gravity if conditional logic isn't precise.

**Platform Sliding Bugs**: The `fix_sliding_on_platforms` setting and velocity clamping (external_velocity.k) imply collision resolution was problematic. Comment: "continuously boot ourselves out of nearly-orthogonal walls by tiny amounts, resulting in a slide"ΓÇöa sign of **fundamental friction between the discrete collision detection and continuous velocity model**.

**Auto-Recenter State Machine Fragility**: Multiple flags (`_RECENTERING_BIT`, `FLAGS_WHICH_PREVENT_RECENTERING`) and conditional logic preventing absolute pitch mode during recentering suggest this feature had subtle bugs. Comment: "no way to cancel"ΓÇöindicates potential player frustration if recentering gets stuck.
