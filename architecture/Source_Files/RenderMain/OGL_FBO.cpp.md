# Source_Files/RenderMain/OGL_FBO.cpp

## File Purpose
Implements OpenGL framebuffer object (FBO) management for off-screen rendering. Provides a single-FBO wrapper (`FBO`) with activation/deactivation stacking, and a dual-FBO ping-ponger (`FBOSwapper`) for post-processing effects like filtering and blending.

## Core Responsibilities
- Create and manage OpenGL framebuffer objects with color textures and depth buffers
- Maintain a stack of active FBOs to support nested rendering contexts
- Implement 2D drawing modes for rendering FBO contents to the screen
- Manage sRGB color-space rendering state during FBO operations
- Provide ping-pong double-buffering between two FBOs for iterative post-processing
- Support filtering, copying, and blending operations between FBOs
- Handle multisample blending with multiple texture units

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FBO | class | Wraps a single OpenGL framebuffer with color texture and depth renderbuffer |
| FBOSwapper | class | Manages two FBOs and swaps between them for ping-pong rendering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| FBO::active_chain | static std::vector<FBO*> | static | Stack of currently active FBOs; enables nested activation contexts |

## Key Functions / Methods

### FBO::FBO (constructor)
- **Signature:** `FBO(GLuint w, GLuint h, bool srgb = false)`
- **Purpose:** Initialize a framebuffer object with specified dimensions and optional sRGB color space.
- **Inputs:** Width `w`, height `h`, sRGB flag.
- **Outputs/Return:** N/A (constructor)
- **Side effects:** Allocates GPU resources: `glGenFramebuffersEXT`, `glGenRenderbuffersEXT`, `glGenTextures`; binds and configures them immediately. Asserts framebuffer completeness.
- **Calls:** `glGenFramebuffersEXT`, `glBindFramebufferEXT`, `glGenRenderbuffersEXT`, `glRenderbufferStorageEXT`, `glFramebufferRenderbufferEXT`, `glGenTextures`, `glBindTexture`, `glTexImage2D`, `glTexParameteri`, `glFramebufferTexture2DEXT`, `glCheckFramebufferStatusEXT`
- **Notes:** Creates `GL_TEXTURE_RECTANGLE_ARB` texture (non-power-of-two friendly). Uses HUD texture filter settings. Asserts completeness; crashes if FBO setup fails.

### FBO::activate
- **Signature:** `void activate(bool clear = false, GLuint fboTarget = GL_FRAMEBUFFER_EXT)`
- **Purpose:** Make this FBO the active render target; push onto activation stack.
- **Inputs:** `clear` flag (clear color/depth on activate); `fboTarget` (GL_FRAMEBUFFER_EXT or GL_DRAW_FRAMEBUFFER_EXT).
- **Outputs/Return:** None
- **Side effects:** Modifies `active_chain` stack, binds FBO, saves viewport, sets viewport to FBO dimensions, enables/disables sRGB based on `_srgb`, optionally clears buffers.
- **Calls:** `glBindFramebufferEXT`, `glPushAttrib`, `glViewport`, `glEnable`/`glDisable` (GL_FRAMEBUFFER_SRGB_EXT), `glClear`
- **Notes:** Skips re-activation if this FBO is already on top of stack. Does not clear if already active (guard prevents double-clear on stack reuse).

### FBO::deactivate
- **Signature:** `void deactivate()`
- **Purpose:** Pop this FBO from the active stack and restore previous render target.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Pops from `active_chain`, restores viewport via `glPopAttrib`, rebinds previous FBO (or default 0), restores sRGB state from previous FBO or global `Using_sRGB` flag.
- **Calls:** `glPopAttrib`, `glBindFramebufferEXT`, `glEnable`/`glDisable` (GL_FRAMEBUFFER_SRGB_EXT)
- **Notes:** Only works if this FBO is on top of stack; no-op otherwise. Restores both framebuffer and sRGB state to support proper nesting.

### FBO::draw
- **Signature:** `void draw()`
- **Purpose:** Render the FBO's texture to the currently active framebuffer.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Binds FBO texture, enables rectangle texture mode, calls `OGL_RenderTexturedRect`, disables texture.
- **Calls:** `glBindTexture`, `glEnable`, `OGL_RenderTexturedRect`, `glDisable`
- **Notes:** Does not set up projection/modelview matrices; caller must prepare those. Expects a prior `activate()` on target FBO.

### FBO::prepare_drawing_mode
- **Signature:** `void prepare_drawing_mode(bool blend = false)`
- **Purpose:** Set up projection and modelview matrices for 2D rendering of FBO contents.
- **Inputs:** `blend` flag (whether to keep blending enabled).
- **Outputs/Return:** None
- **Side effects:** Pushes and resets projection and modelview matrices to identity, disables depth test, optionally disables blending, sets orthographic projection matching FBO dimensions, sets color to white opaque.
- **Calls:** `glMatrixMode`, `glPushMatrix`, `glLoadIdentity`, `glDisable`, `glOrtho`, `glColor4f`
- **Notes:** Caller must call `reset_drawing_mode()` to restore. Sets up coordinate space (0,0) = top-left, y increases downward.

### FBO::reset_drawing_mode
- **Signature:** `void reset_drawing_mode()`
- **Purpose:** Restore projection/modelview matrices and depth/blend state after drawing.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Enables blend and depth test, pops projection and modelview matrices.
- **Calls:** `glEnable`, `glMatrixMode`, `glPopMatrix`
- **Notes:** Inverse of `prepare_drawing_mode()`. Always re-enables depth and blend, regardless of what `prepare_drawing_mode` did.

### FBO::draw_full
- **Signature:** `void draw_full(bool blend = false)`
- **Purpose:** Convenience wrapper: prepare mode, draw, reset mode.
- **Inputs:** `blend` flag.
- **Outputs/Return:** None
- **Side effects:** Calls `prepare_drawing_mode`, `draw`, `reset_drawing_mode` in sequence.
- **Calls:** `prepare_drawing_mode`, `draw`, `reset_drawing_mode`
- **Notes:** Simplifies drawing without manual mode management.

### FBO::~FBO (destructor)
- **Signature:** `~FBO()`
- **Purpose:** Clean up GPU resources.
- **Inputs:** N/A
- **Outputs/Return:** N/A
- **Side effects:** Deletes framebuffer and renderbuffer via OpenGL calls; texture deletion is implicit or handled elsewhere.
- **Calls:** `glDeleteFramebuffersEXT`, `glDeleteRenderbuffersEXT`
- **Notes:** Does not explicitly delete `texID` texture; may leak if not deleted separately. Does not pop from `active_chain` if still active (potential dangling pointer).

### FBOSwapper::activate / deactivate / swap
- **Signature:** `void activate()`, `void deactivate()`, `void swap()`
- **Purpose:** Manage activation state and ping-pong between two FBOs.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** `activate` calls `first.activate()` or `second.activate()` depending on `draw_to_first` flag, sets `active = true`, clears `clear_on_activate`. `swap` calls `deactivate`, flips `draw_to_first`, sets `clear_on_activate = true`.
- **Calls:** Delegates to FBO methods.
- **Notes:** Prevents redundant activations. `clear_on_activate` is set on swap and consumed on next `activate()`.

### FBOSwapper::filter
- **Signature:** `void filter(bool blend = false)`
- **Purpose:** Render `current_contents()` into the inactive FBO (post-processing filter).
- **Inputs:** `blend` flag.
- **Outputs/Return:** None
- **Side effects:** Activates inactive FBO, draws current contents, swaps (deactivates old, makes new active).
- **Calls:** `activate`, `draw`, `swap`
- **Notes:** Implements a single filter pass; can be called repeatedly for iterative effects.

### FBOSwapper::copy / blend
- **Signature:** `void copy(FBO& other, bool srgb)`, `void blend(FBO& other, bool srgb)`
- **Purpose:** Copy or blend another FBO into the swapper's inactive FBO.
- **Inputs:** Reference to source FBO; sRGB flag.
- **Outputs/Return:** None
- **Side effects:** `copy` activates, draws source, swaps. `blend` activates, sets sRGB, draws with blending enabled, deactivates (no swap).
- **Calls:** `activate`, `draw_full`, `swap`, `glEnable`/`glDisable`
- **Notes:** `blend` does not swap, so it blends into the currently active FBO. `copy` swaps, changing which FBO is "current."

### FBOSwapper::blend_multisample
- **Signature:** `void blend_multisample(FBO& other)`
- **Purpose:** Blend another FBO using multiple texture units for antialiasing or filtering.
- **Inputs:** Reference to source FBO.
- **Outputs/Return:** None
- **Side effects:** Swaps, activates, binds source texture to unit 1, configures texture coordinates from source dimensions, calls `draw(true)`, tears down multitexture state, deactivates.
- **Calls:** `swap`, `activate`, `glActiveTextureARB`, `glBindTexture`, `glEnable`/`glDisable`, `glClientActiveTextureARB`, `glEnableClientState`, `glTexCoordPointer`, `glDisableClientState`, `draw`, `deactivate`
- **Notes:** Sets up unit 1 with source FBO texture and hardcoded rect coordinates matching FBO dimensions. Restores unit 0 as active after. Fixed coordinates may not suit dynamic resizing.

## Control Flow Notes
**Initialization/Activation:** FBOs are created on demand (client code), then `activate()`/`deactivate()` to make them render targets. Stack-based nesting allows rendering to one FBO while another is active.

**Frame Update:** `FBOSwapper` is typically used for post-processing: render to one FBO, filter/blend into the other, display results. Calling `filter()` iteratively applies cascading effects.

**Rendering:** `draw()` and `draw_full()` output FBO contents, requiring caller to manage matrix state. `prepare_drawing_mode()` configures 2D orthographic space.

**Shutdown:** Destructors clean GPU resources on scope exit.

## External Dependencies
- **OGL_Setup.h:** Global flags (`Using_sRGB`, `Bloom_sRGB`), sRGB extension macros, texture type info (`TxtrTypeInfoList`).
- **OGL_Render.h:** `OGL_RenderTexturedRect()` function for texture drawing.
- **OGL_Textures.h:** Texture info structures (`TxtrTypeInfoData`, `OGL_Txtr_HUD`).
- **OpenGL EXT functions:** `glGenFramebuffersEXT`, `glBindFramebufferEXT`, `glCheckFramebufferStatusEXT`, `glGenRenderbuffersEXT`, etc. (legacy ARB/EXT extensions).
