# Source_Files/RenderMain/OGL_Headers.h

## File Purpose
Cross-platform compatibility header that conditionally includes OpenGL headers based on the target platform. Provides a single include point for all Aleph One rendering code to access OpenGL without managing platform-specific header differences.

## Core Responsibilities
- Guard against multiple inclusion and provide consistent OpenGL access points
- Abstract platform-specific OpenGL header paths (Windows/Unix/macOS)
- Enable static GLEW linking on Windows (`GLEW_STATIC`)
- Define `GL_GLEXT_PROTOTYPES` for Unix/Linux platforms
- Check for OpenGL support via `HAVE_OPENGL` feature detection

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This is a pure header-only include file with no executable code. It participates in the **initialization phase** by being included early in translation units that need OpenGL functionality. Its preprocessor directives ensure that whichever OpenGL headers are included are done so correctly for the target platform, avoiding linker errors and symbol conflicts.

## External Dependencies
- **Windows:** `<GL/glew.h>` (static linking via `GLEW_STATIC` macro)
- **Unix/Linux:** `<SDL2/SDL_opengl.h>` with `GL_GLEXT_PROTOTYPES` enabled
- **macOS:** `<OpenGL/glu.h>` for GLU utilities
- **Linux fallback:** `<GL/glu.h>` (when not on macOS)
- **Project config:** `"config.h"` (feature detection: `HAVE_OPENGL`, `__WIN32__`, `__APPLE__`, `__MACH__`)
