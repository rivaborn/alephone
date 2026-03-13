# Source_Files/Lua/llimits.h

## File Purpose
Internal Lua header defining platform-specific limits, portable type aliases, and debug/safety macros. Handles compiler-specific features (MSVC, GCC) and supports multiple number-to-integer conversion strategies.

## Core Responsibilities
- Define portable integer/memory types (lu_int32, lu_mem, lu_byte, Instruction)
- Enforce safe limits (MAX_SIZET, MAX_LUMEM, MAX_INT, MAXSTACK, MAXUPVAL)
- Provide type-casting macros with safety checks
- Supply debug assertions (lua_assert, api_check, check_exp)
- Implement numberΓåöinteger conversions with platform-specific optimizations (IEEE754, MSVC assembler)
- Provide thread locking/yielding hooks
- Define user-state lifecycle callbacks for threads

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `lu_int32` | typedef | Portable 32-bit unsigned integer |
| `lu_mem` | typedef | Memory size type (LUAI_UMEM, typically size_t) |
| `l_mem` | typedef | Signed memory type (LUAI_MEM) |
| `lu_byte` | typedef | Unsigned char for small naturals |
| `L_Umaxalign` | union typedef | Maximum-alignment type (double, void*, long) |
| `l_uacNumber` | typedef | "Usual argument conversion" number type |
| `Instruction` | typedef | VM instruction (lu_int32) |

## Global / File-Static State
None.

## Key Functions / Methods

### lua_number2int / lua_number2integer / lua_number2unsigned
- **Purpose:** Convert Lua floating-point numbers to C integer types.
- **Inputs:** `i` (output variable), `n` (lua_Number value)
- **Outputs/Return:** Assigns result to `i` (macro, no return value)
- **Side effects:** None (pure conversion)
- **Notes:** Multiple implementations selected at compile time:
  - MSVC assembler (MS_ASMTRICK): inline FPU instructions (fastest)
  - IEEE754 trick (LUA_IEEE754TRICK): extract bits directly from double union (portable)
  - Fallback: simple C cast (slowest)

### luai_hashnum
- **Purpose:** Deterministically hash a lua_Number into an integer for table lookup.
- **Inputs:** `i` (output int), `n` (lua_Number value)
- **Outputs/Return:** Assigns result to `i`
- **Side effects:** None
- **Notes:** Handles edge cases (┬▒0, infinity) using IEEE754 bit manipulation or frexp decomposition to avoid collisions.

### lua_unsigned2number
- **Purpose:** Convert C unsigned integer to lua_Number with optimized path for small values.
- **Inputs:** `u` (lua_Unsigned)
- **Outputs/Return:** Converted lua_Number
- **Notes:** Uses conditional int-cast for values Γëñ INT_MAX to avoid slow unsignedΓåÆdouble coercion on some platforms.

## Control Flow Notes
This is a configuration header included early in Lua's compile chain (via lua.h ΓåÆ luaconf.h ΓåÆ llimits.h). It does not participate in runtime flow; all macros expand at compile time. Numbers are converted on the boundary between Lua script values and C code (API calls, arithmetic ops).

## External Dependencies
- **Standard C:** `<limits.h>` (INT_MAX, UCHAR_MAX), `<stddef.h>` (size_t), `<float.h>` (DBL_MAX_EXP, conditionally), `<math.h>` (frexp, floor, conditionally), `<assert.h>` (conditionally)
- **Lua internals:** `lua.h` (defines lua_Number, lua_Integer, lua_Unsigned); `luaconf.h` (user customization macros: LUAI_UMEM, LUAI_MEM, LUAI_MAXCCALLS, LUA_IEEE754TRICK, MS_ASMTRICK, etc.)
- **Referenced but not defined here:** luaD_reallocstack, luaC_fullgc, G() macro (defined elsewhere in Lua internals)
