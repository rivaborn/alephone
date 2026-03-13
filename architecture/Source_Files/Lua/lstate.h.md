# Source_Files/Lua/lstate.h

## File Purpose
Defines the global and per-thread state structures that form the core of a Lua VM instance. Manages execution state, call information, garbage collection metadata, and string interning for all threads within a single Lua state.

## Core Responsibilities
- Define `global_State` struct: GC state, memory allocator, string table, metatables, GC lists, thread registry
- Define `lua_State` struct: per-thread execution context, stack, call info, hooks, upvalues
- Define `CallInfo` struct: call stack frame metadata (function index, expected results, C/Lua function union)
- Define `stringtable` struct: hash table for string interning
- Define `GCObject` union: type-safe container for all GC-managed objects (strings, tables, closures, threads, userdata, upvalues)
- Provide type conversion macros (e.g., `gco2ts`, `gco2t`) for safe casting from `GCObject` to specific types
- Define GC operation kinds and CallInfo status flags

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `global_State` | struct | Shared VM state: memory, GC, string table, metatables, thread list |
| `lua_State` | struct | Per-thread execution context: stack, call info, hooks, error handling |
| `CallInfo` | struct | Call frame metadata: function address, stack bounds, Lua/C function state union |
| `stringtable` | struct | Hash table for string interning: hash array, element count, size |
| `GCObject` | union | Type-safe container for all collectable objects |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `struct lua_longjmp` | struct (forward decl) | file-scoped forward declaration | Declared in `ldo.c`; used for exception/error jump points |

## Key Functions / Methods

### luaE_setdebt
- Signature: `void luaE_setdebt(global_State *g, l_mem debt)`
- Purpose: Update GC debt (bytes allocated but not yet compensated by collection)
- Inputs: Global state, debt amount
- Outputs/Return: None
- Side effects: Modifies `g->GCdebt`
- Calls: (not inferable from this file)
- Notes: Part of GC memory accounting; prevents unbounded allocation before collection

### luaE_freethread
- Signature: `void luaE_freethread(lua_State *L, lua_State *L1)`
- Purpose: Free a Lua thread and its associated resources
- Inputs: Parent state, thread to free
- Outputs/Return: None
- Side effects: Deallocates thread stack, call info, upvalues; updates GC lists
- Calls: (not inferable from this file)
- Notes: Thread must be deregistered from global state before calling

### luaE_extendCI
- Signature: `CallInfo *luaE_extendCI(lua_State *L)`
- Purpose: Allocate or extend call info stack when depth increases
- Inputs: Thread state
- Outputs/Return: Pointer to new `CallInfo` entry
- Side effects: May reallocate call info array; updates call stack depth
- Calls: (not inferable from this file)
- Notes: Called during function invocation when current call info is exhausted

### luaE_freeCI
- Signature: `void luaE_freeCI(lua_State *L)`
- Purpose: Free/deallocate call info structures for a thread
- Inputs: Thread state
- Outputs/Return: None
- Side effects: Deallocates call info array; resets call stack
- Calls: (not inferable from this file)
- Notes: Typically called during thread shutdown

## Control Flow Notes
- **Initialization**: When `lua_newstate()` is called (declared in `lua.h`), a `global_State` and main `lua_State` are created.
- **Per-invocation**: Each function call (Lua or C) pushes a `CallInfo` onto the thread's call stack; `luaE_extendCI` grows the stack if needed.
- **GC cycle**: Garbage collector uses `global_State.allgc`, `gray`, `weak`, `tobefnz` lists to traverse and mark objects.
- **Shutdown**: `lua_close()` triggers `luaE_freethread()` on all threads, then deallocates global state.

## External Dependencies
- **Includes**: `lua.h` (base API), `lobject.h` (type definitions and GC header), `ltm.h` (tag methods/metatables), `lzio.h` (buffered I/O)
- **Defined elsewhere**:
  - `lua_longjmp` (in `ldo.c`) ΓÇö exception handling jump buffer
  - `GCObject`, `TValue`, `TString`, `Udata`, `Closure`, `UpVal`, `Proto`, `Table` (in `lobject.h`)
  - `Mbuffer` (in `lzio.h`) ΓÇö temporary buffer for string concatenation
  - GC internals (`luaC_*` functions) ΓÇö garbage collection implementation in `lgc.c`
