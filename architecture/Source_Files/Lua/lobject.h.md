# Source_Files/Lua/lobject.h

## File Purpose

Defines the core type system and object representation for Lua. Establishes tagged values (type + data pair), garbage-collectable object structures, and provides extensive macros for type checking and value manipulation. Foundation for the entire Lua runtime.

## Core Responsibilities

- Define Lua type tags and variant bits (function subtypes, string types, collectable markers)
- Define `TValue` (tagged value) structureΓÇöthe fundamental representation of all Lua values
- Define garbage-collectable object types: strings, tables, closures, upvalues, userdata, threads, prototypes
- Provide type-checking macros (`ttisnil`, `ttisstring`, `ttistable`, etc.)
- Provide value access macros with type assertions (`nvalue`, `tsvalue`, `hvalue`, etc.)
- Provide value-setting macros with GC liveness checks
- Implement optional NaN trick for efficient IEEE 754 number representation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `TValue` | struct | Tagged value: `Value` union + `int` type tag |
| `Value` | union | Holds data: GC pointer, light userdata, boolean, C function, or number |
| `GCObject` | union | Base type for all collectable objects (forward-declared) |
| `GCheader` | struct | Common header embedded in all GC objects (next pointer, type tag, mark byte) |
| `TString` | union | String object with hash, length; includes CommonHeader |
| `Udata` | union | Userdata (opaque C memory) with metatable, environment, length |
| `Proto` | struct | Function prototype: bytecode, constants, nested protos, debug info (locals, upvalues, line info) |
| `UpVal` | struct | Upvalue reference: points to stack or closed value; doubly-linked list when open |
| `CClosure` | struct | C function closure: function pointer + upvalue array |
| `LClosure` | struct | Lua function closure: proto pointer + upvalue array |
| `Closure` | union | Union of C and Lua closures |
| `Table` | struct | Hash table: array part + hash part (node array), metatable, GC list |
| `Node` | struct | Single table entry: value + key (with chaining) |
| `TKey` | union | Table key wrapper: embedded TValue + next pointer for collision chains |
| `LocVar` | struct | Debug info for local variable: name, activation range (startpcΓÇôendpc) |
| `Upvaldesc` | struct | Debug info for upvalue: name, in-stack flag, index |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `luaO_nilobject_` | `const TValue` | global | Canonical nil constant, referenced by macro `luaO_nilobject` |

## Key Functions / Methods

### luaO_int2fb
- **Signature:** `int luaO_int2fb(unsigned int x)`
- **Purpose:** Convert unsigned integer to Lua float-byte format (compact number encoding)
- **Inputs:** Unsigned integer
- **Outputs/Return:** Encoded byte representation
- **Notes:** Used for compact bytecode and object layout encoding

### luaO_fb2int
- **Signature:** `int luaO_fb2int(int x)`
- **Purpose:** Decode Lua float-byte format back to unsigned integer
- **Inputs:** Encoded byte
- **Outputs/Return:** Unsigned integer
- **Notes:** Inverse of `luaO_int2fb`

### luaO_ceillog2
- **Signature:** `int luaO_ceillog2(unsigned int x)`
- **Purpose:** Compute ceiling of logΓéé(x)
- **Notes:** Used for hash table size calculations

### luaO_arith
- **Signature:** `lua_Number luaO_arith(int op, lua_Number v1, lua_Number v2)`
- **Purpose:** Perform arithmetic operation on two numbers (add, sub, mul, div, mod, pow, unary minus)
- **Inputs:** Operation code (`LUA_OPADD`, etc.), two operands
- **Outputs/Return:** Result as `lua_Number`

### luaO_str2d
- **Signature:** `int luaO_str2d(const char *s, size_t len, lua_Number *result)`
- **Purpose:** Parse string to double; supports decimals and hex literals
- **Inputs:** String, length, output pointer
- **Outputs/Return:** Non-zero if successful; result written to output pointer

### luaO_hexavalue
- **Signature:** `int luaO_hexavalue(int c)`
- **Purpose:** Convert hexadecimal character to 0ΓÇô15; -1 if not hex
- **Notes:** Used in number parsing

### luaO_pushvfstring, luaO_pushfstring
- **Signature:** `const char *luaO_pushvfstring(lua_State *L, const char *fmt, va_list argp)`; `const char *luaO_pushfstring(lua_State *L, const char *fmt, ...)`
- **Purpose:** Format and push string onto Lua stack (like `sprintf` + push)
- **Side effects:** Allocates and pushes new string on stack
- **Returns:** Pointer to formatted string

### luaO_chunkid
- **Signature:** `void luaO_chunkid(char *out, const char *source, size_t len)`
- **Purpose:** Extract short chunk identifier from source name for debug output
- **Inputs:** Output buffer, source name, buffer size
- **Side effects:** Writes identifier to output buffer

## Control Flow Notes

This is a **type-definition and macro header**ΓÇödoes not participate directly in init/frame/render cycles. However, its macros are pervasive throughout the VM: every stack operation, GC interaction, and value manipulation uses `TValue` and the type/access macros defined here. The `luaO_*` functions support number parsing, formatting, and debugging infrastructure.

## External Dependencies

- **`<stdarg.h>`**: For `va_list` in format functions
- **`llimits.h`**: Basic types (`lu_byte`, `l_mem`), type limits, cast macros, assertion macros (`check_exp`, `lua_longassert`)
- **`lua.h`**: Type constants (`LUA_TNIL`, `LUA_TFUNCTION`, etc.), version info, public API declarations
- **Defined elsewhere**: 
  - `lua_State`: Opaque Lua execution state (declared in `lua.h`)
  - `G(L)`: Macro to access global state from `lua_State` (defined elsewhere, likely `lstate.h`)
  - `isdead()`: Check if GC object is marked for deletion (defined in GC module, likely `lgc.h`)
