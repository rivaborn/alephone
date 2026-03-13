# Source_Files/Lua/lua_hud_script.cpp

## File Purpose
Implements Lua HUD state management and lifecycle for the Aleph One game engine. Manages loading, initialization, and execution of Lua scripts that customize the heads-up display, including trigger callbacks for init, draw, resize, and cleanup events.

## Core Responsibilities
- Create and maintain a singleton Lua VM instance for HUD scripting
- Load Lua HUD scripts from plugin files or buffers
- Execute script trigger callbacks (init, draw, resize, cleanup) at appropriate lifecycle points
- Register game engine functions accessible to Lua scripts (via `Lua_HUDObjects_register`)
- Track which game asset collections are required by the Lua script
- Manage script lifecycle: load ΓåÆ initialize ΓåÆ run ΓåÆ draw ΓåÆ cleanup

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `LuaHUDState` | class | Wraps Lua VM state and manages trigger dispatch, script loading, and execution |
| `luaL_Reg` | struct | Lua library registration (from lua.h); maps library name to init function |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `hud_state` | `LuaHUDState*` | global | Singleton pointer to active HUD Lua state (NULL if uninitialized) |
| `use_lua_hud_crosshairs` | bool | global | Flag set by `RunLuaHUDScript()` to indicate HUD crosshairs should be used |
| `lualibs[]` | `luaL_Reg[]` | static | Array of standard Lua libraries to load (base, table, string, bit32, math, debug, io) |
| `AngleConvert` | float | extern global | Used by several functions; defined elsewhere |
| `MotionSensorActive` | bool | extern global | Game state queried by HUD logic |
| `world_view` | `view_data*` | extern global | Current view context |
| `static_world` | `static_data*` | extern global | Static world data |

## Key Functions / Methods

### LoadLuaHUDScript
- **Signature:** `bool LoadLuaHUDScript(const char *buffer, size_t len)`
- **Purpose:** Load a Lua script from memory buffer; creates `hud_state` singleton if needed
- **Inputs:** `buffer` (Lua source code), `len` (byte count)
- **Outputs/Return:** `true` if load succeeded, `false` on syntax/memory error
- **Side effects:** Initializes `hud_state` on first call; increments `num_scripts_` counter; logs warnings on error
- **Calls:** `LuaHUDState::Initialize()`, `LuaHUDState::Load()`
- **Notes:** Does not execute the script; only loads it into Lua state

### RunLuaHUDScript
- **Signature:** `bool RunLuaHUDScript()`
- **Purpose:** Execute all loaded Lua scripts; returns `true` on success
- **Inputs:** None
- **Outputs/Return:** `true` if all scripts executed without runtime error
- **Side effects:** Sets `running_ = true` on success; resets `use_lua_hud_crosshairs = false`
- **Calls:** `LuaHUDState::Run()` via `hud_state`
- **Notes:** Scripts must be loaded first; does not call trigger callbacks

### LoadHUDLua
- **Signature:** `void LoadHUDLua()`
- **Purpose:** Discover and load HUD Lua script from plugin system; top-level initialization
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Creates/initializes `hud_state`; reads file from disk; calls `LoadLuaHUDScript()` and `SetLuaHUDScriptSearchPath()`
- **Calls:** `Plugins::instance()->find_hud_lua()`, `FileSpecifier::Open()`, `LoadLuaHUDScript()`, `SetLuaHUDScriptSearchPath()`
- **Notes:** Silently succeeds even if no plugin defines a HUD Lua file

### L_Call_HUDInit, L_Call_HUDDraw, L_Call_HUDResize, L_Call_HUDCleanup
- **Signature:** `void L_Call_HUD*()` (4 functions)
- **Purpose:** Dispatch trigger callbacks from game engine to Lua (init, draw, resize, cleanup)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls corresponding `LuaHUDState::Init()`, `Draw()`, `Resize()`, `Cleanup()`
- **Calls:** `LuaHUDState::Init/Draw/Resize/Cleanup()`
- **Notes:** Null-check `hud_state` before dispatch; safe to call when HUD Lua not loaded

### CloseLuaHUDScript
- **Signature:** `void CloseLuaHUDScript()`
- **Purpose:** Shutdown HUD Lua VM and cleanup
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deletes `hud_state` singleton; sets to NULL
- **Calls:** Destructor of `LuaHUDState`
- **Notes:** Lua state closed via `lua_close` in destructor (via shared_ptr deleter)

### MarkLuaHUDCollections
- **Signature:** `void MarkLuaHUDCollections(bool loading)`
- **Purpose:** Track which game asset collections are needed; defer load/unload as a batch
- **Inputs:** `loading` (true = load phase, false = unload phase)
- **Outputs/Return:** None
- **Side effects:** Calls `hud_state->MarkCollections()` during load phase; calls `mark_collection_for_unloading()` during unload phase
- **Calls:** `LuaHUDState::MarkCollections()`, `mark_collection_for_unloading()` (defined elsewhere)
- **Notes:** Uses static local to preserve collection set across load/unload pairs

## Control Flow Notes
**Initialization:** `LoadHUDLua()` ΓåÆ `LoadLuaHUDScript()` ΓåÆ `LuaHUDState::Initialize()` (load Lua std libs, register HUD functions)

**Execution:** `RunLuaHUDScript()` ΓåÆ all loaded chunks executed; then `L_Call_HUDInit()` dispatches Lua `Triggers.init()` callback

**Per-frame:** `L_Call_HUDDraw()` dispatches `Triggers.draw()` callback from rendering loop

**Shutdown:** `L_Call_HUDCleanup()` ΓåÆ `CloseLuaHUDScript()` ΓåÆ Lua VM destroyed

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö VM creation, state management, stack operations, library loading
- **Game engine:** `interface.h`, `mouse.h` ΓÇö view/input state (defined elsewhere)
- **Plugins:** `Plugins.h` ΓÇö discover HUD script in plugin metadata
- **Logging:** `Logging.h` ΓÇö error/warning output
- **Preferences:** `preferences.h` ΓÇö game configuration (not directly used in this file)
- **Boost iostreams:** `<boost/iostreams/*.hpp>` ΓÇö included but not used in visible code
- **File I/O:** `FileSpecifier`, `OpenedFile` ΓÇö load script from disk (defined elsewhere)
- **HUD objects:** `lua_hud_objects.h` ΓÇö `Lua_HUDObjects_register()` binds game functions to Lua (defined elsewhere)
