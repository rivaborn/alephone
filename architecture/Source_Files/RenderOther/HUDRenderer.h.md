# Source_Files/RenderOther/HUDRenderer.h

## File Purpose
Base class and data structures for rendering the heads-up display (HUD) in the Aleph One game engine. Defines interface for weapon panels, ammo displays, health/oxygen bars, motion sensors, and inventory visualization. Serves as the architectural foundation for platform-specific HUD implementations.

## Core Responsibilities
- Define abstract HUD rendering interface via `HUD_Class` base class
- Manage weapon panel display configuration and data structures
- Track HUD element dirty states to optimize redraw performance
- Coordinate updates for suit energy, oxygen, weapons, ammunition, and inventory
- Provide virtual methods for platform-specific shape drawing, text rendering, and clipping
- Define shape descriptor enums for HUD textures (bars, weapon icons, motion sensor elements)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `weapon_interface_ammo_data` | struct | Defines ammo display layout per weapon (position, bullet shapes, fill direction) |
| `weapon_interface_data` | struct | Complete weapon panel configuration (panel shape, weapon name bounds, ammo data array) |
| `interface_state_data` | struct | Tracks dirty flags for ammo, weapon, shield, oxygen to optimize redraws |
| `HUD_Class` | class | Abstract base class for HUD renderer implementations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `interface_state` | `interface_state_data` | global | Tracks which HUD elements are dirty and need redrawing |
| `weapon_interface_definitions` | `weapon_interface_data[10]` | global | Array of weapon panel configurations loaded from game resources |

## Key Functions / Methods

### update_everything
- **Signature**: `bool update_everything(short time_elapsed)`
- **Purpose**: Main HUD update entry point; coordinates all HUD element updates
- **Inputs**: `time_elapsed` ΓÇô milliseconds since last update
- **Outputs/Return**: `bool` ΓÇô true if screen was updated
- **Side effects**: Modifies `interface_state` dirty flags; may trigger redraw of any HUD element
- **Calls**: All `update_*` methods conditionally based on dirty state
- **Notes**: Primary frame-update method called by game loop

### update_suit_energy
- **Signature**: `void update_suit_energy(short time_elapsed)`
- **Purpose**: Updates and redraws the player's shield/suit energy bar
- **Inputs**: `time_elapsed` ΓÇô for animation/visual effects
- **Outputs/Return**: (void)
- **Side effects**: Modifies player energy display; manages energy bar animation
- **Calls**: `draw_bar()`

### update_suit_oxygen
- **Signature**: `void update_suit_oxygen(short time_elapsed)`
- **Purpose**: Updates and redraws the player's oxygen meter (vacuum environments)
- **Inputs**: `time_elapsed` ΓÇô respects `DELAY_TICKS_BETWEEN_OXYGEN_REDRAW` throttle
- **Outputs/Return**: (void)
- **Side effects**: Throttled redraw of oxygen bar to avoid flicker
- **Calls**: `draw_bar()`

### update_weapon_panel
- **Signature**: `void update_weapon_panel(bool force_redraw)`
- **Purpose**: Updates weapon display panel showing current weapon and its state
- **Inputs**: `force_redraw` ΓÇô if true, bypass dirty flag and force redraw
- **Outputs/Return**: (void)
- **Side effects**: Refreshes weapon panel shape and weapon name text
- **Calls**: `draw_ammo_display_in_panel()`, drawing methods

### update_ammo_display
- **Signature**: `void update_ammo_display(bool force_redraw)`
- **Purpose**: Updates ammunition counter display for current weapon
- **Inputs**: `force_redraw` ΓÇô bypass dirty flag
- **Outputs/Return**: (void)
- **Side effects**: Redraws ammo count and magazine state
- **Calls**: `draw_ammo_display_in_panel()`

### draw_bar
- **Signature**: `void draw_bar(screen_rectangle *rectangle, short actual_height, shape_descriptor top_piece, shape_descriptor full_bar, shape_descriptor background_piece)`
- **Purpose**: Generic bar-drawing utility (energy/oxygen bars); renders segmented fill bar
- **Inputs**: `rectangle` ΓÇô screen location; `actual_height` ΓÇô current fill level; `top_piece`, `full_bar`, `background_piece` ΓÇô shape descriptors for layers
- **Outputs/Return**: (void)
- **Side effects**: Draws to screen via virtual methods
- **Calls**: `DrawShape()`, `FillRect()`
- **Notes**: Handles multi-segment energy bars (single, double, triple)

### Virtual Abstract Methods (platform-specific)
- `update_motion_sensor()`, `render_motion_sensor()` ΓÇô motion sensor update/render
- `draw_or_erase_unclipped_shape()`, `draw_entity_blip()` ΓÇô motion sensor blip drawing
- `DrawShape()`, `DrawShapeAtXY()`, `DrawText()`, `FillRect()`, `FrameRect()` ΓÇô basic drawing primitives
- `DrawTexture()` ΓÇô texture/shape drawing with parameters
- `SetClipPlane()`, `DisableClipPlane()` ΓÇô clipping region management
- `TextWidth()` ΓÇô text measurement for layout

## Control Flow Notes
**Initialization Phase**: Subclasses inherit `HUD_Class` and implement virtual drawing methods appropriate to their renderer (e.g., OpenGL, software rasterizer).

**Frame Update**: `update_everything()` is called once per game tick. It checks dirty flags (set by game state changes) and calls targeted update methods for modified HUD elements. Each update method queries player state and calls virtual draw methods to refresh the screen.

**Dirty-Flag Optimization**: `INVENTORY_DIRTY_BIT`, `INTERFACE_DIRTY_BIT`, and per-element flags in `interface_state_data` allow selective redrawΓÇöonly affected UI elements are refreshed each frame, reducing CPU cost.

**Virtual Dispatch**: All actual drawing (shapes, rectangles, text) is deferred to virtual methods so subclasses can implement platform-specific rendering (e.g., OpenGL texturing vs. software blitting).

## External Dependencies
- **cseries.h**: Core platform abstraction (fixed-point types, basic macros, SDL headers)
- **map.h**: `TICKS_PER_SECOND`, shape descriptors, world geometry
- **interface.h**: Shape/animation data structures, collection management
- **player.h**: Player data structures, weapon and item types
- **SoundManager.h**: Sound playback for button clicks and alerts
- **motion_sensor.h**: Motion sensor shape descriptors and initialization
- **items.h**: Item type enums (powerups, ammo, weapons)
- **weapons.h**: Weapon definitions and ammo display info
- **network_games.h**: Network compass shape descriptors
- **screen_drawing.h**: Screen coordinate and clipping utilities

**Defined Elsewhere**: `shape_descriptor`, `screen_rectangle`, `point2d` (imported from interface/screen headers); player/weapon/item state queried from global `current_player` and map data.
