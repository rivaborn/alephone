# Source_Files/RenderMain/OGL_Faders.h

## File Purpose
Header file for OpenGL screen fade effect rendering in the Aleph One game engine. Defines the interface for managing and rendering fader effects (color overlays with transparency) over the game viewport.

## Core Responsibilities
- Declare whether OpenGL-based faders are currently active
- Define fader queue categories (Liquid, Other) for organizing different fade types
- Define the `OGL_Fader` data structure for fader properties
- Provide queue access to retrieve fader entries by index
- Declare the main fader rendering function that applies fade effects to a rectangular region

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Fader` | struct | Encapsulates a single fader effect: fade type, RGBA color values, and default constructor |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_FaderActive
- Signature: `bool OGL_FaderActive()`
- Purpose: Query whether any OpenGL faders are currently queued and active
- Inputs: None
- Outputs/Return: Boolean indicating active fader state
- Side effects: None inferable from signature
- Calls: Not visible in this file (defined elsewhere)
- Notes: Used to optimize frame rendering by skipping fader logic when inactive

### GetOGL_FaderQueueEntry
- Signature: `OGL_Fader *GetOGL_FaderQueueEntry(int Index)`
- Purpose: Access a specific fader from the internal fader queue by index
- Inputs: `Index` ΓÇô queue entry identifier (corresponds to `FaderQueue_Liquid` or `FaderQueue_Other`)
- Outputs/Return: Pointer to `OGL_Fader` struct
- Side effects: None inferable
- Calls: Not visible in this file
- Notes: Index should match enum values (`FaderQueue_Liquid` = 0, `FaderQueue_Other` = 1)

### OGL_DoFades
- Signature: `bool OGL_DoFades(float Left, float Top, float Right, float Bottom)`
- Purpose: Render all active faders as overlay effects within a rectangular viewport region
- Inputs: `Left, Top, Right, Bottom` ΓÇô bounding box coordinates for fade region
- Outputs/Return: Boolean indicating whether any faders were actually rendered
- Side effects: OpenGL state modification (rendering)
- Calls: Not visible in this file (defined elsewhere)
- Notes: Intended as part of frame render pipeline; returns false if no faders were active

## Control Flow Notes
This header is part of the OpenGL render pipeline. `OGL_DoFades()` is called during frame rendering (likely after scene geometry) to composite fade overlays. The fader queue is populated elsewhere; this file only defines the rendering interface.

## External Dependencies
- `#include "cstypes.h"` ΓÇô provides standard types and the `NONE` constant used in `OGL_Fader` default constructor
- Uses standard C types: `short`, `float`, `bool`
