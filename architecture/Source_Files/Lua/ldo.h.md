# Source_Files/Lua/ldo.h

## File Purpose
Defines the stack management and protected execution framework for Lua's virtual machine. Provides macros and function declarations for stack bounds checking, function call setup, exception handling, and debugging hooks.

## Core Responsibilities
- Stack overflow prevention and dynamic resizing
- Protected execution of Lua code and C functions with exception handling
- Function call protocol (pre-call setup, post-call cleanup)
- Debug hook invocation
- Parser invocation with error containment
- Stack pointer serialization/restoration for relocatable stacks

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Pfunc` | typedef | Function pointer for protected execution callbacks: `void (*)(lua_State *L, void *ud)` |

## Global / File-Static State
None.

## Key Functions / Methods

### luaD_checkstack
- Signature: `#define luaD_checkstack(L,n)` ΓÇö macro
- Purpose: Ensure the stack has at least `n` free slots; grow or relocate if needed
- Inputs: `L` (Lua state), `n` (required free slots)
- Outputs/Return: None (in-place stack adjustment)
- Side effects: May reallocate stack, calls `luaD_growstack()` or `condmovestack()`
- Notes: Called frequently to guard stack operations; assumes reallocation is expensive but necessary

### incr_top
- Signature: `#define incr_top(L)` ΓÇö macro
- Purpose: Increment stack top pointer and check bounds
- Inputs: `L` (Lua state)
- Outputs/Return: None
- Side effects: Increments `L->top`, calls `luaD_checkstack()`
- Notes: Must be used instead of bare `L->top++` to maintain invariants

### savestack / restorestack
- Signature: `#define savestack(L,p)` / `#define restorestack(L,n)`
- Purpose: Convert stack pointers to offsets and back; enables safe stack relocation
- Inputs: `L` (Lua state), `p` or `n` (pointer or offset)
- Outputs/Return: Byte offset or TValue pointer
- Notes: Used during stack growth to invalidate old pointers without losing references

### luaD_protectedparser
- Signature: `int luaD_protectedparser(lua_State *L, ZIO *z, const char *name, const char *mode)`
- Purpose: Parse Lua source with exception containment
- Inputs: `L`, `z` (input stream), `name` (source name for debug), `mode` (parsing mode)
- Outputs/Return: Error status (0 = success, nonzero = error)
- Side effects: May allocate on VM stack; error code set in `L->status`
- Calls: Parser and exception handler

### luaD_call
- Signature: `void luaD_call(lua_State *L, StkId func, int nResults, int allowyield)`
- Purpose: Execute a Lua or C function with no exception handling
- Inputs: `L`, `func` (function to call on stack), `nResults` (expected returns), `allowyield` (yield permission)
- Outputs/Return: None; results placed on stack
- Side effects: Modifies stack, may invoke metamethods, calls `luaD_precall()` and `luaV_execute()`
- Notes: Unprotected; exceptions propagate to caller

### luaD_pcall
- Signature: `int luaD_pcall(lua_State *L, Pfunc func, void *u, ptrdiff_t oldtop, ptrdiff_t ef)`
- Purpose: Protected function call with error handler
- Inputs: `L`, `func` (callback to execute), `u` (user data), `oldtop` (saved stack top), `ef` (error handler stack index)
- Outputs/Return: Error status (0 = success, nonzero = error code)
- Side effects: Catches exceptions, restores stack on failure, calls error handler
- Notes: Primary mechanism for insulating C code from Lua errors

### luaD_throw
- Signature: `l_noret luaD_throw(lua_State *L, int errcode)`
- Purpose: Throw an exception with error code
- Inputs: `L`, `errcode` (error type)
- Outputs/Return: Never returns (noreturn)
- Side effects: Longjmp to nearest error handler
- Notes: Used internally; applications should use `luaD_pcall()`

---

### luaD_precall, luaD_poscall, luaD_rawrunprotected, luaD_hook, luaD_reallocstack, luaD_growstack, luaD_shrinkstack
- **Purpose (summary):** Call protocol setup, post-call frame cleanup, unprotected callback execution, debug hook dispatch, and stack allocation control
- **Notes:** These are low-level execution primitives; most interactions go through `luaD_call()` and `luaD_pcall()`

## Control Flow Notes
**Frame Execution:** When Lua executes a function, `luaD_precall()` sets up a new call frame (`CallInfo`), then `luaV_execute()` runs bytecode. After the function returns, `luaD_poscall()` adjusts the stack and pops the frame.

**Error Handling:** Exceptions use `setjmp/longjmp` (via `lua_longjmp` in lstate.h). Protected calls set up a recovery point; `luaD_throw()` jumps back to it.

**Stack Management:** The stack grows dynamically. `luaD_checkstack()` is called before operations that push values; if space is insufficient, `luaD_growstack()` reallocates and updates all saved pointers via `restorestack()`.

**Initialization:** `luaD_reallocstack()` allocates the initial stack during state creation.

## External Dependencies
- **lobject.h** ΓÇö `TValue`, `StkId`, `Closure`, `Proto` type definitions
- **lstate.h** ΓÇö `lua_State`, `global_State`, `CallInfo` structures; `G(L)` macro
- **lzio.h** ΓÇö `ZIO` stream type for parser input
- **lua.h** ΓÇö Public Lua API types and error codes (referenced but not included in this snippet)
