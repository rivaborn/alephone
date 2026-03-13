# Source_Files/Lua/lfunc.h

## File Purpose
Header file declaring the public API for Lua closure and prototype management. Provides functions to create, allocate, and free function prototypes, closures (both C and Lua), and upvaluesΓÇöthe runtime representation of nested function scopes and captured variables.

## Core Responsibilities
- Create function prototypes (bytecode templates with metadata)
- Allocate and initialize C closures (C functions with captured Lua values)
- Allocate and initialize Lua closures (Lua functions with captured variables)
- Create and track upvalue descriptors for nested function bindings
- Locate or create upvalues at specific stack levels
- Close (finalize) upvalues when scopes exit
- Deallocate prototypes and upvalues during garbage collection
- Provide debug information retrieval (local variable names by instruction pointer)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Proto` | struct | Function prototype: bytecode, constants, nested function list, debug metadata |
| `Closure` | union | Discriminated union holding either a C closure or Lua closure |
| `CClosure` | struct | C function closure with array of captured TValues |
| `LClosure` | struct | Lua function closure with array of captured UpVal pointers |
| `UpVal` | struct | Upvalue: points to a stack location or holds a closed value |
| `LocVar` | struct | Debug metadata for a local variable (name, active bytecode range) |
| `Upvaldesc` | struct | Debug metadata for an upvalue (name, stack/outer indicator, index) |

## Global / File-Static State
None.

## Key Functions / Methods

### luaF_newproto
- **Signature:** `Proto *luaF_newproto (lua_State *L)`
- **Purpose:** Allocate and initialize a new function prototype.
- **Inputs:** Lua state.
- **Outputs/Return:** Pointer to newly allocated Proto.
- **Side effects:** Allocates memory; may trigger garbage collection.
- **Calls:** (Not visible in header; defined in lfunc.c)
- **Notes:** Proto is the template for all function closures; contains bytecode and metadata.

### luaF_newCclosure
- **Signature:** `Closure *luaF_newCclosure (lua_State *L, int nelems)`
- **Purpose:** Allocate a C closure with space for `nelems` upvalues.
- **Inputs:** Lua state; number of captured TValues.
- **Outputs/Return:** Pointer to newly allocated Closure (C variant).
- **Side effects:** Allocates variable-size block (calculated by `sizeCclosure` macro).
- **Calls:** (Not visible in header)
- **Notes:** nelems is the count of upvalues to capture; size is layout-dependent.

### luaF_newLclosure
- **Signature:** `Closure *luaF_newLclosure (lua_State *L, int nelems)`
- **Purpose:** Allocate a Lua closure with space for `nelems` upvalue pointers.
- **Inputs:** Lua state; number of captured upvalues.
- **Outputs/Return:** Pointer to newly allocated Closure (Lua variant).
- **Side effects:** Allocates variable-size block (calculated by `sizeLclosure` macro).
- **Calls:** (Not visible in header)
- **Notes:** Similar to `newCclosure` but holds UpVal pointers instead of raw TValues.

### luaF_newupval
- **Signature:** `UpVal *luaF_newupval (lua_State *L)`
- **Purpose:** Allocate a new open upvalue.
- **Inputs:** Lua state.
- **Outputs/Return:** Pointer to newly allocated UpVal.
- **Side effects:** Allocates; upvalue initially points to stack or holds a value.
- **Calls:** (Not visible in header)

### luaF_findupval
- **Signature:** `UpVal *luaF_findupval (lua_State *L, StkId level)`
- **Purpose:** Find or create an upvalue for the variable at stack position `level`.
- **Inputs:** Lua state; stack index (StkId = TValue pointer).
- **Outputs/Return:** UpVal for that position, reusing if already open.
- **Side effects:** May allocate; maintains open upvalue linked list.
- **Calls:** (Not visible in header)
- **Notes:** Called during function definition to bind free variables; manages a doubly-linked list of open upvalues.

### luaF_close
- **Signature:** `void luaF_close (lua_State *L, StkId level)`
- **Purpose:** Close all open upvalues at or above `level` on the stack.
- **Inputs:** Lua state; stack threshold.
- **Outputs/Return:** None.
- **Side effects:** Converts open upvalues to closed (copies their values off the stack).
- **Calls:** (Not visible in header)
- **Notes:** Called when exiting a scope to capture final variable values before stack unwinding.

### luaF_freeproto & luaF_freeupval
- **Purpose:** Deallocate a prototype or upvalue during garbage collection.
- **Signature:** `void luaF_freeproto (lua_State *L, Proto *f)` / `void luaF_freeupval (lua_State *L, UpVal *uv)`
- **Side effects:** Releases memory; assumes object is unreachable.
- **Notes:** Called by GC; not directly invoked by user code.

### luaF_getlocalname
- **Signature:** `const char *luaF_getlocalname (const Proto *func, int local_number, int pc)`
- **Purpose:** Retrieve the name of a local variable for debug introspection.
- **Inputs:** Function prototype; local variable index; program counter (bytecode position).
- **Outputs/Return:** Variable name string, or NULL if not available.
- **Side effects:** None.
- **Calls:** (Not visible in header)
- **Notes:** Used by debuggers and error messages; names only available if debug info was retained.

## Control Flow Notes
This header defines allocation and lifecycle primitives for Lua's function system. Functions are typically called during:
- **Function definition** (luaF_newproto, luaF_newLclosure, luaF_findupval) when the parser creates a closure.
- **Scope entry/exit** (luaF_close) during function call stack management and return.
- **Garbage collection** (luaF_freeproto, luaF_freeupval) when objects become unreachable.
- **Debug/introspection** (luaF_getlocalname) during error reporting and debugging.

## External Dependencies
- **`lobject.h`:** Provides Proto, Closure, CClosure, LClosure, UpVal, TValue, Upvaldesc, LocVar types.
- **`lua.h`:** Provides lua_State, lua_CFunction, and core type definitions.
- **`LUAI_FUNC` macro:** Used for extern function declarations; allows platform-specific visibility/calling conventions.

**Notes:** 
- Size macros `sizeCclosure` and `sizeLclosure` use pointer arithmetic to calculate variable-size allocations.
- All functions operate on Lua state and assume the GC system manages lifetime.
