# Source_Files/RenderOther/overhead_map.h

## File Purpose
Defines the interface for rendering a tactical/overhead map overlay during gameplay and saved game previews. Provides configuration constants, map state management, and rendering entry points with MML-based customization support.

## Core Responsibilities
- Define overhead map scaling limits and defaults (scale 1ΓÇô4, default 3)
- Manage rendering modes: saved game preview, checkpoint map, live game map
- Store overhead map rendering state (scale, origin point, screen position/dimensions)
- Expose rendering function to draw the scaled top-down map view
- Support MML (XML) configuration parsing and reset for customization

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `overhead_map_data` | struct | Holds complete map rendering state: mode, scale, world origin point, origin polygon reference, screen half/full dimensions, top-left position, and draw-all flag |
| Modes (anonymous enum) | enum | Three rendering contexts: `_rendering_saved_game_preview`, `_rendering_checkpoint_map`, `_rendering_game_map` |

## Global / File-Static State
None.

## Key Functions / Methods

### _render_overhead_map
- Signature: `void _render_overhead_map(struct overhead_map_data *data);`
- Purpose: Render the overhead map to screen using provided configuration
- Inputs: Pointer to `overhead_map_data` struct (scale, origin, dimensions, mode, flags)
- Outputs/Return: void (framebuffer modification)
- Side effects: Writes to framebuffer/display; accesses render system and world geometry
- Calls: Implementation not visible (defined elsewhere)
- Notes: Transforms world coordinates to screen space via scale and origin; behavior varies by mode (e.g., may hide enemies in preview mode)

### parse_mml_overhead_map
- Signature: `void parse_mml_overhead_map(const InfoTree& root);`
- Purpose: Parse MML/XML configuration tree to customize overhead map appearance
- Inputs: `InfoTree` reference (XML parse tree root)
- Outputs/Return: void (modifies global config state)
- Side effects: Updates global overhead map configuration
- Notes: Enables data/mod customization without recompilation

### reset_mml_overhead_map
- Signature: `void reset_mml_overhead_map();`
- Purpose: Reset overhead map configuration to compiled defaults
- Inputs: None
- Outputs/Return: void
- Notes: Clears MML customizations; called during config reload or map transitions

## Control Flow Notes
The overhead map is a UI overlay rendered during the HUD phase, typically after 3D world rendering. The `_render_overhead_map()` call is invoked once per frame when the map should be visible. MML configuration is parsed during engine initialization or map load, with state reset between maps or on configuration reload.

## External Dependencies
- `#include "world.h"` ΓÇö Provides `world_point2d`, `world_distance`, angle types, and world coordinate constants (WORLD_ONE, etc.)
- `InfoTree` class ΓÇö Forward-declared; XML/MML parse tree (defined elsewhere, likely in config parsing system)
- Implied: render system (framebuffer, drawing primitives), game world/polygon data
