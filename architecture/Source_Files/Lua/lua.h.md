# Source_Files/Lua/lua.h

## File Purpose

Public C API header for Lua 5.2 interpreter. Defines the complete interface for embedding Lua in C applications, including state lifecycle management, stack operations, value conversions, code execution, and debug introspection. All core runtime operations between C and Lua communicate via the value stack defined here.

## Core Responsibilities

- **State lifecycle**: Create/destroy Lua execution states and threads
- **Value stack management**: Push, pop, peek, and manipulate values on the stack
- **Type system and conversions**: Query types and convert between Lua and C representations
- **Execution control**: Load and execute Lua code, call Lua functions from C
- **Memory management**: Define allocator callbacks, control garbage collection
- **Coroutine/threading**: Resume suspended Lua coroutines from C
- **Debug support**: Set hooks, inspect stack frames, query function metadata
- **Utility macros**: Convenience wrappers for common stack operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `lua_State` | opaque struct | Lua execution state; all operations require a pointer to an active state |
| `lua_CFunction` | function pointer typedef | Signature for C functions callable from Lua; `int (*)(lua_State *L)` |
| `lua_Reader` | function pointer typedef | Callback for streaming Lua code chunks during load; reads and returns buffer + size |
| `lua_Writer` | function pointer typedef | Callback for writing serialized Lua code during dump |
| `lua_Alloc` | function pointer typedef | Callback for all memory allocation/deallocation; `void* (*)(void *ud, void *ptr, size_t osize, size_t nsize)` |
| `lua_Hook` | function pointer typedef | Debug hook callback; `void (*)(lua_State *L, lua_Debug *ar)` |
| `lua_Debug` | struct | Activation record containing function/source metadata for debugging (event, name, source, line numbers, upvalue count, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `lua_ident` | `const char[]` | extern | RCS identifier string (implementation provided elsewhere) |

## Key Functions / Methods

### lua_newstate
- **Signature:** `lua_State *(lua_newstate)(lua_Alloc f, void *ud)`
- **Purpose:** Create a new Lua execution state with custom memory allocator.
- **Inputs:** `f` = allocator callback, `ud` = opaque data passed to allocator
- **Outputs/Return:** Pointer to new `lua_State`, or NULL on allocation failure
- **Side effects:** Allocates memory, initializes global state
- **Calls:** Custom allocator `f`

### lua_close
- **Signature:** `void (lua_close)(lua_State *L)`
- **Purpose:** Destroy a Lua state and free all associated memory.
- **Inputs:** `L` = state to close
- **Side effects:** Deallocates all memory, runs finalizers, calls allocator with nsize=0

### lua_gettop / lua_settop
- **Signature:** `int (lua_gettop)(lua_State *L)` / `void (lua_settop)(lua_State *L, int idx)`
- **Purpose:** Query or modify stack height
- **Inputs:** `idx` = new stack height (for settop); negative indices adjust relative to current top
- **Outputs/Return:** Current stack height (for gettop)

### lua_pushvalue / lua_remove / lua_insert / lua_replace / lua_copy
- **Purpose:** Stack manipulation: duplicate, remove, or rearrange values
- **Inputs:** `idx` = stack index to operate on, `fromidx`/`toidx` for copy operations
- **Side effects:** Modify stack structure

### lua_type / lua_isnumber / lua_isstring / lua_istable / lua_isfunction
- **Purpose:** Query type of value at stack position
- **Inputs:** `idx` = stack index, optionally `tp` = type constant to check against
- **Outputs/Return:** Type constant (LUA_TNIL, LUA_TNUMBER, etc.) or boolean

### lua_tonumberx / lua_tointegerx / lua_toboolean / lua_tolstring / lua_tocfunction / lua_touserdata / lua_tothread / lua_topointer
- **Purpose:** Convert stack value to C representation
- **Inputs:** `idx` = stack index, `isnum` = output parameter to indicate success (optional)
- **Outputs/Return:** Converted value in C type (double, int, bool, char*, etc.)
- **Notes:** Return false-equivalents if type mismatch; `lua_tolstring` returns pointer + length

### lua_pushnil / lua_pushnumber / lua_pushinteger / lua_pushstring / lua_pushboolean / lua_pushcclosure / lua_pushthread
- **Purpose:** Push C values onto the stack
- **Inputs:** Value to push; for `lua_pushcclosure`, function pointer `fn` and upvalue count `n`
- **Side effects:** Increments stack height; for closure, pops `n` upvalues from stack

### lua_getglobal / lua_setglobal / lua_gettable / lua_settable / lua_getfield / lua_setfield / lua_rawget / lua_rawset
- **Purpose:** Access Lua table/global values
- **Inputs:** Table stack index, key (string name or stack index), value (for set operations)
- **Side effects:** May trigger metamethods (except raw variants); modifies stack

### lua_callk / lua_pcallk / lua_call / lua_pcall
- **Signature:** `void (lua_callk)(lua_State *L, int nargs, int nresults, int ctx, lua_CFunction k)` (macro: `lua_call(L,n,r)`)
- **Purpose:** Call a Lua function on the stack; `pcall` version catches errors
- **Inputs:** `nargs` = number of arguments on stack, `nresults` = expected return values (or LUA_MULTRET), `errfunc` = error handler stack index (pcall only)
- **Outputs/Return:** 0 (LUA_OK) on success, error code on failure (pcall); return values left on stack
- **Side effects:** Executes Lua code, may modify stack significantly
- **Notes:** Function and arguments must be on stack before call; pops them, pushes results

### lua_load
- **Signature:** `int (lua_load)(lua_State *L, lua_Reader reader, void *dt, const char *chunkname, const char *mode)`
- **Purpose:** Load Lua source or bytecode chunks via reader callback
- **Inputs:** `reader` = callback to fetch chunks, `chunkname` = name for error messages, `mode` = "t" (text), "b" (binary), or "bt" (both)
- **Outputs/Return:** 0 on success; nonzero on syntax error (error message on stack)
- **Side effects:** Calls reader callback repeatedly, pushes loaded function onto stack

### lua_dump
- **Signature:** `int (lua_dump)(lua_State *L, lua_Writer writer, void *data)`
- **Purpose:** Serialize a Lua function to bytecode via writer callback
- **Inputs:** Function at top of stack, `writer` = callback to receive bytecode chunks
- **Side effects:** Calls writer callback with bytecode fragments

### lua_yieldk / lua_yield
- **Signature:** `int (lua_yieldk)(lua_State *L, int nresults, int ctx, lua_CFunction k)` (macro: `lua_yield(L,n)`)
- **Purpose:** Suspend current Lua coroutine, returning control and values to caller
- **Inputs:** `nresults` = number of values being returned
- **Outputs/Return:** Error code or continuation context

### lua_resume
- **Signature:** `int (lua_resume)(lua_State *L, lua_State *from, int narg)`
- **Purpose:** Resume a suspended coroutine
- **Inputs:** `from` = state resuming this one (for cross-state resumption), `narg` = number of arguments on stack
- **Outputs/Return:** LUA_OK if yields/returns, error code if fails; results on stack

### lua_gc
- **Signature:** `int (lua_gc)(lua_State *L, int what, int data)`
- **Purpose:** Control garbage collection behavior
- **Inputs:** `what` = command (LUA_GCCOLLECT, LUA_GCSTOP, LUA_GCCOUNT, etc.), `data` = optional parameter
- **Outputs/Return:** Command-dependent (e.g., byte count for LUA_GCCOUNT)

### lua_getstack / lua_getinfo / lua_getlocal / lua_setlocal / lua_getupvalue / lua_setupvalue
- **Purpose:** Debug introspection: query call stack, function metadata, local variables, upvalues
- **Inputs:** `level` = call stack depth, `ar` = debug info struct (output/input), `n` = variable index
- **Outputs/Return:** Non-zero on success; fills `lua_Debug` struct with frame info and variable names

### lua_sethook / lua_gethook / lua_gethookmask / lua_gethookcount
- **Purpose:** Install/query debug hooks that fire on function calls, returns, lines, or counts
- **Inputs:** `func` = hook callback, `mask` = events to hook (bitmask of LUA_MASKCALL, etc.), `count` = line/instruction interval
- **Side effects:** Modifies interpreter's debug behavior; hooks called asynchronously during execution

## Control Flow Notes

**Lifecycle:** Application calls `lua_newstate()` to initialize, loads/executes Lua code via `lua_load()` + `lua_callk()`, accesses results via stack read functions, then `lua_close()` on shutdown.

**Stack-based communication:** All Lua-C interaction uses the value stack as the transfer medium. C code reads from Lua (via `lua_toXXX` functions), modifies/creates values (via `lua_pushXXX`), and invokes Lua functions (via `lua_callk`).

**Error handling:** Synchronous errors from `lua_callk` are propagated; asynchronous errors in `lua_pcallk` are caught and error objects left on stack.

**Coroutines/threads:** New threads created with `lua_newthread()` share global state but have independent call stacks. Resumed with `lua_resume()` from any thread.

Not inferable from this file: actual VM execution semantics, internal stack implementation, GC algorithm details (all in separate source files like lvm.c, lstate.c, lgc.c).

## External Dependencies

- **Standard C headers:** `<stdarg.h>` (va_list), `<stddef.h>` (size_t, ptrdiff_t)
- **Local configuration:** `"luaconf.h"` ΓÇö platform-specific settings, type definitions (LUA_NUMBER, LUA_INTEGER), API export macros (LUA_API, LUALIB_API), allocator signatures
- **Optional user header:** `LUA_USER_H` (if defined at compile-time) ΓÇö allows embedding custom extensions
- **Defined elsewhere:** All function implementations (state.c, stack.c, vm.c, etc.), `lua_ident` string, `lua_Debug` struct full definition
