# Source_Files/Lua/ldebug.h

## File Purpose
Header for Lua's debug interface auxiliary functions. Defines debugging macros and declares error-reporting functions used to propagate runtime errors (type errors, arithmetic errors, etc.) throughout the Lua VM.

## Core Responsibilities
- Define debugging utility macros (program counter calculation, line info lookup, hook management, call info extraction)
- Declare error-reporting functions for runtime violations
- Provide non-returning error handlers for type, arithmetic, ordering, and concatenation failures

## Key Types / Data Structures
None (header declares externals; types defined in `lstate.h`).

## Global / File-Static State
None.

## Key Functions / Methods

### luaG_typeerror
- Signature: `l_noret luaG_typeerror(lua_State *L, const TValue *o, const char *opname)`
- Purpose: Report a type error when an operation is applied to an incompatible type
- Inputs: Lua state, value with invalid type, operation name (string)
- Outputs/Return: Never returns (`l_noret`)
- Side effects: Throws Lua error; terminates execution path
- Calls: (Defined elsewhere)
- Notes: Called by VM when operand type check fails

### luaG_concaterror
- Signature: `l_noret luaG_concaterror(lua_State *L, StkId p1, StkId p2)`
- Purpose: Report concatenation of non-string/non-number values
- Inputs: Lua state, two stack indices with invalid concatenation operands
- Outputs/Return: Never returns
- Side effects: Throws Lua error
- Calls: (Defined elsewhere)

### luaG_aritherror
- Signature: `l_noret luaG_aritherror(lua_State *L, const TValue *p1, const TValue *p2)`
- Purpose: Report arithmetic operation on non-numeric operands
- Inputs: Lua state, two values with invalid arithmetic types
- Outputs/Return: Never returns
- Side effects: Throws Lua error

### luaG_ordererror
- Signature: `l_noret luaG_ordererror(lua_State *L, const TValue *p1, const TValue *p2)`
- Purpose: Report comparison operation on incompatible types
- Inputs: Lua state, two values with incomparable types
- Outputs/Return: Never returns
- Side effects: Throws Lua error

### luaG_runerror
- Signature: `l_noret luaG_runerror(lua_State *L, const char *fmt, ...)`
- Purpose: Report formatted runtime error with variadic arguments
- Inputs: Lua state, printf-style format string and arguments
- Outputs/Return: Never returns
- Side effects: Throws Lua error; formats and propagates message

### luaG_errormsg
- Signature: `l_noret luaG_errormsg(lua_State *L)`
- Purpose: Propagate error message from Lua stack to error handler
- Inputs: Lua state (error message at top of stack)
- Outputs/Return: Never returns
- Side effects: Invokes error handling machinery; may trigger longjmp

## Key Macros
- `pcRel(pc, p)`: Compute bytecode offset relative to function prototype `p`
- `getfuncline(f, pc)`: Retrieve source line number for instruction at `pc` in function `f`
- `resethookcount(L)`: Reset debug hook execution counter to base value
- `ci_func(ci)`: Extract Lua closure from call info (used to get current function during execution)

## Control Flow Notes
Error-reporting functions are called during VM instruction execution when operand validation fails. All declared functions terminate control flow via Lua's error mechanism (longjmp). These form the error-propagation boundary between the C VM and Lua-level error handlers.

## External Dependencies
- **Includes**: `lstate.h` (for `lua_State`, `CallInfo`, `StkId`, `TValue` types)
- **Macros used**: `LUAI_FUNC`, `l_noret`, `cast`, `clLvalue`
- **Types defined elsewhere**: `lua_State`, `TValue`, `StkId`, `CallInfo`, `Proto`
