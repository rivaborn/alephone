# Source_Files/Lua/lua_hud_objects.h

## File Purpose

Header file declaring Lua bindings for HUD objects in the Aleph One game engine. Provides Lua script access to HUD player state, game information, screen properties, and UI-related enumerations (colors, fonts, rectangles, inventory sections, renderer types, sensor blips, textures).

## Core Responsibilities

- Declare extern name strings for each Lua class and enumeration type
- Define typedef aliases for L_Class template instances (HUDPlayer, HUDGame, HUDScreen)
- Define typedef aliases for L_Enum and L_EnumContainer template instances for UI/game enums
- Provide the registration function to bind all HUD types to a Lua state
- Export types for use by Lua binding implementations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Lua_HUDPlayer | typedef (L_Class) | Lua binding for player state in HUD context |
| Lua_HUDGame | typedef (L_Class) | Lua binding for game information accessible from HUD |
| Lua_HUDScreen | typedef (L_Class) | Lua binding for screen/rendering information |
| Lua_InterfaceColor / Colors | typedef (L_Enum / L_EnumContainer) | UI color enumeration and container |
| Lua_InterfaceFont / Fonts | typedef (L_Enum / L_EnumContainer) | UI font enumeration and container |
| Lua_InterfaceRect / Rects | typedef (L_Enum / L_EnumContainer) | Predefined screen rectangle enumeration |
| Lua_InventorySection / Sections | typedef (L_Enum / L_EnumContainer) | Inventory section type enumeration |
| Lua_RendererType / Types | typedef (L_Enum / L_EnumContainer) | Renderer type enumeration |
| Lua_SensorBlipType / Types | typedef (L_Enum / L_EnumContainer) | Motion sensor blip type enumeration |
| Lua_TextureType / Types | typedef (L_Enum / L_EnumContainer) | Texture type enumeration |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Lua_HUDPlayer_Name, Lua_HUDGame_Name, Lua_HUDScreen_Name | extern char[] | extern | Class name strings for Lua metatable registration |
| Lua_InterfaceColor_Name, Lua_InterfaceColors_Name | extern char[] | extern | Name strings for color enum and container |
| Lua_InterfaceFont_Name, Lua_InterfaceFonts_Name | extern char[] | extern | Name strings for font enum and container |
| Lua_InterfaceRect_Name, Lua_InterfaceRects_Name | extern char[] | extern | Name strings for rect enum and container |
| Lua_InventorySection_Name, Lua_InventorySections_Name | extern char[] | extern | Name strings for inventory section enum |
| Lua_RendererType_Name, Lua_RendererTypes_Name | extern char[] | extern | Name strings for renderer type enum |
| Lua_SensorBlipType_Name, Lua_SensorBlipTypes_Name | extern char[] | extern | Name strings for sensor blip enum |
| Lua_TextureType_Name, Lua_TextureTypes_Name | extern char[] | extern | Name strings for texture type enum |
| NUMBER_OF_LUA_TEXTURE_TYPES | #define (5) | static | Constant for array bounds |

## Key Functions / Methods

### Lua_HUDObjects_register

- **Signature:** `int Lua_HUDObjects_register(lua_State *L);`
- **Purpose:** Register all HUD object types and enumerations with the Lua interpreter.
- **Inputs:** `lua_State *L` ΓÇô Lua interpreter state
- **Outputs/Return:** `int` ΓÇô Status code (0 for success, error code otherwise)
- **Side effects:** Modifies Lua state by registering metatables, C functions, and enumerations; makes HUD classes and enums available to Lua scripts
- **Calls:** (Not visibleΓÇöimplementation in corresponding .cpp file; likely calls `Register()` methods on each L_Class/L_Enum type)
- **Notes:** This is the sole public entry point. Must be called during HUD system initialization to enable Lua script access to HUD functionality.

## Control Flow Notes

This is a declaration-only header. Actual control flow resides elsewhere:
- **Initialization:** `Lua_HUDObjects_register()` called once to set up all Lua bindings
- **Runtime:** Lua scripts access HUD player, game state, and screen properties via registered classes; enumerations provide mnemonics for UI constants (colors, fonts, rectangles, etc.)
- The implementation relies on template definitions in `lua_templates.h` (L_Class, L_Enum, L_EnumContainer) to generate boilerplate accessor and registration code.

## External Dependencies

- **cseries.h** ΓÇô Platform abstraction and core types
- **lua.h, lauxlib.h, lualib.h** ΓÇô Lua 5.2 C API
- **items.h, map.h** ΓÇô Game world definitions (for context; not directly used in bindings)
- **lua_templates.h** ΓÇô Template classes (L_Class, L_Enum, L_EnumContainer, L_ObjectClass) that implement Lua-to-C++ binding machinery
