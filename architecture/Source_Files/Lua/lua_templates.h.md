# Source_Files/Lua/lua_templates.h

## File Purpose
Provides C++ template classes for exposing C++ game objects to Lua as userdata with get/set accessors, enum mnemonics, containers, and object-to-userdata mapping. Core Lua/C interface bridge for the Aleph One game engine.

## Core Responsibilities
- **L_Class**: Template wrapper binding arbitrary C++ objects as Lua userdata with indexed access and method dispatch
- **L_Enum**: Extends L_Class; adds string-to-number mnemonic lookup and equality operators
- **L_Container**: Makes collections iterable in Lua with numeric and string indexing
- **L_EnumContainer**: Container supporting both numeric and mnemonic string lookups
- **L_ObjectClass**: Stores actual C++ objects mapped by numeric index; auto-generates indices
- **Registry management**: Helper functions for script paths and engine state queries

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| L_Class | template class | Generic Lua userdata wrapper with index tracking and method dispatch |
| L_Enum | template class | Enum class with mnemonic name resolution and equality |
| L_LazyEnum | template class | Eager-error enum variant |
| L_Container | template class | Iterable container with __len, __call, __index |
| L_EnumContainer | template class | Container with string mnemonic lookup |
| L_ObjectClass | template class | Maps indices to held C++ objects |
| UserdataBlock | nested struct | Userdata layout: [L_Class* basePtr][aligned instance buffer] |
| ValidRange, always_valid, object_valid | functors | Pluggable validity checkers via std::function |
| lang_def | struct (from lua_mnemonics.h) | {const char* name; int32 value} mnemonic pair |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Valid | std::function<bool(index_t)> | static per L_Class/L_Enum template | Validates whether an index is currently alive |
| Length | std::function<T::index_type()> | static per L_Container template | Returns container iteration bound |
| _objects | std::map<index_t, object_t> | static per L_ObjectClass template | Holds C++ objects keyed by index |

## Key Functions / Methods

### L_Class<name, index_t>::Register
- Signature: `static void Register(lua_State *L, const luaL_Reg get[]=0, const luaL_Reg set[]=0, const luaL_Reg metatable[]=0)`
- Purpose: Register a class in Lua registry with metatable, get/set method tables, and instance cache
- Inputs: Lua state; optional get/set method arrays; optional extra metatable methods
- Outputs/Return: void; modifies Lua registry
- Side effects: Creates registry metatable, method tables, instance table; registers global `is_<name>` function
- Calls: luaL_newmetatable, lua_pushcfunction, lua_setfield, luaL_setfuncs, lua_settable, lua_setglobal
- Notes: Lua 5.1 compatibility layer for metatable string mapping; uses light userdata keys for internal tables

### L_Class<name, index_t>::Push<instance_t>
- Signature: `static instance_t *Push(lua_State *L, index_t index)`
- Purpose: Push cached or newly created userdata for index onto stack; returns instance pointer
- Inputs: index to wrap or retrieve
- Outputs: Instance pointer (nullptr if invalid); userdata pushed on Lua stack
- Side effects: Cache miss triggers NewInstance; updates instance table
- Calls: Valid(), NewInstance, lua_newuserdata, lua_gettable, lua_settable, lua_remove, lua_pushnumber, lua_isnil, lua_pop
- Notes: Returns nil on invalid index; idempotentΓÇöcalling twice with same index returns same userdata

### L_Class<name, index_t>::NewInstance<instance_t>
- Signature: `static instance_t *NewInstance(lua_State *L, index_t index)`
- Purpose: Allocate userdata block and construct instance in-place with placement new
- Inputs: index for instance
- Outputs: Pointer to constructed instance; userdata on stack with metatable set
- Side effects: Allocates and constructs; stores base pointer at userdata offset 0
- Calls: lua_newuserdata (invokes placement-new), luaL_getmetatable, lua_setmetatable
- Notes: UserdataBlock stores L_Class* then aligned instance buffer; enables Instance() to be non-template

### L_Class<name, index_t>::Instance
- Signature: `static L_Class *Instance(lua_State *L, int index)`
- Purpose: Extract base class pointer from userdata
- Inputs: Stack index of userdata
- Outputs: Dereferenced base pointer or nullptr
- Calls: lua_touserdata, static_cast (dereference at offset 0)
- Notes: Safe for derived classes; enables unified type-independent extraction

### L_Class<name, index_t>::Index / Is / Invalidate
- **Index**: `static index_t Index(lua_State *L, int index)` ΓÇö extract m_index; error if invalid userdata
- **Is**: `static bool Is(lua_State *L, int index)` ΓÇö check metatable equality; return false if not userdata
- **Invalidate**: `static void Invalidate(lua_State *L, index_t index)` ΓÇö remove from instance table and custom fields

### L_Class<name, index_t>::_get / _set (Lua metamethods)
- **_get** (__index): Dispatch to get method function or fetch custom field (if name[0]=='_'); pcall with error reporting
- **_set** (__newindex): Dispatch to set method function or store custom field; pcall with error reporting
- Custom fields stored in persistent table by (name, index) key for underscore-prefixed properties

### L_Class<name, index_t>::_index / _is / _new / _tostring (Lua metamethods)
- **_index**: Return numeric index
- **_is**: Return Is() boolean
- **_new**: Metatable __new; create instance from number via Push
- **_tostring**: Format "ClassName index" via ostringstream

### L_Enum<name, index_t>::Register
- Signature: `static void Register(lua_State *L, const luaL_Reg get[]=0, const luaL_Reg set[]=0, const luaL_Reg metatable[]=0, const lang_def mnemonics[]=0)`
- Purpose: Register enum class with mnemonic table and equality operator
- Side effects: Calls L_Class::Register; adds __eq and __tostring; builds bidirectional mnemonic table in registry
- Calls: L_Class<>::Register, lua_pushcfunction, lua_setfield, luaL_setfuncs, luaL_getmetatable, lua_newtable, lua_settable, lua_pop
- Notes: Mnemonics array null-terminated; table stores both stringΓåÆint and intΓåÆstring mappings

### L_Enum<name, index_t>::ToIndex
- Signature: `static index_t ToIndex(lua_State *L, int index)`
- Purpose: Convert Lua value (number, string, or instance) to numeric index via _lookup
- Outputs: Numeric index or error via luaL_error
- Inputs: Can be number, mnemonic string, or enum instance
- Calls: _lookup, luaL_error
- Notes: Tries in order: instance check, numeric, string mnemonic; L_LazyEnum specializes this for stricter validation

### L_Enum<name, index_t>::_equals / _get_mnemonic / _set_mnemonic
- **_equals** (__eq): Compare two enum values via _lookup; return equality
- **_get_mnemonic**: Lookup mnemonic string from number in table; return string or nil
- **_set_mnemonic**: Update mnemonic entry (bidirectional)

### L_Container<name, T>::Register
- Signature: `static void Register(lua_State *L, const luaL_Reg get[]=0, const luaL_Reg set[]=0, const luaL_Reg metatable[]=0)`
- Purpose: Register iterable container; set __index, __call, __len; register global variable
- Side effects: Calls L_Class::Register; overrides metatable; registers container as global
- Calls: L_Class<>::Register, lua_pushcfunction, lua_setfield, L_Class<>::Push, lua_setglobal
- Notes: __index delegates to _get_container; __call returns iterator closure; __len returns Length()

### L_Container<name, T>::_iterator
- Purpose: C closure iterating from 0 to Length(), yielding valid instances
- Side effects: Maintains current index in upvalue; increments on each valid item
- Calls: T::Valid, T::Push, lua_pushnumber, lua_replace, lua_upvalueindex
- Notes: Skips invalid indices; returns nil when exhausted

### L_EnumContainer<name, T>::Register / _get_enumcontainer
- **Register**: Calls L_Container::Register; overrides __index to _get_enumcontainer
- **_get_enumcontainer**: Try numeric index (T::Push), else string mnemonic (T::PushMnemonicTable), else methods
- Calls: T::Valid, T::PushMnemonicTable, lua_istable, lua_gettable, lua_isnumber, lua_pop, L_Container<>::_get_container

### L_ObjectClass<name, object_t, index_t>::Push<instance_t>
- Signature: `static instance_t *Push(lua_State *L, object_t object)`
- Purpose: Allocate unique index, store object, create and return userdata wrapper
- Inputs: C++ object to wrap
- Outputs: Pointer to instance; userdata on stack
- Side effects: Finds unused index via linear search; stores in _objects map
- Calls: _objects.find/insert, L_Class<>::NewInstance
- Notes: Auto-generates index; Used for non-indexed C++ objects

### L_ObjectClass<name, object_t, index_t>::Object / ObjectAtIndex / Invalidate
- **Object**: Extract object from Lua userdata via Index lookup
- **ObjectAtIndex**: Direct map lookup; error if not found
- **Invalidate**: Erase from _objects map; subsequent Valid() returns false
- Calls: L_Class<>::Index, _objects.find, luaL_typerror, _objects.erase

## Control Flow Notes
**Startup**: Register() sets up metatable and method tables in Lua registry.  
**Instance creation**: Script calls `ClassName(index)` ΓåÆ `__new` ΓåÆ Push() ΓåÆ caches userdata.  
**Property access**: `obj.prop` ΓåÆ `__index` ΓåÆ _get() ΓåÆ dispatches to get method function (pcall'd).  
**Iteration**: `for x in container()` ΓåÆ __call ΓåÆ _iterator closure advances through 0..Length()-1, yielding valid items.  
**Enum lookup**: ToIndex(lua_value) tries instance, numeric, string mnemonic in order.  
**Cleanup**: Engine calls Invalidate(index) ΓåÆ removes from cache; subsequent Push returns nil.

## External Dependencies
- **lua.h, lauxlib.h, lualib.h**: Lua 5.2 C API (lua_State, lua_CFunction, luaL_Reg, etc.)
- **cseries.h**: Engine type definitions (int16, uint16, uint8, OSErr, etc.)
- **lua_script.h**: Game-facing Lua declarations (L_Error, L_Call_Init, L_Persistent_Table_Key())
- **lua_mnemonics.h**: lang_def struct; extern mnemonic arrays (Lua_ItemType_Mnemonics, etc.)
- **<sstream>, <map>, <new>, <functional>**: C++ standard library
- **luaL_typerror()**: Helper defined in this file; wraps lua_pushfstring + luaL_argerror
- **Defined elsewhere (extern)**: L_Set_Search_Path(), L_Get_Search_Path(), L_Get_Proper_Item_Accounting(), L_Set_Proper_Item_Accounting(), L_Get_Nonlocal_Overlays(), L_Set_Nonlocal_Overlays(), L_Persistent_Table_Key()
