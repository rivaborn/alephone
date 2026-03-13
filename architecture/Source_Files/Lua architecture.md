# Subsystem Overview

## Purpose

The Lua subsystem provides scripting capabilities to Aleph One, enabling map creators and modders to customize game behavior through Lua scripts. It manages script loading, execution, event dispatch, and state serialization, along with exposing game world objects (entities, map geometry, player state, audio, HUD) to Lua via a C API binding layer.

## Key Files

| File | Role |
|------|------|
| lua_script.cpp/h | Core script loading, execution, event dispatch, and state management |
| lua_templates.h | C++ template classes for binding C++ objects to Lua as userdata |
| lua_serialize.cpp/h | Binary serialization/deserialization of Lua state |
| lua_mnemonics.h | String-to-integer mnemonic lookup tables for game constants |
| language_definition.h | Compile-time constant definitions for script symbols |
| lua_map.cpp/h | Bindings for map geometry (polygons, lines, platforms, lights, media) |
| lua_monsters.cpp/h | Bindings for monster entities and properties |
| lua_objects.cpp/h | Bindings for map objects (effects, items, scenery) |
| lua_player.cpp/h | Bindings for player state (position, inventory, weapons, energy) |
| lua_projectiles.cpp/h | Bindings for projectile entities |
| lua_hud_objects.cpp/h | Bindings for HUD elements and game state |
| lua_hud_script.cpp/h | HUD script lifecycle management (init, draw, cleanup) |
| lua_music.cpp/h | Bindings for music system control |
| lua_ephemera.cpp/h | Bindings for ephemera (temporary visual effects) |
| lua_saved_objects.cpp/h | Read-only bindings for map-defined objects (goals, spawns, sounds) |
| lua.h, lauxlib.h, lualib.h, luaconf.h, etc. | Lua 5.2 core interpreter headers |

## Core Responsibilities

- **Script lifecycle**: Load Lua scripts from map files, plugins, and buffers; initialize VM state with engine functions and sandbox configuration
- **Event dispatch**: Route game events (player actions, damage, kills, item pickup, terminal interaction, level state changes) to registered Lua callbacks
- **Game world exposure**: Expose map entities, geometry, physics objects, and game state to Lua through typed bindings with getter/setter methods
- **State persistence**: Serialize/deserialize Lua objects to binary format with reference deduplication for game save/load
- **HUD scripting**: Manage HUD script lifecycle (init, draw, resize, cleanup) with callbacks for custom interface overlays
- **Mutability control**: Enforce read-only vs. read-write access to game systems via `LuaMutabilityInterface` to restrict script capabilities by context
- **Input/output integration**: Route Lua-requested actions (weapon change, camera animation, console output) into game engine, manage action queues
- **Constant definitions**: Provide symbolic name-to-code mappings for weapons, items, monsters, damage types, game modes, and other enumerations

## Key Interfaces & Data Flow

**Inbound (to Lua subsystem):**
- Game events: damage, kills, projectile hits, item interactions, switch triggers, terminal access, level state changes
- Script load requests: from map headers, plugin metadata, or console commands
- Game state queries: player position/inventory, monster list, map geometry, game rules

**Outbound (from Lua subsystem):**
- Callback dispatch to registered Lua functions for game events
- State modifications: player inventory, weapon selection, monster properties, ephemera creation/deletion, HUD overlay updates
- Console output and error reporting
- Camera animation and cutscene control

**Internal data flow:**
- Lua C API as bridge: C++ objects wrapped as userdata ΓåÆ Lua stack manipulation ΓåÆ registered getter/setter functions
- Script state serialization via binary stream (BOStream/BIStream)
- Mnemonic lookup tables convert string names Γåö numeric engine constants
- Template system (`L_Class`, `L_Container`, `L_Enum`) generates boilerplate Lua FFI code

## Runtime Role

- **Initialization**: Scripts loaded at engine startup (embedded/netscript contexts) or level load (solo/map contexts); Lua VMs created per context
- **Frame**: Per-frame Lua callbacks executed (e.g., `idle()` for HUD scripts, event handlers triggered by game actions)
- **Event handling**: Game systems invoke Lua event handlers asynchronously (e.g., player damaged ΓåÆ call `damaged_player` callback)
- **Shutdown**: Lua state serialized to persistent storage, VMs destroyed, script cleanup callbacks invoked

## Notable Implementation Details

- **Lua 5.2 core**: Full Lua interpreter embedded; configured via `luaconf.h` with game engine feature flags (`OPENGL`, `SDL_net`, `zlib`, etc.)
- **Template-based bindings**: `L_Class<>`, `L_Enum<>`, `L_Container<>` metaprogramming eliminates boilerplate; supports both indexed and named access
- **Multiple script contexts**: Separate Lua VMs for embedded scripts, solo levels, netscript (multiplayer), stats, and achievements with independent persistence
- **Serialization with deduplication**: Circular references and shared objects tracked via reference tables during save/restore to preserve object identity
- **Backward compatibility layer**: Deprecated Lua APIs wrapped via compatibility functions; old scripts continue working
- **Mutability enforcement**: Gate write operations through `LuaMutabilityInterface`; read-only contexts prevent unintended world modifications
- **Binary chunk precompilation**: Lua bytecode can be dumped and reloaded for faster script startup
