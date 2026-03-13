# Source_Files/RenderOther/game_window.h

## File Purpose
Header declaring the game window initialization, HUD rendering, and interface state management functions. Provides the main entry points for updating and drawing the on-screen HUD each frame, along with dirty-flag functions to invalidate UI elements.

## Core Responsibilities
- Initialize the game window at startup
- Render the HUD and interface each frame (draw and update passes)
- Manage UI element states (ammo, shields, oxygen, weapons, inventory)
- Handle inventory scrolling and manipulation
- Invalidate UI elements via dirty flags to trigger efficient redrawing
- Parse and apply XML-based interface definitions (MML format)
- Ensure HUD framebuffer is allocated and valid

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Rect` | struct | Forward declared; represents screen rectangle regions for HUD drawing |
| `InfoTree` | class | Forward declared; represents parsed XML interface configuration |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_game_window
- Signature: `void initialize_game_window(void)`
- Purpose: Perform one-time initialization of the game window and HUD subsystem at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes global HUD state, allocates buffers
- Calls: Not inferable from this file
- Notes: Called once during engine initialization phase

### draw_interface
- Signature: `void draw_interface(void)`
- Purpose: Render the HUD elements to the screen
- Inputs: None
- Outputs/Return: None
- Side effects: Draws to backbuffer
- Calls: Not inferable from this file
- Notes: Called each frame in the render pass

### update_interface
- Signature: `void update_interface(short time_elapsed)`
- Purpose: Update HUD logic and animations based on elapsed time
- Inputs: `time_elapsed` ΓÇö milliseconds since last frame
- Outputs/Return: None
- Side effects: Updates HUD animation state
- Calls: Not inferable from this file
- Notes: Called each frame in the update pass

### OGL_DrawHUD
- Signature: `void OGL_DrawHUD(Rect &dest, short time_elapsed)`
- Purpose: Draw the HUD using OpenGL into a specified destination rectangle
- Inputs: `dest` ΓÇö target screen region; `time_elapsed` ΓÇö frame delta
- Outputs/Return: None
- Side effects: Renders to OpenGL framebuffer
- Calls: Not inferable from this file
- Notes: Likely an alternate or specialized HUD render path

### parse_mml_interface
- Signature: `void parse_mml_interface(const InfoTree& root)`
- Purpose: Load and apply interface customizations from XML (MML format)
- Inputs: `root` ΓÇö parsed XML tree containing interface definitions
- Outputs/Return: None
- Side effects: Reconfigures HUD appearance and layout
- Calls: Not inferable from this file
- Notes: Allows data-driven interface customization

### Dirty-flag functions
- `mark_ammo_display_as_dirty()`, `mark_shield_display_as_dirty()`, `mark_oxygen_display_as_dirty()`, `mark_weapon_display_as_dirty()`, `mark_player_inventory_screen_as_dirty()`, `mark_player_inventory_as_dirty()`, `mark_player_network_stats_as_dirty()`
  - Purpose: Flag HUD elements for redrawing on next frame
  - Inputs: Player index and/or item index where applicable
  - Side effects: Set dirty flags to skip redundant rendering
  - Notes: Efficient UI update pattern; called from gameplay code when state changes

## Control Flow Notes
Fits into the main game loop: `initialize_game_window()` runs at engine startup; `update_interface()` and `draw_interface()` are called each frame during the update and render phases respectively. Dirty-flag functions are called by gameplay subsystems when player state changes (health, ammo, inventory).

## External Dependencies
- **`Rect`** ΓÇö forward declared; defined elsewhere (likely in graphics/geometry module)
- **`InfoTree`** ΓÇö forward declared; defined elsewhere (likely in XML parsing module)
- Indirectly uses OpenGL (see `OGL_DrawHUD` naming)
- Copyright indicates Bungie Studios / Aleph One engine
