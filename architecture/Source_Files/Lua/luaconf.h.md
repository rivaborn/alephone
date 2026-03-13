# Source_Files/Lua/luaconf.h

## File Purpose

Configuration header for the Lua interpreter/library. Defines platform-specific feature detection, module paths, type sizes, API visibility macros, and numeric type formatting. All definitions are compile-time configuration with no runtime behaviorΓÇötuning is done by editing macros before compilation.

## Core Responsibilities

- **Platform detection and feature enablement** ΓÇö conditionally enable POSIX, Linux, macOS, or Windows-specific functionality
- **Module and library paths** ΓÇö define default search paths for Lua and C modules
- **API visibility and export** ΓÇö mark functions for DLL export, visibility ("hidden"), or static linkage
- **Numeric type definition** ΓÇö configure whether Lua uses `double`, `float`, or custom numeric types; define integer types
- **Number formatting and conversion** ΓÇö specify scanf/printf format strings and conversion functions for numbers
- **Stack and buffer limits** ΓÇö constrain Lua stack depth and auxiliary library buffer sizes
- **Math operation macros** ΓÇö define fast paths for arithmetic (+, ΓêÆ, ├ù, ├╖, mod, pow) conditional on compilation unit
- **Backward compatibility** ΓÇö provide deprecated function replacements and old API bindings via `LUA_COMPAT_*` flags
- **Error output routing** ΓÇö configure where `print()` and error messages are sent (stdout, stderr)
- **IEEE754 and endianness tricks** ΓÇö select fast double-to-integer conversion strategies per CPU architecture

## Key Types / Data Structures

None. This file is purely preprocessor macros and typedef aliases; no struct/enum/class definitions.

## Global / File-Static State

None. All definitions are compile-time constants (macros, typedefs). No runtime global or static variables.

## Key Functions / Methods

None. This is a configuration header. It defines macros that expand to function calls elsewhere (e.g., `luai_writestring` ΓåÆ `fwrite`), but no functions are defined here.

## Control Flow Notes

Not applicable to runtime control flow. This header affects **compilation**:
- Platform macros control which .c files #include which headers and define which functions
- `LUA_COMPAT_*` flags enable/disable entire subsystems at compile time
- Number type and math operation macros determine code generation for the Lua core VM and libraries

The bundled `config.h` (Aleph One build output) shows Lua is compiled with SDL, OpenGL, Boost, and zlib supportΓÇötypical for a game engine embedding Lua for scripting.

## External Dependencies

### Includes (Conditional)
| Header | Condition | Purpose |
|--------|-----------|---------|
| `<limits.h>` | Always | `INT_MAX` for bit-width detection |
| `<stddef.h>` | Always | `size_t`, `ptrdiff_t` |
| `<stdio.h>` | `LUA_LIB` or `lua_c` | `fwrite()`, `fprintf()` for print and error output |
| `<math.h>` | `lobject_c` or `lvm_c` | `floor()`, `pow()` for Lua math |
| `"config.h"` | `HAVE_CONFIG_H` | Build-system-generated feature macros (Aleph One) |

### Platform/Compiler Symbols Used (Defined Elsewhere)
- `__STRICT_ANSI__`, `_WIN32`, `_WIN32_WCE` ΓÇö compiler/platform macros
- `__GNUC__`, `__ELF__` ΓÇö GCC visibility attributes
- `_MSC_VER`, `_M_IX86` ΓÇö MSVC/architecture detection
- `__i386__`, `__x86_64`, `__POWERPC__` ΓÇö CPU architecture macros

### Relationship to Aleph One (from config.h)
The packaged config.h names this as **Aleph One 20140104** (a Marathon game engine remake). Lua is configured with file I/O (`<pwd.h>`), graphics (`OPENGL`, `SDL_image`), networking (`SDL_net`, `miniupnpc`), audio (`SDL_ttf`), and compression (`zlib`, `zzip`) supportΓÇötypical for embedding Lua in a game engine for scripting and modding.
