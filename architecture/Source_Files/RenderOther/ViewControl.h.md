# Source_Files/RenderOther/ViewControl.h

## File Purpose
View control subsystem for the Aleph One game engine. Manages camera/viewport parameters including field-of-view, landscape rendering, teleport effects, and on-screen display font. All parameters are configurable via XML.

## Core Responsibilities
- Field-of-view (FOV) management: accessors and adjustments for normal, extravision, and tunnel-vision modes
- Landscape/sky rendering configuration per environment/texture
- Teleport visual effects control: fold effects, static distortion, interlevel transition effects
- On-screen display (HUD) font retrieval
- Overhead map state query
- XML configuration parsing and reset for view and landscape settings

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `LandscapeOptions` | struct | Encodes landscape/sky texture scaling, aspect-ratio, azimuth offset, repeat/clamp modes, and spheremap flag. Used to customize environment rendering per shape/texture. |

## Global / File-Static State
None.

## Key Functions / Methods

### View_MapActive
- Signature: `bool View_MapActive()`
- Purpose: Query whether the overhead map view can be displayed.
- Inputs: None
- Outputs/Return: Boolean; true if map is available.
- Side effects: None

### View_FOV_Normal, View_FOV_ExtraVision, View_FOV_TunnelVision
- Signature: `float View_FOV_Normal()`, `float View_FOV_ExtraVision()`, `float View_FOV_TunnelVision()`
- Purpose: Accessors for fixed field-of-view angles in three vision modes.
- Inputs: None
- Outputs/Return: FOV angle in float (degrees or radians, not specified).
- Side effects: None

### View_AdjustFOV
- Signature: `bool View_AdjustFOV(float& FOV, float FOV_Target)`
- Purpose: Smoothly adjust FOV toward a target value (likely for smooth transitions between vision modes).
- Inputs: Current FOV (by reference), target FOV
- Outputs/Return: True if FOV was modified; updated FOV written to first parameter
- Side effects: Modifies FOV in place

### View_FOV_FixHorizontalNotVertical
- Signature: `bool View_FOV_FixHorizontalNotVertical()`
- Purpose: Determine whether to maintain horizontal (true) or vertical (false, default) FOV angle during aspect-ratio changes.
- Inputs: None
- Outputs/Return: Boolean
- Side effects: None

### View_DoFoldEffect, View_DoStaticEffect
- Signature: `bool View_DoFoldEffect()`, `bool View_DoStaticEffect()`
- Purpose: Query whether to apply fold-in/fold-out or static distortion when teleporting.
- Inputs: None
- Outputs/Return: Boolean
- Side effects: None

### View_DoInterlevelTeleportInEffects, View_DoInterlevelTeleportOutEffects
- Signature: `bool View_DoInterlevelTeleportInEffects()`, `bool View_DoInterlevelTeleportOutEffects()`
- Purpose: Query whether to apply visual effects when entering or exiting a level via teleport.
- Inputs: None
- Outputs/Return: Boolean
- Side effects: None

### GetOnScreenFont
- Signature: `FontSpecifier& GetOnScreenFont()`
- Purpose: Retrieve the font used for on-screen display (HUD text, messages).
- Inputs: None
- Outputs/Return: Reference to FontSpecifier object
- Side effects: None

### View_GetLandscapeOptions
- Signature: `LandscapeOptions *View_GetLandscapeOptions(shape_descriptor Desc)`
- Purpose: Look up landscape/sky rendering options for a given shape descriptor (texture).
- Inputs: Shape descriptor (collection + shape index)
- Outputs/Return: Pointer to LandscapeOptions; null if shape has no custom landscape settings
- Side effects: None (query only)

### parse_mml_view, reset_mml_view, parse_mml_landscapes, reset_mml_landscapes
- Purpose: XML parsing entry points for view and landscape configuration. Parse functions read XML and update state; reset functions restore defaults.
- Inputs: `parse_mml_*` take const `InfoTree&` root node
- Outputs/Return: None (void)
- Side effects: Modify global/persistent view/landscape configuration state
- Notes: Trivial helpers for XML integration; full definitions elsewhere

## Control Flow Notes
**Initialization phase**: XML parsers called to load view and landscape settings.

**Per-frame/render**: FOV adjustment queried and applied; landscape options looked up by shape descriptor when rendering sky/environment. Teleport effect flags checked during level transitions or player teleport events.

**Shutdown**: Reset functions may be called to restore defaults.

## External Dependencies
- `#include "world.h"` ΓÇö provides `angle` type
- `#include "FontHandler.h"` ΓÇö provides `FontSpecifier` class
- `#include "shape_descriptors.h"` ΓÇö provides `shape_descriptor` type
- `InfoTree` class (forward-declared, defined elsewhere) for XML parsing
- Implementation must define all `View_*` functions and landscape lookup table
