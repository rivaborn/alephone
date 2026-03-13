# Source_Files/Lua/lstring.h

## File Purpose
Manages Lua's string interning system and memory allocation for strings and userdata. This header defines the public interface for creating, hashing, comparing, and resizing the global string table, which ensures all identical strings share the same memory location.

## Core Responsibilities
- String creation and interning (ensuring string uniqueness via hash table)
- Memory size calculation for TString and Udata structures
- String hashing and equality comparison for short and long strings
- String table resizing and management
- Userdata allocation and management
- Marking strings as fixed (non-collectable)
- Reserved word detection for lexer integration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TString | union (defined in lobject.h) | Collectable string structure with header, hash, and length |
| Udata | union (defined in lobject.h) | Collectable userdata structure with metatable and environment |

## Global / File-Static State
None.

## Key Functions / Methods

### luaS_hash
- **Signature:** `unsigned int luaS_hash (const char *str, size_t l, unsigned int seed)`
- **Purpose:** Compute hash value for a string using seed
- **Inputs:** C string pointer, string length, randomized seed
- **Outputs/Return:** Computed hash value
- **Side effects:** None (pure function)
- **Calls:** None visible
- **Notes:** Uses seed from global state to resist hash collision attacks

### luaS_eqstr
- **Signature:** `int luaS_eqstr (TString *a, TString *b)`
- **Purpose:** Equality comparison for both short and long strings
- **Inputs:** Two TString pointers
- **Outputs/Return:** Boolean (1 if equal, 0 otherwise)
- **Side effects:** None
- **Calls:** `luaS_eqlngstr` (for long strings)
- **Notes:** Delegates to `luaS_eqlngstr` for non-short strings; short strings can use pointer equality since they're interned

### luaS_eqlngstr
- **Signature:** `int luaS_eqlngstr (TString *a, TString *b)`
- **Purpose:** Equality comparison for long strings (non-interned)
- **Inputs:** Two TString pointers
- **Outputs/Return:** Boolean
- **Side effects:** None
- **Calls:** None visible
- **Notes:** Compares length and content; long strings may not be interned

### luaS_newlstr
- **Signature:** `TString *luaS_newlstr (lua_State *L, const char *str, size_t l)`
- **Purpose:** Create or retrieve interned string of specified length
- **Inputs:** Lua state, string buffer, string length
- **Outputs/Return:** Pointer to interned TString
- **Side effects:** Allocates memory, updates global string hash table, may trigger GC
- **Calls:** Indirectly calls GC and memory allocation
- **Notes:** Core string creation; performs interning lookup first

### luaS_new
- **Signature:** `TString *luaS_new (lua_State *L, const char *str)`
- **Purpose:** Create interned string from C string (NUL-terminated)
- **Inputs:** Lua state, C string
- **Outputs/Return:** Pointer to interned TString
- **Side effects:** Allocates memory, updates global string hash table
- **Calls:** `luaS_newlstr` (wrapper)
- **Notes:** Convenience wrapper; computes length via strlen

### luaS_resize
- **Signature:** `void luaS_resize (lua_State *L, int newsize)`
- **Purpose:** Resize the global string hash table
- **Inputs:** Lua state, new hash table size
- **Outputs/Return:** None
- **Side effects:** Reorganizes string hash table in global state, rehashes all strings
- **Calls:** Memory allocation, GC if necessary
- **Notes:** Called when hash table load factor exceeds threshold

### luaS_newudata
- **Signature:** `Udata *luaS_newudata (lua_State *L, size_t s, Table *e)`
- **Purpose:** Allocate a new userdata object with associated environment
- **Inputs:** Lua state, userdata size in bytes, environment table
- **Outputs/Return:** Pointer to newly allocated Udata
- **Side effects:** Allocates memory from GC system, may trigger GC
- **Calls:** GC object allocation
- **Notes:** Userdata is collectable; environment allows C data to reference Lua tables

## Key Macros
- **sizestring(s)**, **sizeudata(u)**: Calculate allocation sizes including header
- **luaS_newliteral(L, s)**: Create string literal from C string constant (computed at compile-time)
- **luaS_fix(s)**: Mark string as FIXED to prevent GC collection
- **isreserved(s)**: Check if short string is a reserved word (used by lexer)
- **eqshrstr(a,b)**: Fast equality for short strings via pointer comparison (works because strings are interned)

## Control Flow Notes
String creation is triggered during:
- Lua script parsing (lexer creates identifier and literal strings)
- Runtime string operations (concatenation, string.sub, etc.)
- C API calls (`lua_pushstring`)

String hash table resizing is part of the dynamic memory management cycle. Fixed strings (reserved words, commonly-used strings) are protected from garbage collection to reduce allocation overhead.

## External Dependencies
- **lgc.h**: Garbage collection marking, `FIXEDBIT` and bit manipulation macros
- **lobject.h**: `TString`, `Udata`, `GCObject` type definitions; type tag constants (`LUA_TSHRSTR`, `LUA_TLNGSTR`)
- **lstate.h**: `lua_State`, `global_State` for accessing `strt` (string hash table) and memory allocator
