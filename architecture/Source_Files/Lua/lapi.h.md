# Source_Files/Lua/lapi.h

## File Purpose
Header providing auxiliary macros for Lua C API stack and call frame management. Implements safety checks and state adjustments for the public API layer.

## Core Responsibilities
- Provide stack pointer management with bounds checking
- Validate call frame resources before API operations
- Adjust return value handling for multi-return functions
- Enforce API preconditions and invariants

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### api_incr_top
- Signature: `#define api_incr_top(L)`
- Purpose: Increment stack top pointer with overflow detection
- Inputs: `L` (lua_State pointer)
- Outputs/Return: None (macro, modifies in-place)
- Side effects: Increments `L->top`; calls `api_check` to validate `L->top <= L->ci->top`
- Calls: `api_check` (from llimits.h)
- Notes: Prevents stack overflow by asserting new top does not exceed call frame boundary

### adjustresults
- Signature: `#define adjustresults(L,nres)`
- Purpose: Adjust call info top pointer when multiple results returned
- Inputs: `L` (lua_State), `nres` (number of results, or `LUA_MULTRET` sentinel)
- Outputs/Return: None (macro, conditional assignment)
- Side effects: Sets `L->ci->top = L->top` if `nres == LUA_MULTRET && L->ci->top < L->top`
- Calls: None
- Notes: Handles variadic returns by extending call frame boundary to actual stack top

### api_checknelems
- Signature: `#define api_checknelems(L,n)`
- Purpose: Validate sufficient stack elements available for pop/access
- Inputs: `L` (lua_State), `n` (number of elements required)
- Outputs/Return: None (macro, assertion)
- Side effects: Calls `api_check`
- Calls: `api_check` (from llimits.h)
- Notes: Verifies `n < (L->top - L->ci->func)`, i.e., at least `n` elements exist above call frame base

## Control Flow Notes
These are utility assertions used throughout the Lua C API during function calls, stack manipulation, and return value processing. Not part of a sequential init/frame/shutdown sequence; rather, called on-demand during API operations.

## External Dependencies
- `llimits.h`: provides `api_check` macro and type limits
- `lstate.h`: provides `lua_State` structure and `CallInfo` definition
- Implicitly depends on: `lua_State::top`, `lua_State::ci`, `CallInfo::top`, `CallInfo::func`
