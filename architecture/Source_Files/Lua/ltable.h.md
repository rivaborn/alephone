# Source_Files/Lua/ltable.h

## File Purpose
Public header for Lua table (hash table) operations. Declares the interface for creating, accessing, modifying, and destroying tables, and provides convenience macros for accessing table node fields.

## Core Responsibilities
- Define macros for safe access to table node components (`gnode`, `gkey`, `gval`, `gnext`)
- Declare public table management functions (creation, deletion, resizing)
- Declare table lookup and insertion functions (integer keys, string keys, generic TValue keys)
- Declare table iteration and length queries
- Invalidate tagmethod cache when table structure changes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Table` | struct (from lobject.h) | Main table structure; contains array part, hash node part, and metadata |
| `Node` | struct (from lobject.h) | Hash table node containing a key-value pair and next pointer for collision chaining |
| `TKey` | union (from lobject.h) | Key variant; either a TValue with collision chain link, or standalone TValue |
| `TValue` | struct (from lobject.h) | Tagged value; generic Lua value representation |

## Global / File-Static State
None.

## Key Functions / Methods

### luaH_getint
- Signature: `const TValue *luaH_getint (Table *t, int key)`
- Purpose: Retrieve value associated with an integer key in the table's array part
- Inputs: table pointer, integer key
- Outputs/Return: pointer to TValue (const); NULL-like if not found
- Side effects: None (read-only)
- Calls: Defined elsewhere (ltable.c)

### luaH_setint
- Signature: `void luaH_setint (lua_State *L, Table *t, int key, TValue *value)`
- Purpose: Set or update value for an integer key
- Inputs: lua_State, table, integer key, value pointer
- Outputs/Return: None
- Side effects: May trigger table resize if array part needs expansion; may allocate memory
- Calls: Defined elsewhere
- Notes: Takes explicit lua_State for memory allocation context

### luaH_getstr
- Signature: `const TValue *luaH_getstr (Table *t, TString *key)`
- Purpose: Retrieve value for a string key (optimized path)
- Inputs: table, string key
- Outputs/Return: pointer to TValue (const)
- Side effects: None (read-only)
- Calls: Defined elsewhere

### luaH_get
- Signature: `const TValue *luaH_get (Table *t, const TValue *key)`
- Purpose: Generic lookup for any TValue key type
- Inputs: table, key pointer
- Outputs/Return: pointer to TValue (const)
- Side effects: None (read-only)
- Calls: Defined elsewhere

### luaH_newkey
- Signature: `TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key)`
- Purpose: Insert a new key into the table and return mutable reference to its value slot
- Inputs: lua_State, table, key
- Outputs/Return: mutable pointer to value (for caller to populate)
- Side effects: May resize table; may trigger GC; allocates new node
- Calls: Defined elsewhere

### luaH_set
- Signature: `TValue *luaH_set (lua_State *L, Table *t, const TValue *key)`
- Purpose: Get or create value slot for key; returns mutable reference
- Inputs: lua_State, table, key
- Outputs/Return: mutable pointer to value slot
- Side effects: May allocate if key is new; may resize
- Calls: Defined elsewhere

### luaH_new
- Signature: `Table *luaH_new (lua_State *L)`
- Purpose: Allocate and initialize a new empty table
- Inputs: lua_State
- Outputs/Return: pointer to new Table
- Side effects: Allocates memory; initializes table to empty state
- Calls: Defined elsewhere

### luaH_resize
- Signature: `void luaH_resize (lua_State *L, Table *t, int nasize, int nhsize)`
- Purpose: Resize both array and hash parts of a table
- Inputs: lua_State, table, new array size, new hash size (power of 2)
- Outputs/Return: None
- Side effects: Reallocates table storage; rehashes all entries; may trigger GC
- Calls: Defined elsewhere
- Notes: Critical for performance; called when table load factor exceeds threshold

### luaH_resizearray
- Signature: `void luaH_resizearray (lua_State *L, Table *t, int nasize)`
- Purpose: Resize only the array part (contiguous integer indices)
- Inputs: lua_State, table, new array size
- Outputs/Return: None
- Side effects: Reallocates array storage; may migrate entries between array and hash parts
- Calls: Defined elsewhere

### luaH_free
- Signature: `void luaH_free (lua_State *L, Table *t)`
- Purpose: Deallocate table and all its memory
- Inputs: lua_State, table
- Outputs/Return: None
- Side effects: Frees all heap memory associated with table; invalidates all references
- Calls: Defined elsewhere

### luaH_next
- Signature: `int luaH_next (lua_State *L, Table *t, StkId key)`
- Purpose: Iterate to next key-value pair in table traversal
- Inputs: lua_State, table, current key position (stack index)
- Outputs/Return: 1 if next pair exists (key/value on stack), 0 if end of table
- Side effects: Modifies stack (pushes/updates key and value)
- Calls: Defined elsewhere
- Notes: Used by `pairs()` iteration; traverses array part then hash part

### luaH_getn
- Signature: `int luaH_getn (Table *t)`
- Purpose: Compute table length (for `#` operator)
- Inputs: table
- Outputs/Return: integer length
- Side effects: None (read-only)
- Calls: Defined elsewhere
- Notes: Counts array part up to first nil; used for sequence semantics

## Helper Macros
- `gnode(t,i)` ΓÇö Get pointer to node at index i
- `gkey(n)` ΓÇö Get key pointer from node
- `gval(n)` ΓÇö Get value pointer from node
- `gnext(n)` ΓÇö Get next pointer (for collision chain)
- `invalidateTMcache(t)` ΓÇö Clear flags; invalidates tagmethod cache when table structure changes

## Control Flow Notes
This is an interface header; actual control flow is in **ltable.c**. Tables are fundamental to Lua runtime:
- **Initialization**: `luaH_new` called when Lua code creates `{}`
- **Runtime**: Getters called during table access; setters during assignment
- **Garbage collection**: `luaH_free` called during GC sweep phase
- **Iteration**: `luaH_next` called by Lua's `pairs()` iterator
- **Resizing**: Resizing functions called transparently when load factors trigger resize heuristics

## External Dependencies
- **Includes**: `lobject.h` (provides TValue, Table, Node, TKey, StkId definitions)
- **Transitively**: `lua.h` (Lua core API types), `llimits.h` (type limits)
- **Used symbols** (defined elsewhere): `lua_State`, `TValue`, `Table`, `Node`, `TString`, `StkId`, `LUAI_FUNC` macro
