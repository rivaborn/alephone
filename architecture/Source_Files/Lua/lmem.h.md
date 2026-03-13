# Source_Files/Lua/lmem.h

## File Purpose
Memory manager interface for Lua VM. Provides type-safe allocation, deallocation, and dynamic growth macros with built-in overflow checking. Acts as the central API for all memory operations within the Lua interpreter.

## Core Responsibilities
- Define safe memory allocation macros with runtime overflow detection
- Provide type-safe wrappers around the core reallocation function
- Support dynamic vector/array growth with automatic sizing
- Prevent integer overflow on allocation size calculations
- Signal out-of-memory errors to the VM (via `luaM_toobig`)
- Offer unified memory management across object, vector, and array allocations

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### luaM_realloc_
- **Signature:** `void *luaM_realloc_(lua_State *L, void *block, size_t oldsize, size_t size)`
- **Purpose:** Core memory reallocation function; backend for all allocation/deallocation operations.
- **Inputs:** VM state, memory block (or NULL), old size in bytes, new size in bytes.
- **Outputs/Return:** Reallocated pointer; on failure or deallocation, behavior depends on allocator (typically calls allocator function set via `lua_setallocf`).
- **Side effects:** Modifies heap; calls user-defined allocator callback; may trigger GC indirectly.
- **Calls:** User-supplied allocator (via `lua_Alloc` callback).
- **Notes:** Not called directlyΓÇömacros wrap it. When `size == 0`, frees the block. When `block == NULL`, allocates new memory.

### luaM_growaux_
- **Signature:** `void *luaM_growaux_(lua_State *L, void *block, int *size, size_t size_elem, int limit, const char *what)`
- **Purpose:** Helper for dynamic vector growth; reallocates and updates size counter.
- **Inputs:** VM state, vector block, pointer to current size (updated in-place), element size, max limit, error description string.
- **Outputs/Return:** Reallocated vector pointer.
- **Side effects:** Modifies `*size` in-place with new capacity; calls `luaM_realloc_`.
- **Calls:** `luaM_realloc_`.
- **Notes:** Grows exponentially (typical pattern); updates size counter for caller; `what` used in error messages if limit exceeded.

### luaM_toobig
- **Signature:** `l_noret luaM_toobig(lua_State *L)`
- **Purpose:** Raise a fatal out-of-memory error and abort execution.
- **Inputs:** VM state.
- **Outputs/Return:** None (does not return).
- **Side effects:** Triggers error handling; terminates execution path.
- **Calls:** VM error/panic mechanism.
- **Notes:** Declared `l_noret` (non-returning); invoked by macros when allocation size overflows.

## Macro API Highlights
| Macro | Purpose |
|-------|---------|
| `luaM_reallocv(L,b,on,n,e)` | Realloc with compile-time overflow check; avoids runtime division. |
| `luaM_malloc(L,s)` / `luaM_new(L,t)` | Allocate uninitialized memory; `new` is type-safe (casted). |
| `luaM_newvector(L,n,t)` | Allocate array of `n` elements of type `t`. |
| `luaM_free(L,b)` / `luaM_freearray(L,b,n)` | Free single object or array. |
| `luaM_growvector(L,v,nelems,size,t,limit,e)` | Conditionally grow vector; calls `luaM_growaux_` if needed. |
| `luaM_reallocvector(L,v,oldn,n,t)` | Reallocate vector to new element count. |

## Control Flow Notes
This is a utility/support module, not part of the VM's frame loop. Called during:
- **Initialization**: String table, stack allocation
- **Runtime**: Table resize, object creation, string interning, buffer expansion
- **GC**: During garbage collection phases (indirectly, via GC code)

Not directly involved in execute/render cycles; accessed on-demand by other VM subsystems.

## External Dependencies
- **Includes:** `<stddef.h>` (size_t), `llimits.h` (MAX_SIZET, l_noret, cast macro), `lua.h` (lua_State)
- **External symbols used:** `lua_State` (opaque handle), `MAX_SIZET` (size limit constant)
