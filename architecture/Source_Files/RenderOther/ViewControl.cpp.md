# Source_Files/RenderOther/ViewControl.cpp

## File Purpose
Manages view rendering parameters including field-of-view (FOV), visual effects (fold/static effects on teleport), on-screen display font, and landscape texture options. Provides configuration parsing via Marathon Markup Language (MML) with state reset capabilities.

## Core Responsibilities
- Accessor functions for view settings (map visibility, teleport effects)
- FOV value management with smooth interpolation toward target values
- On-screen font initialization, scaling, and retrieval based on HUD scale level
- Landscape texture options storage and retrieval per collection/frame
- MML parsing and resetting for view and landscape configurations
- Backup/restore mechanism for configuration values

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `view_settings_definition` | struct | Boolean flags for map, fold effect, static effect, interlevel teleport in/out |
| `FOV_settings_definition` | struct | FOV values (Normal, ExtraVision, TunnelVision) + ChangeRate and FixHorizontalNotVertical flag |
| `LandscapeOptions` | struct | Landscape rendering: HorizExp, VertExp, OGL_AspRatExp, VertRepeat, Azimuth, SphereMap |
| `LandscapeOptionsEntry` | struct | Wraps LandscapeOptions with a Frame index for per-frame configuration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `view_settings` | `view_settings_definition` | static | Current view behavior settings |
| `FOV_settings` | `FOV_settings_definition` | static | Current FOV angles and change rate |
| `OnScreenFont` | `FontSpecifier` | static | Base on-screen display font specification |
| `LoadedOnScreenFont` | `FontSpecifier` | static | Scaled/loaded version used at runtime |
| `ScreenFontInited` | `bool` | static | Whether font has been initialized |
| `ScreenFontInitedSize` | `short` | static | Tracks current applied font size (-1 = uninitialized) |
| `LOList[NUMBER_OF_COLLECTIONS]` | `vector<LandscapeOptionsEntry>[]` | static | Landscape options per collection |
| `DefaultLandscape` | `LandscapeOptions` | static | Fallback landscape options when no match found |
| `original_view_settings` | `view_settings_definition*` | static | Backup of original view settings for reset |
| `original_FOV_settings` | `FOV_settings_definition*` | static | Backup of original FOV settings for reset |
| `original_OnScreenFont` | `FontSpecifier` | static | Backup of original font for reset |

## Key Functions / Methods

### GetOnScreenFont
- **Signature:** `FontSpecifier& GetOnScreenFont()`
- **Purpose:** Returns the on-screen font, scaling it dynamically based on HUD scale level and screen height.
- **Inputs:** None (reads `get_screen_mode()->hud_scale_level` and `MainScreenLogicalHeight()`)
- **Outputs/Return:** Reference to `LoadedOnScreenFont` (the scaled font)
- **Side effects:** Initializes or updates `LoadedOnScreenFont` size; sets `ScreenFontInited` and `ScreenFontInitedSize`; calls `LoadedOnScreenFont.Init()` or `LoadedOnScreenFont.Update()`
- **Calls:** `get_screen_mode()`, `MainScreenLogicalHeight()`, `LoadedOnScreenFont.Update()`, `LoadedOnScreenFont.Init()`
- **Notes:** Implements two-stage scaling: stage 1 doubles size if height > 960; stage 2 proportionally scales based on height vs. 480. Caches size to avoid redundant operations.

### View_AdjustFOV
- **Signature:** `bool View_AdjustFOV(float& FOV, float FOV_Target)`
- **Purpose:** Moves FOV value closer to a target, simulating smooth FOV transitions during gameplay.
- **Inputs:** `FOV` (current FOV), `FOV_Target` (goal FOV)
- **Outputs/Return:** `true` if FOV changed; `false` if already at target
- **Side effects:** Modifies `FOV` parameter in-place; may modify `FOV_ChangeRate` sign (ensures positive)
- **Calls:** `MAX()`, `MIN()` macros
- **Notes:** Clamps result to target after adjustment to avoid overshooting; uses `FOV_ChangeRate` per frame/tick.

### View_GetLandscapeOptions
- **Signature:** `LandscapeOptions *View_GetLandscapeOptions(shape_descriptor Desc)`
- **Purpose:** Retrieves landscape rendering options for a specific shape descriptor (collection + frame).
- **Inputs:** `shape_descriptor Desc` (packed shape/collection info)
- **Outputs/Return:** Pointer to matching `LandscapeOptions`, or `&DefaultLandscape` if no match
- **Side effects:** None (read-only lookup)
- **Calls:** `GET_DESCRIPTOR_SHAPE()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_COLLECTION()` macros
- **Notes:** Matches by frame or `AnyFrame` (-1); first match returned; constant-time lookup per collection vector.

### parse_mml_view
- **Signature:** `void parse_mml_view(const InfoTree& root)`
- **Purpose:** Parses MML XML configuration for view settings (map, effects, FOV, font).
- **Inputs:** `root` (InfoTree node containing view configuration)
- **Outputs/Return:** None (modifies static `view_settings`, `FOV_settings`, `OnScreenFont`)
- **Side effects:** Backs up original values (if not already done); reads XML attributes; calls `root.read_attr()` and `root.children_named()`
- **Calls:** `root.read_attr()`, `root.children_named()`, `font.read_font()`, `fov.read_attr_bounded()`
- **Notes:** Idempotent for backups (only allocates backup once); resets `ScreenFontInitedSize` on font change.

### parse_mml_landscapes
- **Signature:** `void parse_mml_landscapes(const InfoTree& root)`
- **Purpose:** Parses MML XML configuration for landscape texture options per collection/frame.
- **Inputs:** `root` (InfoTree node containing landscape definitions)
- **Outputs/Return:** None (modifies static `LOList`)
- **Side effects:** Iterates over XML children named "clear" or "landscape"; adds or replaces entries in `LOList[coll]`
- **Calls:** `child.read_indexed()`, `child.read_attr()`, `child.read_angle()`, vector `.push_back()`, `.clear()`
- **Notes:** "clear" elements delete a collection's or all landscapes; "landscape" elements add/replace frame entries; handles projection (1 = sphere map).

### reset_mml_view
- **Signature:** `void reset_mml_view()`
- **Purpose:** Resets view settings, FOV, and font to their original (pre-MML) values.
- **Inputs:** None
- **Outputs/Return:** None (modifies `view_settings`, `FOV_settings`, `OnScreenFont`)
- **Side effects:** Restores from backed-up originals; frees backup memory; resets `ScreenFontInitedSize`
- **Calls:** `free()`
- **Notes:** Safe to call if backups don't exist (checks null pointers).

### reset_mml_landscapes
- **Signature:** `void reset_mml_landscapes()`
- **Purpose:** Clears all landscape options.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears all vectors in `LOList`
- **Calls:** `LODeleteAll()` ΓåÆ `LODelete()` for each collection

## Control Flow Notes
This file is **configuration/initialization focused**. It participates in:
1. **Startup:** MML parsing (`parse_mml_*` called during engine init)
2. **Runtime queries:** Accessor functions called during rendering/gameplay to check view flags and retrieve parameters
3. **FOV transitions:** `View_AdjustFOV()` called per-frame to smoothly interpolate FOV (e.g., entering tunnel vision)
4. **Font rendering:** `GetOnScreenFont()` called when rendering HUD text
5. **Config reset:** `reset_mml_*` called when unloading or reloading MML

No frame loop participation; purely query/configuration logic.

## External Dependencies
- **includes:** `<vector>`, `<string.h>`, `cseries.h`, `world.h`, `SoundManager.h`, `shell.h`, `screen.h`, `ViewControl.h`, `InfoTree.h`, `preferences.h`
- **defined elsewhere:** `graphics_preferences` (global prefs), `get_screen_mode()`, `MainScreenLogicalHeight()`, `FontSpecifier`, `InfoTree::read_*()` methods, `GET_DESCRIPTOR_*()` macros, shape descriptor utilities
