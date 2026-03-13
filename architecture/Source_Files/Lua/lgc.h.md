# Source_Files/Lua/lgc.h

## File Purpose
Garbage collector interface and macros for Lua's tri-color mark-and-sweep GC. Defines the color-marking scheme, GC state machine, write barriers to maintain GC invariants, and declares all GC-related functions.

## Core Responsibilities
- Define tri-color (white/gray/black) marking system for incremental collection
- Manage GC state transitions (propagate ΓåÆ atomic ΓåÆ sweep phases)
- Provide write barriers (`luaC_barrier*` macros) to maintain the blackΓåÆwhite invariant
- Configure memory allocation step sizes and pause intervals
- Declare GC step, full collection, and object allocation functions
- Support both normal and generational garbage collection modes

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### luaC_step
- Signature: `void luaC_step(lua_State *L)`
- Purpose: Perform one incremental GC step
- Inputs: Current Lua state
- Outputs/Return: None
- Side effects: Advances GC state machine, marks/sweeps objects, modifies global state
- Calls: Defined in `lgc.c` (not visible here)
- Notes: Macro `luaC_checkGC` invokes this when `GCdebt > 0`

### luaC_forcestep
- Signature: `void luaC_forcestep(lua_State *L)`
- Purpose: Force an immediate GC step regardless of debt
- Inputs: Current Lua state
- Outputs/Return: None
- Side effects: Triggers GC collection phase
- Calls: Defined in `lgc.c`

### luaC_fullgc
- Signature: `void luaC_fullgc(lua_State *L, int isemergency)`
- Purpose: Run a complete garbage collection cycle
- Inputs: `L` (Lua state), `isemergency` (flag for emergency vs. normal mode)
- Outputs/Return: None
- Side effects: Completes all GC phases, frees unreachable objects
- Calls: Defined in `lgc.c`
- Notes: Called during shutdown or memory pressure

### luaC_newobj
- Signature: `GCObject *luaC_newobj(lua_State *L, int tt, size_t sz, GCObject **list, int offset)`
- Purpose: Allocate and initialize a new collectable object
- Inputs: Type tag `tt`, size `sz`, GC list insertion point, offset within object
- Outputs/Return: Pointer to newly allocated GCObject
- Side effects: Updates memory accounting, inserts into GC list
- Calls: Defined in `lgc.c`

### luaC_barrier_
- Signature: `void luaC_barrier_(lua_State *L, GCObject *o, GCObject *v)`
- Purpose: Write barrier when assigning a GC reference; called by `luaC_barrier` macro
- Inputs: `o` (black object receiving assignment), `v` (white object being assigned)
- Outputs/Return: None
- Side effects: Marks `v` as gray to preserve the blackΓåÆwhite invariant
- Calls: Defined in `lgc.c`
- Notes: Only invoked if `v` is white and `o` is black

### luaC_barrierback_
- Signature: `void luaC_barrierback_(lua_State *L, GCObject *o)`
- Purpose: Backward barrier for table assignments; less precise than forward barrier
- Inputs: `o` (black object, typically a table)
- Outputs/Return: None
- Side effects: Marks the entire object for re-scanning in atomic phase
- Calls: Defined in `lgc.c`

### luaC_barrierproto_
- Signature: `void luaC_barrierproto_(lua_State *L, Proto *p, Closure *c)`
- Purpose: Barrier for prototype function references in closures
- Inputs: `p` (black proto), `c` (white closure)
- Outputs/Return: None
- Side effects: Marks closure as gray
- Calls: Defined in `lgc.c`

### luaC_checkupvalcolor
- Signature: `void luaC_checkupvalcolor(global_State *g, UpVal *uv)`
- Purpose: Validate and fix upvalue color consistency during atomic phase
- Inputs: Global state, upvalue to check
- Outputs/Return: None
- Side effects: May recolor upvalue if needed
- Calls: Defined in `lgc.c`

### luaC_changemode
- Signature: `void luaC_changemode(lua_State *L, int mode)`
- Purpose: Switch between normal and generational GC modes
- Inputs: `mode` (KGC_NORMAL or KGC_GEN)
- Outputs/Return: None
- Side effects: Reconfigures GC strategy
- Calls: Defined in `lgc.c`

### luaC_freeallobjects
- Signature: `void luaC_freeallobjects(lua_State *L)`
- Purpose: Free all remaining collectable objects (shutdown path)
- Inputs: Lua state
- Outputs/Return: None
- Side effects: Deallocates all GC objects
- Calls: Defined in `lgc.c`

### luaC_runtilstate
- Signature: `void luaC_runtilstate(lua_State *L, int statesmask)`
- Purpose: Run GC steps until reaching a state matching the provided mask
- Inputs: `statesmask` (bitmask of desired GC states)
- Outputs/Return: None
- Side effects: Advances GC state machine
- Calls: Defined in `lgc.c`

## Control Flow Notes
The GC operates as a state machine driven by `luaC_step`:
- **GCSpropagate**: Marking phase; traverse gray objects and recolor to black
- **GCSatomic**: Stop-the-world mark completion; handle upvalues and weak tables
- **GCSsweepstring**: Sweep string table
- **GCSsweepudata**: Sweep userdata objects
- **GCSsweep**: Sweep main object list
- **GCSpause**: Wait until next collection cycle

Write barriers (`luaC_barrier`, `luaC_barrierback`, etc.) are invoked during normal execution whenever a black object receives a white reference, ensuring the invariant blackΓåÆwhite never occurs. The macros `luaC_condGC` and `luaC_checkGC` trigger incremental steps when memory debt accumulates.

## External Dependencies
- **lobject.h**: `GCObject`, `GCheader`, `TString`, `Udata`, `Proto`, `Closure`, `UpVal`, `Table`, `iscollectable()`, `gcvalue()`
- **lstate.h**: `lua_State`, `global_State`, `G()` macro for accessing global state from thread
