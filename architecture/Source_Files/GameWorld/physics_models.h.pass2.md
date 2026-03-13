# Source_Files/GameWorld/physics_models.h - Enhanced Analysis

## Architectural Role

This file serves as the **physics parameter dictionary** for the GameWorld subsystem, defining the two character movement models (walking, running) that govern all player and monster locomotion. It acts as a critical data interface between the Files subsystem (which deserializes physics from binary WAD data via `game_wad.cpp` and `import_definitions.cpp`) and the physics simulation engine (`physics.cpp`). The hardcoded `original_physics_models` array provides immutable defaults, while the serialization functions enable runtime physics customization via loaded mapsΓÇöessential for Marathon's modding ecosystem.

## Key Cross-References

### Incoming (who depends on this)
- **Files subsystem** (`game_wad.cpp`, `import_definitions.cpp`): calls `unpack_physics_constants()` and `unpack_m1_physics_constants()` during WAD parsing and legacy map import to load custom physics into the runtime engine
- **GameWorld initialization** (`marathon2.cpp`, startup phase): calls `init_physics_constants()` at engine startup to populate the active physics state from defaults
- **Physics simulation** (`physics.cpp`, `players.cpp`): reads from `original_physics_models[model_type]` and runtime-loaded physics constants during per-frame movement calculations and collision response
- **Monster AI** (`monsters.cpp`): accesses physics constants for movement speed, acceleration, and climbing behavior when computing monster trajectories and pathfinding

### Outgoing (what this depends on)
- **CSeries/world.h**: provides `_fixed` fixed-point type, `FIXED_ONE` macro (unit scalar), `QUARTER_CIRCLE` constant (angle unit for rotations), and cross-platform type compatibility
- **Standard C types** (`uint8`, `size_t`): sized integer types for portable binary I/O

## Design Patterns & Rationale

**Fixed-Point Physics Parameters**: All velocity, acceleration, and distance values use `_fixed` (fixed-point arithmetic) rather than floating-point. This ensures **deterministic physics across platforms**ΓÇöcritical for Marathon's peer-to-peer multiplayer where all clients must simulate identical physics independently. The hardcoded rational fractions (e.g., `FIXED_ONE/14`) are multiply-accurate precision values tuned during the original 1994 design.

**Const Data Table with Enum Indexing**: The `enum` (`_model_game_walking`, `_model_game_running`) directly indexes the `original_physics_models[]` array. This pattern avoids runtime lookup tables, string matching, or hash tablesΓÇöO(1) direct access was essential on 1990s hardware. The const qualifier guarantees these values never change during play.

**Versioned Binary Serialization**: The presence of both `unpack_physics_constants()` and `unpack_m1_physics_constants()` reflects **backward compatibility by format versioning**. When Marathon evolved from version 1 to Infinity, physics constants layout likely changed. Rather than deprecate old maps, Aleph One includes a legacy unpacker to translate Marathon 1 format into the current `physics_constants` struct. This is typical of game engines that must support user-created content across versions.

**Stream-Based Pack/Unpack**: The `uint8 *` return value (advanced stream pointer) follows a **stream-cursor pattern** common in binary I/O before C++ streams. Each function advances the pointer and returns the new position, enabling sequential packing of heterogeneous structures (e.g., multiple physics constants, followed by other map data). See `AStream.h`/`BStream.h` for similar patterns throughout Files subsystem.

## Data Flow Through This File

**Initialization Phase (startup)**:
1. `init_physics_constants()` is called during engine bootstrap
2. Populates internal physics state structures (implementation in `physics.cpp`) from `original_physics_models[]` defaults

**Map Load Phase (when opening a level)**:
1. `game_wad.cpp` parses the WAD file header and identifies physics data chunks
2. Calls `unpack_physics_constants(stream, objects, count)` or `unpack_m1_physics_constants(stream, count)` to deserialize from binary
3. Loaded physics constants override or extend the hardcoded defaults
4. Physics engine now uses custom constants for this map's monsters/player movement

**Runtime Simulation (per-frame)**:
1. `physics.cpp` queries `original_physics_models[current_model]` or runtime-loaded physics constants
2. Applies acceleration/deceleration to player/monster velocity
3. Integrates gravity, collision constraints, media (liquid) drag
4. Movement model selection determined by game state (walking vs. running)

## Learning Notes

**Era-Specific Design**: This file exemplifies **1990s game engine physics design**:
- **Fixed-point arithmetic** replaced floats for reproducibility before IEEE 754 standardization across platforms
- **Hardcoded tuning constants** (FIXED_ONE/14) assume deep domain expertise; modern engines use YAML configs or physics libraries
- **Enum-based model selection** predates inheritance/polymorphism patterns; direct indexing was the performance norm
- **Binary serialization** with manual pack/unpack replaced by JSON/Protobuf in modern engines
- **Const tables in headers** avoids runtime initialization overheadΓÇötoday's linker optimization achieves this automatically

**Modding Affordance**: The public serialization functions reveal Marathon's **modding philosophy**: physics is a tunable parameter, not engine magic. Custom WADs can redefine movement speed, gravity, and sizeΓÇöenabling drastically different gameplay (e.g., low-gravity mods, super-fast runners) without source code changes.

**Determinism Priority**: The exclusive use of fixed-point arithmetic and hardcoded compile-time constants shows **determinism was prioritized over precision**. Floating-point variance (rounding, CPU differences) would break multiplayer sync; exact replication across machines was non-negotiable.

## Potential Issues

**No Range Validation on Unpack**: The `unpack_*` functions deserialize raw bytes into `physics_constants` structs with no bounds checking. Malformed WAD data (e.g., zero values for `acceleration`, `deceleration`, `gravitational_acceleration`) could cause division-by-zero or stall-like behavior in `physics.cpp` integration. A corrupted map file could crash or freeze the simulation.

**Stream Overrun Risk**: The pack/unpack functions accept a `count` parameter but lack length validation against the stream's actual size. Reading beyond end-of-stream would access uninitialized memory, causing undefined behavior and potential security issues in online play.

**Rigid Structure Evolution**: The 21-field `physics_constants` struct is baked into binary WAD files. Adding a new physics parameter (e.g., air-resistance coefficient) would break compatibility with all existing maps and require versioning all serialization functionsΓÇöthere is no forward-compatible extensibility mechanism.

**Magic Number Opacity**: Constants like `FIXED_ONE/400` for `gravitational_acceleration` lack semantic documentation. Future maintainers must reverse-engineer whether the value is tuned or accidental by comparing to gameplay feel. A comment like `/* ~10 pixels/sec┬▓ *` would help.
