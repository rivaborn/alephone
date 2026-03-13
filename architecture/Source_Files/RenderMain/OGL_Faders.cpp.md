# Source_Files/RenderMain/OGL_Faders.cpp

## File Purpose
Implements OpenGL rendering of fade effects (visual overlays) applied to the game view. Provides functionality to render various fade types including tinting, static/randomization, negation, dodge, burn, and soft-tinting effects to achieve visual feedback for game events.

## Core Responsibilities
- Determine if OpenGL fader rendering is enabled
- Manage access to the fader effect queue
- Apply color transformations (alpha pre-multiplication, color inversion)
- Render up to 6 distinct fade effect types using OpenGL blending modes
- Composite multiple simultaneous faders onto the screen

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Fader` | struct (from header) | Stores fade type and RGBA color |
| `GM_Random` | class (from header) | Pseudo-random number generator for static effect |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FlatStaticRandom` | `GM_Random` | static | RNG for randomized "flat static" fade effect |
| `UseFlatStatic` | `bool` | static | Selects between transparent vs. logic-op static rendering |
| `FlatStaticColor[4]` | `uint16[4]` | static | Pre-computed RGBA values for flat static effect |
| `FaderQueue[NUMBER_OF_FADER_QUEUE_ENTRIES]` | `OGL_Fader[]` | static | Queue of pending fade effects to render |

## Key Functions / Methods

### OGL_FaderActive
- **Signature:** `bool OGL_FaderActive()`
- **Purpose:** Check whether OpenGL fader rendering is currently enabled
- **Inputs:** None
- **Outputs/Return:** Boolean; true if OpenGL is active and fader flag is set
- **Side effects:** None
- **Calls:** `OGL_IsActive()`, `Get_OGL_ConfigureData()`, macro `TEST_FLAG()`
- **Notes:** Always returns false if OpenGL rendering itself is inactive

### GetOGL_FaderQueueEntry
- **Signature:** `OGL_Fader *GetOGL_FaderQueueEntry(int Index)`
- **Purpose:** Retrieve a fader from the queue by index with bounds validation
- **Inputs:** Index (0ΓÇô`NUMBER_OF_FADER_QUEUE_ENTRIES`ΓêÆ1)
- **Outputs/Return:** Pointer to the queued `OGL_Fader`
- **Side effects:** Asserts if index is out of bounds
- **Calls:** `assert()`

### MultAlpha (inline)
- **Signature:** `void MultAlpha(GLfloat *InColor, GLfloat *OutColor)`
- **Purpose:** Pre-multiply RGB by alpha channel for proper OpenGL blending
- **Inputs:** `InColor` (RGBA float array)
- **Outputs/Return:** `OutColor` (pre-multiplied RGBA)
- **Side effects:** Modifies output array
- **Calls:** None
- **Notes:** Performs `OutColor[c] = InColor[c] * InColor[3]` for RGB; alpha channel copied unchanged

### ComplementColor (inline)
- **Signature:** `void ComplementColor(GLfloat *InColor, GLfloat *OutColor)`
- **Purpose:** Invert RGB channels (compute 1 ΓêÆ channel) for dodge/negate effects
- **Inputs:** `InColor` (RGBA float array)
- **Outputs/Return:** `OutColor` (inverted RGB, unchanged alpha)
- **Side effects:** Modifies output array
- **Calls:** None

### OGL_DoFades
- **Signature:** `bool OGL_DoFades(float Left, float Top, float Right, float Bottom)`
- **Purpose:** Render all queued fade effects as full-screen or region quads using OpenGL
- **Inputs:** Screen-space rectangle coordinates (`Left`, `Top`, `Right`, `Bottom`)
- **Outputs/Return:** Boolean; true if any faders were rendered
- **Side effects:** 
  - Modifies OpenGL state: disables texture coords, sets blend modes, color, logic ops
  - Renders to framebuffer
  - Allocates local `Vertices[4][2]` and `BlendColor[4]` arrays
- **Calls:** 
  - `OGL_FaderActive()`, `Get_OGL_ConfigureData()`
  - GL functions: `glDisableClientState`, `glVertexPointer`, `glDisable`, `glEnable`, `glColor*`, `glDrawArrays`, `glBlendFunc`, `glLogicOp`
  - `MultAlpha()`, `ComplementColor()`
  - Random methods: `KISS()`, `LFIB4()` (for static effect)
- **Notes:** 
  - Iterates through entire `FaderQueue` regardless of `Type`
  - Implements 6 fader types with distinct blending/logic strategies:
    - `_tint_fader_type`: Simple color blend
    - `_randomize_fader_type`: XOR-based random flipping or flat color noise
    - `_negate_fader_type`: Approximates screen inversion
    - `_dodge_fader_type`: Two-pass dodge blend (brighten)
    - `_burn_fader_type`: Two-pass burn blend (darken) with color complement
    - `_soft_tint_fader_type`: Multiplicative tint (simulates colored light)
  - Restores default blend function after each effect
  - Note: ATI Rage 128 AppleGL limitation prevents full negation support

## Control Flow Notes
Fits into the **frame/render phase** after main scene composition. During rendering:
1. Engine calls `OGL_FaderActive()` to check if faders are enabled
2. Fader queue is populated externally (by other game systems)
3. `OGL_DoFades()` is invoked with screen bounds to composite all pending effects
4. Multiple faders stack additively due to sequential OpenGL blending

## External Dependencies
- **OpenGL:** `GL*` functions for state and rendering (blend, color, vertex arrays, logic ops)
- **fades.h:** Fader type constants (`NONE`, `_tint_fader_type`, etc.); `NUMBER_OF_FADER_QUEUE_ENTRIES`
- **Random.h:** `GM_Random` class with `KISS()` and `LFIB4()` methods
- **OGL_Render.h:** `OGL_IsActive()`
- **OGL_Setup.h:** `Get_OGL_ConfigureData()`, flags (`OGL_Flag_Fader`, `OGL_Flag_FlatStatic`)
- **OGL_Headers.h:** OpenGL header inclusion wrapper
