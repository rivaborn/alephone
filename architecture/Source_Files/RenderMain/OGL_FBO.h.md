# Source_Files/RenderMain/OGL_FBO.h

## File Purpose
Defines classes for managing OpenGL Framebuffer Objects (FBOs) used for off-screen rendering and texture rendering targets. Provides utilities for activating, deactivating, and compositing between FBOs, supporting both single FBO and double-buffered FBO swapping patterns.

## Core Responsibilities
- Manage individual FBO lifecycle (allocation, activation, deactivation, cleanup)
- Track active FBOs via a static chain for nested/stacked activation
- Provide drawing and composition operations (blit, blend, filter, copy)
- Implement FBOSwapper for ping-pong rendering (e.g., post-processing pipelines)
- Handle sRGB color space management for FBOs
- Support clearing and state management for FBO render targets

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FBO` | class | Encapsulates a single OpenGL framebuffer object with depth buffer and texture attachment |
| `FBOSwapper` | class | Manages two FBOs for alternating render targets (ping-pong pattern) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FBO::active_chain` | `std::vector<FBO*>` | static | Stack of currently active FBOs for nested/stacked management |

## Key Functions / Methods

### FBO::FBO (constructor)
- Signature: `FBO(GLuint w, GLuint h, bool srgb = false)`
- Purpose: Allocate and initialize a framebuffer object with attached depth buffer and color texture
- Inputs: Width, height, sRGB flag (default false)
- Outputs/Return: None (constructor)
- Side effects: Allocates OpenGL resources (FBO, depth renderbuffer, texture); manages GPU memory
- Calls: (defined elsewhere, likely GL creation calls)
- Notes: sRGB determines color space interpretation for the FBO

### FBO::activate
- Signature: `void activate(bool clear = false, GLuint fboTarget = GL_FRAMEBUFFER_EXT)`
- Purpose: Make this FBO the current render target and push to active chain
- Inputs: Clear flag (default false), framebuffer target (default GL_FRAMEBUFFER_EXT)
- Outputs/Return: None
- Side effects: Updates OpenGL render target; modifies `active_chain`; may clear framebuffer
- Calls: (defined elsewhere, OpenGL bind/clear calls)
- Notes: Supports nested activation via chain stack

### FBO::deactivate
- Signature: `void deactivate()`
- Purpose: Restore previous render target by popping from active chain
- Inputs: None
- Outputs/Return: None
- Side effects: Restores OpenGL render target from `active_chain`
- Calls: (defined elsewhere)
- Notes: Assumes matching activate/deactivate pairs

### FBO::draw, prepare_drawing_mode, reset_drawing_mode, draw_full
- Purpose: Blit FBO contents to current render target; manage GL state for blended drawing
- Inputs: `blend` flag for blended composition
- Outputs/Return: None
- Side effects: Issues draw calls; may modify blend state
- Calls: (defined elsewhere)
- Notes: `prepare_drawing_mode` / `reset_drawing_mode` bracket state changes; `draw_full` combines both

### FBO::active_fbo (static)
- Signature: `static FBO *active_fbo()`
- Purpose: Return currently active FBO from chain (top of stack)
- Inputs: None
- Outputs/Return: Pointer to topmost active FBO
- Side effects: None
- Calls: (accesses `active_chain`)
- Notes: Used for querying current render target

### FBOSwapper methods
- `activate()` / `deactivate()`: Enable/disable swapper state
- `swap()`: Toggle `draw_to_first` to alternate render targets
- `draw()`, `filter()`: Blit current contents to caller's target
- `copy()`, `blend()`: Composite current FBO to another with optional sRGB conversion
- `blend_multisample()`: Specialized blend for multisample FBOs
- `current_contents()`: Return the FBO holding rendered data (opposite of render target)

## Control Flow Notes
FBOs are typically activated before rendering a frame or pass, then deactivated to restore the previous target. FBOSwapper is used in multi-pass rendering: render to first FBO, swap, render effects to second FBO, swap again, etc. The `active_chain` enables safe nesting of FBO contexts (similar to matrix stack in fixed-function GL).

## External Dependencies
- **`OGL_Headers.h`**: Provides OpenGL type definitions and extension headers (GLuint, GL_FRAMEBUFFER_EXT, GLEW/SDL_opengl)
- **`cseries.h`**: Aleph One utility headers (types, macros)
- **`<vector>`** (STL): Dynamic array for `active_chain`
- **Defined elsewhere**: Actual method implementations, OpenGL binding/state calls, texture setup
