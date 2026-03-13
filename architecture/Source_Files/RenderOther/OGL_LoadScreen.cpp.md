# Source_Files/RenderOther/OGL_LoadScreen.cpp

## File Purpose
Implements OpenGL-based load screen rendering for the Aleph One game engine. Displays splash/loading images with optional progress bars during asset loading or level transitions. Manages screen initialization, image display, progress updates, and cleanup via a singleton pattern.

## Core Responsibilities
- Manage singleton instance of the load screen system
- Load and validate image files for display
- Configure image scaling and centering based on aspect ratio and screen dimensions
- Render progress bars with configurable position, size, and colors
- Coordinate OpenGL buffer operations (clear, swap) for proper frame display
- Calculate transformation matrices for image and progress bar positioning
- Handle configuration state (paths, scaling modes, progress settings)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_LoadScreen` | class | Singleton managing load screen lifecycle and rendering |
| `ImageDescriptor` | struct (from ImageLoader.h) | Encapsulates loaded image data and metadata |
| `OGL_Blitter` | class (from OGL_Blitter.h) | Handles GPU-accelerated image rendering |
| `SDL_Rect` | struct | Defines destination rectangle for rendered image (x, y, w, h) |
| `rgb_color` | struct | Color values for progress bar background/foreground |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `instance_` | `OGL_LoadScreen*` | static in `instance()` method | Singleton instance holder |

## Key Functions / Methods

### instance()
- **Signature:** `static OGL_LoadScreen *instance()`
- **Purpose:** Returns the singleton instance, creating it on first call
- **Inputs:** None
- **Outputs/Return:** Pointer to static `OGL_LoadScreen` instance
- **Side effects:** Allocates singleton on heap on first invocation
- **Calls:** Constructor `OGL_LoadScreen()`
- **Notes:** Simple lazy-initialization singleton; never freed (intentional for process-lifetime object)

### Start()
- **Signature:** `bool Start()`
- **Purpose:** Initializes and displays the load screen image with calculated transformations
- **Inputs:** None (uses member variables set by `Set()`)
- **Outputs/Return:** `true` if successful, `false` if image load fails or path is empty
- **Side effects:** Loads image file, initializes blitter, clears OpenGL screen, calls `Progress(0)`, modifies `m_dst`, `x_offset`, `y_offset`, `x_scale`, `y_scale`, sets `use = true`
- **Calls:** `FileSpecifier::Exists()`, `FileSpecifier::SetNameWithPath()`, `ImageDescriptor::LoadFromFile()`, `OGL_Blitter::Load()`, `alephone::Screen::instance()->bound_screen()`, `OGL_ClearScreen()`, `Progress()`
- **Notes:** Calculates destination rect based on `scale` and `stretch` flags; applies aspect-ratio-preserving scaling if needed; centers image on 640├ù480 screen

### Stop()
- **Signature:** `void Stop()`
- **Purpose:** Cleans up the load screen and resets OpenGL state
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears screen twice, swaps buffers, calls `Clear()`
- **Calls:** `OGL_ClearScreen()` (2├ù), `OGL_SwapBuffers()`, `Clear()`
- **Notes:** Double-clear appears intentional for buffer coherence

### Progress(int progress)
- **Signature:** `void Progress(const int progress)`
- **Purpose:** Updates and renders the load screen with an optional progress bar
- **Inputs:** `progress` (expected range 0ΓÇô100, percent complete)
- **Outputs/Return:** None
- **Side effects:** Clears screen, renders image via `OGL_Blitter::Draw()`, optionally renders progress bar, swaps buffers; modifies OpenGL matrix state
- **Calls:** `OGL_ClearScreen()`, `OGL_Blitter::BoundScreen()`, `OGL_Blitter::Draw()`, OpenGL matrix and color functions, `OGL_RenderRect()`, `OGL_SwapBuffers()`
- **Notes:** Only draws progress bar if `useProgress == true`; uses `glMatrixMode(GL_MODELVIEW)` and scaling to transform progress bar coordinates from image space to screen space; vertical vs. horizontal progress bar determined by aspect ratio (`height > width`)

### Set(std::string Path, bool Stretch, bool Scale) [overload 1]
- **Signature:** `void Set(std::string Path, bool Stretch, bool Scale)`
- **Purpose:** Configures load screen without progress bar display
- **Inputs:** `Path` (image file path), `Stretch` (stretch to fill screen), `Scale` (enable scaling)
- **Outputs/Return:** None
- **Side effects:** Calls overloaded `Set()` with X=0, Y=0, W=0, H=0; sets `useProgress = false`
- **Calls:** `Set(Path, Stretch, Scale, 0, 0, 0, 0)`
- **Notes:** Convenience wrapper

### Set(std::string Path, bool Stretch, bool Scale, short X, short Y, short W, short H) [overload 2]
- **Signature:** `void Set(std::string Path, bool Stretch, bool Scale, short X, short Y, short W, short H)`
- **Purpose:** Configures load screen with progress bar parameters
- **Inputs:** `Path`, `Stretch`, `Scale`, and `(X, Y, W, H)` for progress bar rect in image coordinates
- **Outputs/Return:** None
- **Side effects:** Stores all parameters as members, clears image, sets `use = true`, `useProgress = true`, `percent = 0`
- **Calls:** `ImageDescriptor::Clear()`, `OGL_Blitter::Unload()`
- **Notes:** Ready state for `Start()` call; image clearing allows file reload

### Clear()
- **Signature:** `void Clear()`
- **Purpose:** Resets all state and cleans up resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears screen (3├ù total with stop sequence), swaps buffers, resets `use`, `useProgress`, `path`, `image`, `blitter`
- **Calls:** `OGL_ClearScreen()` (2├ù), `OGL_SwapBuffers()`, `ImageDescriptor::Clear()`, `OGL_Blitter::Unload()`
- **Notes:** Called by `Stop()` and as part of state reset

## Control Flow Notes
- **Initialization:** `Set()` configures parameters; `Start()` loads image and renders first frame (progress 0%)
- **Loading loop:** Caller repeatedly invokes `Progress(percent)` during asset loading (e.g., every frame or checkpoint)
- **Cleanup:** `Stop()` flushes buffers and calls `Clear()` to reset state
- **Lifecycle:** Fits into engine startup (level load, menu transitions) where blocking load times require visual feedback

## External Dependencies
- **OGL_Render.h:** `OGL_ClearScreen()`, `OGL_SwapBuffers()`, `OGL_RenderRect()`
- **OGL_Blitter.h:** `OGL_Blitter` class, `OGL_Blitter::BoundScreen()`
- **ImageLoader.h:** `ImageDescriptor`, `ImageLoader_Colors` enum constant
- **screen.h:** `alephone::Screen::instance()`, `Screen::bound_screen()`
- **OpenGL (via included headers):** `glMatrixMode()`, `glPushMatrix()`, `glTranslated()`, `glScaled()`, `glColor3us()`, `glPopMatrix()`
- **SDL2:** `SDL_Rect` type
- **Standard library:** `std::string` for file paths
