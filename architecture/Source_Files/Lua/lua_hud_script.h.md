# Source_Files/Lua/lua_hud_script.h

## File Purpose
Header file declaring the public interface for Lua-based HUD (Heads-Up Display) scripting in Aleph One. Provides lifecycle callbacks for HUD initialization, drawing, and cleanup, along with script loading and management functions.

## Core Responsibilities
- Declare HUD lifecycle callbacks (Init, Cleanup, Draw, Resize)
- Declare script loading and execution functions
- Provide HUD script path management (set/get)
- Expose Lua garbage collection marking for HUD objects
- Query HUD script runtime state

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### L_Call_HUDInit
- Signature: `void L_Call_HUDInit()`
- Purpose: Invoke the Lua HUD initialization callback
- Inputs: None
- Outputs/Return: None
- Side effects: Executes Lua code; may modify HUD state
- Calls: (defined elsewhere)
- Notes: Called during HUD setup phase

### L_Call_HUDCleanup
- Signature: `void L_Call_HUDCleanup()`
- Purpose: Invoke the Lua HUD cleanup callback
- Inputs: None
- Outputs/Return: None
- Side effects: Executes Lua code; may release HUD resources
- Calls: (defined elsewhere)
- Notes: Called during shutdown/teardown phase

### L_Call_HUDDraw
- Signature: `void L_Call_HUDDraw()`
- Purpose: Invoke the Lua HUD render callback
- Inputs: None
- Outputs/Return: None
- Side effects: Executes Lua drawing commands; modifies framebuffer
- Calls: (defined elsewhere)
- Notes: Called each frame during rendering

### L_Call_HUDResize
- Signature: `void L_Call_HUDResize()`
- Purpose: Invoke the Lua HUD resize callback
- Inputs: None
- Outputs/Return: None
- Side effects: Executes Lua code; may recalculate HUD layout
- Calls: (defined elsewhere)
- Notes: Called on viewport/window resize

### LoadLuaHUDScript
- Signature: `bool LoadLuaHUDScript(const char *buffer, size_t len)`
- Purpose: Load and compile a Lua HUD script from a memory buffer
- Inputs: `buffer` (script source), `len` (buffer size in bytes)
- Outputs/Return: `bool` (success/failure)
- Side effects: Parses Lua; initializes script state
- Calls: (defined elsewhere; likely Lua C API)
- Notes: Script is compiled but not executed; check RunLuaHUDScript()

### RunLuaHUDScript
- Signature: `bool RunLuaHUDScript()`
- Purpose: Execute the loaded Lua HUD script
- Inputs: None
- Outputs/Return: `bool` (success/failure)
- Side effects: Executes Lua code; initializes HUD state
- Calls: (defined elsewhere)
- Notes: Must call LoadLuaHUDScript() first

### LuaHUDRunning
- Signature: `bool LuaHUDRunning()`
- Purpose: Query whether a HUD script is loaded and active
- Inputs: None
- Outputs/Return: `bool` (true if script is active)
- Side effects: None
- Calls: (defined elsewhere)
- Notes: State query function

### SetLuaHUDScriptPath / GetLuaHUDScriptPath
- Signature: `void SetLuaHUDScriptPath(const std::string& path)` / `std::string GetLuaHUDScriptPath()`
- Purpose: Manage the file system path to the HUD script
- Inputs: (path reference for setter)
- Outputs/Return: (path string for getter)
- Side effects: (setter) updates internal path state
- Calls: (defined elsewhere)
- Notes: Decouples script location from loading logic

### MarkLuaHUDCollections
- Signature: `void MarkLuaHUDCollections(bool loading)`
- Purpose: Mark HUD Lua objects for garbage collection (GC sweep)
- Inputs: `loading` (true during load phase, false during unload)
- Outputs/Return: None
- Side effects: Updates Lua GC state; may deallocate objects
- Calls: (defined elsewhere; likely Lua GC API)
- Notes: Supports proper lifetime management of Lua objects in memory

Trivial helpers: `LoadHUDLua()`, `CloseLuaHUDScript()` ΓÇö likely wrappers for load-from-file and cleanup respectively.

## Control Flow Notes
Typical HUD lifecycle:
1. **Load phase**: `LoadLuaHUDScript()` or `LoadHUDLua()` ΓåÆ `RunLuaHUDScript()` ΓåÆ `L_Call_HUDInit()`
2. **Frame loop**: `L_Call_HUDDraw()` (per frame), `L_Call_HUDResize()` (on window events)
3. **Shutdown**: `L_Call_HUDCleanup()` ΓåÆ `CloseLuaHUDScript()` ΓåÆ `MarkLuaHUDCollections(false)`

## External Dependencies
- `cseries.h` ΓÇö platform abstraction, SDL2 integration, type definitions
- `<string>` ΓÇö C++ STL for path management
- Lua C API ΓÇö Not directly included; used by implementation (defined elsewhere)
