# Source_Files/RenderOther/OGL_LoadScreen.h

## File Purpose
Defines a singleton class that manages OpenGL-based load screens for the game engine. Handles displaying background images, progress indicators, and visual feedback during asset loading phases.

## Core Responsibilities
- Singleton instance management and lifecycle (Start/Stop)
- Loading and configuring load screen images with flexible positioning and scaling
- Rendering progress indicators with customizable colors
- Managing OpenGL texture resources and blitting parameters
- Querying active state and providing color palette access for rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OGL_LoadScreen | class | Main singleton managing load screen rendering |
| ImageDescriptor | class (external) | Holds loaded image pixel data and metadata |
| OGL_Blitter | class (external) | Performs OpenGL texture rendering to framebuffer |
| SDL_Rect | struct (external) | SDL rectangle for destination rendering area |

## Global / File-Static State
None (static instance managed via `instance()` method; implementation file not provided).

## Key Functions / Methods

### Start
- **Signature:** `bool Start()`
- **Purpose:** Initialize and begin displaying the load screen
- **Outputs/Return:** Boolean success status
- **Side effects:** Allocates OpenGL texture resources, begins rendering loop

### Stop
- **Signature:** `void Stop()`
- **Purpose:** Cease load screen display and release resources
- **Side effects:** Deallocates textures, halts rendering

### Progress
- **Signature:** `void Progress(const int percent)`
- **Purpose:** Update progress bar percentage for visual feedback
- **Inputs:** `percent` ΓÇö progress value (presumably 0ΓÇô100)
- **Side effects:** Updates internal `percent` state; may trigger re-render

### Set (Configure Image)
- **Signature:** `void Set(std::string Path, bool Stretch, bool Scale)` / `void Set(std::string Path, bool Stretch, bool Scale, short X, short Y, short W, short H)`
- **Purpose:** Load and configure background image with positioning/scaling options
- **Inputs:** `Path` (file path), `Stretch` (expand to fill), `Scale` (apply scaling), `X/Y/W/H` (position/dimensions)
- **Side effects:** Loads image from disk, updates blitter and transform parameters

### Clear
- **Signature:** `void Clear()`
- **Purpose:** Reset load screen to default/inactive state
- **Side effects:** Clears image data, resets flags

### Use
- **Signature:** `bool Use()`
- **Purpose:** Query whether load screen is currently active
- **Outputs/Return:** True if in use

### Colors
- **Signature:** `rgb_color *Colors()`
- **Purpose:** Provide access to color palette (e.g., for progress bar rendering)
- **Outputs/Return:** Pointer to 2-element color array

## Control Flow Notes
Typical lifecycle: `Start()` ΓåÆ repeated `Progress()` calls during asset load ΓåÆ `Stop()` on completion. The singleton pattern ensures a single load screen instance persists across frame updates. Configuration (`Set()`) typically precedes `Start()`. Private constructor enforces singleton instantiation via `instance()`.

## External Dependencies
- **cseries.h** ΓÇö Core type definitions, SDL includes
- **OGL_Headers.h** ΓÇö OpenGL context and function declarations
- **OGL_Blitter.h** ΓÇö Image blitting to OpenGL framebuffer
- **ImageLoader.h** ΓÇö `ImageDescriptor` class for image data storage
- **SDL2** ΓÇö Rect structures and surface abstractions
