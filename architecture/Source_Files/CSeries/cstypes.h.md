# Source_Files/CSeries/cstypes.h

## File Purpose
Foundational type definitions and macros for the Aleph One game engine. Provides portable integer types with specific bit widths, fixed-point arithmetic support (16.16 format), and common constants. Ensures cross-platform compatibility through conditional compilation.

## Core Responsibilities
- Define fixed-width integer types (uint8, int8, uint16, int16, uint32, int32) via SDL
- Provide fixed-point arithmetic macros and constants (16.16 format)
- Define platform-independent constants (NONE, UNONE, MEG, KILO)
- Supply utility macros for four-character codes (resource identifiers)
- Ensure OpenGL availability fallback for non-OpenGL builds
- Define time-related types (TimeType/time_t)

## Key Types / Data Structures
| Name | Kind (typedef) | Purpose |
|------|---|---|
| `uint8`, `int8`, `uint16`, `int16`, `uint32`, `int32` | typedef | Fixed-width integer types from SDL |
| `_fixed` | typedef (int32) | 16.16 fixed-point signed integer |
| `TimeType` | typedef (time_t) | Portable time representation |
| `byte` | typedef (uint8) | Unsigned 8-bit value |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `NONE` | enum constant (int) | global | Sentinel value for "no value" (-1) |
| `UNONE` | enum constant (uint) | global | Unsigned sentinel (65535) |
| `MEG` | const int | global | 1 megabyte in bytes (0x100000) |
| `KILO` | const int | global | 1 kilobyte in bytes (0x400) |
| `FIXED_FRACTIONAL_BITS` | #define | global | Bit shift count for 16.16 fixed-point (16) |
| `FIXED_ONE`, `FIXED_ONE_HALF` | #define | global | Fixed-point unit and half-unit constants |

## Key Functions / Methods
None (type definition header only).

**Macros of note**:
- `INTEGER_TO_FIXED(i)` ΓÇö Convert integer to 16.16 fixed-point (left shift by 16)
- `FIXED_INTEGERAL_PART(f)` ΓÇö Extract integer part from fixed-point (right shift by 16)
- `FOUR_CHARS_TO_INT(a,b,c,d)` ΓÇö Pack four bytes into uint32 (resource code construction)
- `FIXED_ONE`, `FIXED_ONE_HALF` ΓÇö Constants representing 1.0 and 0.5 in fixed-point

## Control Flow Notes
Initialization-time header; no runtime control flow. Included early to provide types for all downstream modules. No frame/update/render dependencies.

## External Dependencies
- `<limits.h>` ΓÇö Integer limits
- `<SDL2/SDL_types.h>` ΓÇö Uint8, Sint8, Uint16, Sint16, Uint32, Sint32 type definitions
- `<time.h>` ΓÇö time_t definition
- `config.h` (conditional) ΓÇö Build-time feature flags (HAVE_OPENGL, etc.)
