# Source_Files/Lua/lua_hud_objects.cpp

## File Purpose
Implements Lua bindings for HUD (heads-up display) objects in the Aleph One game engine, exposing game world state, player information, rendering primitives, and interface configuration to Lua scripts. Provides 40+ Lua classes and enums for scripting custom HUDs and UI overlays.

## Core Responsibilities
- Define and register Lua wrapper classes for HUD-related engine objects (images, shapes, fonts, screen state)
- Expose game state and player data to Lua (ticks, difficulty, scoring mode, kill limits, player roster)
- Provide color and geometry accessors for HUD interface elements
- Implement drawing functions for text, images, and shapes
- Register enum types for game constants (renderer types, masking modes, texture types, player colors, etc.)
- Initialize and validate Lua type hierarchy during engine startup

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Image` | typedef (L_ObjectClass) | Wrapper around `Image_Blitter*` for bitmap rendering |
| `Lua_Shape` | typedef (L_ObjectClass) | Wrapper around `Shape_Blitter*` for shape rendering |
| `Lua_Font` | class | Custom subclass of `L_ObjectClass<FontSpecifier*>` with scale support |
| `Lua_Screen` | typedef (L_Class) | Singleton HUD screen properties and drawing methods |
| `Lua_HUDGame` | typedef (L_Class) | Read-only game state (difficulty, game type, scoring mode, ticks) |
| `Lua_HUDGame_Players` | typedef (L_Class) | Indexed collection of player data |
| `Lua_HUDPlayer` | typedef (L_Class) | Player HUD state (weapons, items, velocity, compass) |
| `Lua_HUDLevel` | typedef (L_Class) | Level metadata (name, index, map checksum, persistent stash) |
| `Lua_HUDLighting` | typedef (L_Class) | Lighting state and fader effects |
| `Lua_InterfaceColor` | typedef (L_Enum) | Named color from interface palette |
| `Lua_TextureType`, `Lua_RendererType`, `Lua_SensorBlipType` | typedef (L_Enum) | Game constants with mnemonic names |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `world_view` | `struct view_data*` | extern | Current camera/view state |
| `current_player` | `player_data*` | extern | Active player pointer |
| `dynamic_world` | world state | extern | Runtime game state (ticks, difficulty, player count) |
| `static_world` | map data | extern | Level metadata (name) |
| `lua_stash` | `unordered_map<string,string>` | extern | Persistent key-value storage per level |
| `use_lua_hud_crosshairs` | `bool` | extern | Toggles Lua-scripted crosshair rendering |
| `AngleConvert` | `const float` | file-static | Conversion factor: `360/FULL_CIRCLE` |

## Key Functions / Methods

### Lua_HUDColor_Lookup
- **Signature:** `static float Lua_HUDColor_Lookup(lua_State *L, int pos, int idx, const char *name1, const char *name2)`
- **Purpose:** Extract RGBA components from Lua color tables with flexible field naming (e.g., "r"/"red")
- **Inputs:** Lua state, stack position of table, numeric index fallback, two alternate field names
- **Outputs/Return:** Float value [0.0, 1.0] for color component; defaults to 1.0 if not found
- **Side effects:** Manipulates Lua stack (pops values)
- **Calls:** `lua_pushstring`, `lua_gettable`, `lua_tonumber`, `lua_pop`
- **Notes:** Tries first name, then second name, then numeric index; returns 1.0 (white/opaque) if all missing

### Lua_Images_New
- **Signature:** `int Lua_Images_New(lua_State *L)`
- **Purpose:** Factory function to create `Image_Blitter` from either resource ID or file path with optional mask
- **Inputs:** Lua table with keys: `resource` (int), `path` (string), `mask` (string)
- **Outputs/Return:** 1 (pushes Image userdata) or 1 (pushes nil on failure)
- **Side effects:** Allocates `Image_Blitter`; loads image/mask files; may select OpenGL or CPU blitter based on acceleration mode
- **Calls:** `Lua_Image::Push`, `ImageDescriptor::LoadFromFile`, `FileSpecifier::SetNameWithPath`, `new Image_Blitter`, `new OGL_Blitter`
- **Notes:** Prefers resource ID over path; both can fail gracefully; mask loading failure is silent

### Lua_Fonts_New
- **Signature:** `int Lua_Fonts_New(lua_State *L)`
- **Purpose:** Create `FontSpecifier` from Lua table with optional interface font template or custom file
- **Inputs:** Lua table with keys: `interface` (enum), `file` (string), `size` (int), `style` (int)
- **Outputs/Return:** 1 (pushes Font userdata) or 1 (pushes nil if line spacing Γëñ 0)
- **Side effects:** Allocates and initializes `FontSpecifier`; may reset OpenGL state; applies scoped search path
- **Calls:** `get_interface_font`, `FontSpecifier::Init`, `Lua_Font::Push`
- **Notes:** Defaults to Monaco 12pt; OpenGL fonts use HUD texture filter; validates line spacing > 0

### Lua_Screen_Fill_Rect
- **Signature:** `int Lua_Screen_Fill_Rect(lua_State *L)`
- **Purpose:** Draw filled rectangle to HUD with color
- **Inputs:** Numbers at stack positions 1ΓÇô4 (x, y, w, h); Lua table at position 5 (color)
- **Outputs/Return:** 0 (no return value)
- **Side effects:** Queues drawing operation via `Lua_HUDInstance()->fill_rect()`
- **Calls:** `Lua_HUDColor_Get_R/G/B/A`, `Lua_HUDInstance`
- **Notes:** Color table keys: "r"/"red", "g"/"green", "b"/"blue", "a"/"alpha"; values [0.0, 1.0]

### Lua_HUDGame_Get_Difficulty
- **Signature:** `static int Lua_HUDGame_Get_Difficulty(lua_State *L)`
- **Purpose:** Retrieve current game difficulty level
- **Inputs:** Lua state
- **Outputs/Return:** 1 (pushes Difficulty enum)
- **Side effects:** Pushes value onto stack
- **Calls:** `Lua_DifficultyType::Push`, `dynamic_world->game_information.difficulty_level`
- **Notes:** Read-only; reflects game_information state

### Lua_HUDObjects_register
- **Signature:** `int Lua_HUDObjects_register(lua_State *L)`
- **Purpose:** **Main entry point** ΓÇô registers all HUD classes, enums, and globals to Lua
- **Inputs:** Lua state (usually from game startup)
- **Outputs/Return:** 0
- **Side effects:** Registers 40+ Lua types; creates global `Screen`, `Game`, `Level`, `Player`, `Fonts`, `Images`, `Shapes`, `Lighting` objects; sets up method tables and validation functions
- **Calls:** `Lua_Collection::Register`, `Lua_DifficultyType::Register`, etc. (one per major type)
- **Notes:** Must be called exactly once during Lua initialization; each type's `Register()` method configures metatable, getter/setter functions, and validity checks

## Control Flow Notes
This is a **registration and accessor file** with no active frame loop. Control flow is:
1. **Initialization:** `Lua_HUDObjects_register()` is called once at Lua VM startup
2. **Runtime:** Lua scripts call getter/setter functions (e.g., `Screen.width`, `Game.ticks`) which invoke C functions in this file
3. **Queries/Actions:** Draw calls (`Screen.fill_rect()`, `Image.draw()`) queue operations via `Lua_HUDInstance()`
4. **No teardown:** Objects are cleaned up by Lua garbage collection (see `__gc` metamethods on Image, Shape, Font)

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (FFI layer)
- **Game engine core:** `world_view`, `dynamic_world`, `static_world`, `current_player`, `local_player_index` (game state)
- **HUD/Render:** `HUDRenderer.h`, `HUDRenderer_Lua.h`, `Image_Blitter.h`, `OGL_Blitter.h`, `Shape_Blitter.h`, `FontHandler.h` (drawing primitives)
- **Game world:** `items.h`, `player.h`, `motion_sensor.h`, `screen.h`, `map.h` (object definitions)
- **Game constants:** `alephversion.h`, `collection_definition.h` (defines accessed via extern functions)
- **Utilities:** `std::unordered_map`, `std::algorithm`, `std::cmath` (C++ stdlib)
