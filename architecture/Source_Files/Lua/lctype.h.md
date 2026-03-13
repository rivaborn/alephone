# Source_Files/Lua/lctype.h

## File Purpose
Provides Lua-specific character classification and case-conversion macros. Optimizes for Lua's needs by differing from standard C ctype.h, supporting both ASCII-table and standard-ctype implementations based on platform.

## Core Responsibilities
- Define character property bits (alphabetic, digit, printable, space, hex-digit)
- Provide character classification macros that test these properties on input characters
- Support both custom lookup-table and standard C ctype implementations via compile-time switch
- Include case-conversion macro for alphabetic characters
- Accommodate EOZ (end-of-zone, index -1) by off-by-one indexing into the classification table

## Key Types / Data Structures
None (macros only; external typedef `lu_byte` from llimits.h).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `luai_ctype_` | `const lu_byte[UCHAR_MAX + 2]` | global | Lookup table for character properties (ASCII-optimized path only) |

## Key Functions / Methods
None (this file defines macros, not functions).

### testprop(c, p)
- **Purpose:** Core property-test macro; checks if character `c` has property `p` by indexing the lookup table and masking.
- **Inputs:** character `c`, property bitmask `p`
- **Outputs/Return:** Result of bitwise AND (nonzero if property set).
- **Side effects:** None.
- **Notes:** Adds 1 to `c` to allow index -1 (EOZ) as a valid table index.

### ltolower(c)
- **Purpose:** Convert alphabetic character to lowercase using bitwise OR.
- **Inputs:** character `c`
- **Outputs/Return:** Lowercased character (ASCII only).
- **Side effects:** None.
- **Notes:** Assumes ASCII ('A'=65, 'a'=97); simple XOR would not work correctly here.

## Control Flow Notes
Conditional compilation at the top determines the implementation:
- If `LUA_USE_CTYPE` is 0 (ASCII, fast path): includes `<limits.h>` and defines macros using the `luai_ctype_` table.
- If `LUA_USE_CTYPE` is 1 (non-ASCII or fallback): includes `<ctype.h>` and wraps standard ctype functions, adding underscore checks.

This header is included early in the Lua pipeline to provide classification utilities for lexer/parser operations.

## External Dependencies
- **Notable includes:** `lua.h`, `llimits.h`, conditionally `<limits.h>`, conditionally `<ctype.h>`
- **External symbols used:**
  - `luai_ctype_[UCHAR_MAX + 2]` ΓÇô defined elsewhere (llex.c or similar)
  - `lu_byte` ΓÇô typedef from llimits.h
  - `UCHAR_MAX` ΓÇô from standard `<limits.h>`
