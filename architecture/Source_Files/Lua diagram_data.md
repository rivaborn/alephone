# Source_Files/Lua/language_definition.h
## File Purpose
A compile-time constant definition file that maps human-readable symbolic names to internal numeric codes for game engine entities. It serves as the "language definition" for script writers, enabling them to reference items, monsters, sounds, damage types, and game mechanics by mnemonic names rather than raw hex values.

## Core Responsibilities
- Define symbolic constants for weapons, items, and powerups with hexadecimal IDs
- Map monster/creature type mnemonics to engine-internal codes
- Define damage source type constants for hit/death attribution
- Enumerate monster behavioral states and action codes
- Map audio effect IDs to sound event names
- Define projectile/attack type mnemonics
- Categorize level geometry (polygon) types and properties
- Enumerate game mode constants (deathmatch, CTF, king-of-the-hill, etc.)
- Provide backwards-compatible naming aliases (underscore-prefixed and non-prefixed variants)

## External Dependencies
- **Undefined symbol references** (netscript section): `_panel_is_oxygen_refuel`, `_panel_is_shield_refuel`, `_panel_is_double_shield_refuel`, `_panel_is_triple_shield_refuel`, `_weapon_fist`, `_weapon_pistol`, `_weapon_plasma_pistol`, `_weapon_shotgun`, `_weapon_assault_rifle`, `_weapon_smg`, `_weapon_flamethrower`, `_weapon_missile_launcher`, `_weapon_alien_shotgun`, `_weapon_ball`, `_game_of_most_points`, `_game_of_most_time`, `_game_of_least_points`, `_game_of_least_time` ΓÇö defined elsewhere.


# Source_Files/Lua/lapi.h
## File Purpose
Header providing auxiliary macros for Lua C API stack and call frame management. Implements safety checks and state adjustments for the public API layer.

## Core Responsibilities
- Provide stack pointer management with bounds checking
- Validate call frame resources before API operations
- Adjust return value handling for multi-return functions
- Enforce API preconditions and invariants

## External Dependencies
- `llimits.h`: provides `api_check` macro and type limits
- `lstate.h`: provides `lua_State` structure and `CallInfo` definition
- Implicitly depends on: `lua_State::top`, `lua_State::ci`, `CallInfo::top`, `CallInfo::func`

# Source_Files/Lua/lauxlib.h
## File Purpose
Header file declaring the Lua Auxiliary Library, which provides convenience functions and macros for C code building Lua libraries and interacting with the Lua stack. Includes argument validation, type conversion, buffer management, file handling, and library registration utilities.

## Core Responsibilities
- Declare auxiliary functions for parameter validation and type checking
- Provide type conversion utilities (strings, numbers, integers, userdata)
- Support metatable manipulation and userdata handling
- Implement string buffer building for efficient concatenation
- Manage file handles for Lua I/O operations
- Support library and module registration mechanisms
- Define convenience macros for common stack operations

## External Dependencies
- **`lua.h`** ΓÇö Core Lua API (types: `lua_State`, `lua_CFunction`, `lua_Number`, `lua_Integer`; functions: `lua_createtable`, `lua_pcall`, `lua_getfield`, etc.)
- **`<stddef.h>`** ΓÇö Standard C types (`size_t`, `NULL`)
- **`<stdio.h>`** ΓÇö File I/O (`FILE`)
- **`luaconf.h`** (via `lua.h`) ΓÇö Lua configuration constants and type definitions

**Macros/Constants from lua.h used in lauxlib.h:**
- Error codes: `LUA_ERRERR`, `LUA_ERRSYNTAX`, `LUA_ERRFILE`
- Stack indices: `LUA_REGISTRYINDEX`
- Return values: `LUA_OK`, `LUA_MULTRET`
- Type constants: `LUA_TSTRING`, `LUA_TNUMBER`, etc.
- Version: `LUA_VERSION_NUM`

# Source_Files/Lua/lcode.h
## File Purpose
Code generator interface for the Lua virtual machine compilation pipeline. Translates parsed expressions, statements, and control flow into bytecode instructions, managing register allocation and code patching for the compiler backend.

## Core Responsibilities
- Bytecode instruction generation (ABC, ABx, Ax instruction formats)
- Expression-to-bytecode conversion (registers, constants, upvalues)
- Constant pooling and interning (numbers, strings)
- Register allocation and stack management
- Jump/patch list handling for control flow (loops, conditionals)
- Unary and binary operator code generation
- Table and variable operations (assignment, indexing, method calls)

## External Dependencies
- **llex.h**: Token, LexState (lexical analysis state, used indirectly via lparser.h)
- **lobject.h**: TString, Proto, Closure, expdesc-related types, lua_Number, lua_State
- **lopcodes.h**: OpCode enum, instruction encoding macros (MAXARG_*, CREATE_*, GETARG_*, SETARG_*)
- **lparser.h**: expdesc, FuncState, BinOpr, UnOpr context definitions

# Source_Files/Lua/lctype.h
## File Purpose
Provides Lua-specific character classification and case-conversion macros. Optimizes for Lua's needs by differing from standard C ctype.h, supporting both ASCII-table and standard-ctype implementations based on platform.

## Core Responsibilities
- Define character property bits (alphabetic, digit, printable, space, hex-digit)
- Provide character classification macros that test these properties on input characters
- Support both custom lookup-table and standard C ctype implementations via compile-time switch
- Include case-conversion macro for alphabetic characters
- Accommodate EOZ (end-of-zone, index -1) by off-by-one indexing into the classification table

## External Dependencies
- **Notable includes:** `lua.h`, `llimits.h`, conditionally `<limits.h>`, conditionally `<ctype.h>`
- **External symbols used:**
  - `luai_ctype_[UCHAR_MAX + 2]` ΓÇô defined elsewhere (llex.c or similar)
  - `lu_byte` ΓÇô typedef from llimits.h
  - `UCHAR_MAX` ΓÇô from standard `<limits.h>`

# Source_Files/Lua/ldebug.h
## File Purpose
Header for Lua's debug interface auxiliary functions. Defines debugging macros and declares error-reporting functions used to propagate runtime errors (type errors, arithmetic errors, etc.) throughout the Lua VM.

## Core Responsibilities
- Define debugging utility macros (program counter calculation, line info lookup, hook management, call info extraction)
- Declare error-reporting functions for runtime violations
- Provide non-returning error handlers for type, arithmetic, ordering, and concatenation failures

## External Dependencies
- **Includes**: `lstate.h` (for `lua_State`, `CallInfo`, `StkId`, `TValue` types)
- **Macros used**: `LUAI_FUNC`, `l_noret`, `cast`, `clLvalue`
- **Types defined elsewhere**: `lua_State`, `TValue`, `StkId`, `CallInfo`, `Proto`

# Source_Files/Lua/ldo.h
## File Purpose
Defines the stack management and protected execution framework for Lua's virtual machine. Provides macros and function declarations for stack bounds checking, function call setup, exception handling, and debugging hooks.

## Core Responsibilities
- Stack overflow prevention and dynamic resizing
- Protected execution of Lua code and C functions with exception handling
- Function call protocol (pre-call setup, post-call cleanup)
- Debug hook invocation
- Parser invocation with error containment
- Stack pointer serialization/restoration for relocatable stacks

## External Dependencies
- **lobject.h** ΓÇö `TValue`, `StkId`, `Closure`, `Proto` type definitions
- **lstate.h** ΓÇö `lua_State`, `global_State`, `CallInfo` structures; `G(L)` macro
- **lzio.h** ΓÇö `ZIO` stream type for parser input
- **lua.h** ΓÇö Public Lua API types and error codes (referenced but not included in this snippet)

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

## External Dependencies
- **`lobject.h`:** Provides Proto, Closure, CClosure, LClosure, UpVal, TValue, Upvaldesc, LocVar types.
- **`lua.h`:** Provides lua_State, lua_CFunction, and core type definitions.
- **`LUAI_FUNC` macro:** Used for extern function declarations; allows platform-specific visibility/calling conventions.

**Notes:** 
- Size macros `sizeCclosure` and `sizeLclosure` use pointer arithmetic to calculate variable-size allocations.
- All functions operate on Lua state and assume the GC system manages lifetime.

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

## External Dependencies
- **lobject.h**: `GCObject`, `GCheader`, `TString`, `Udata`, `Proto`, `Closure`, `UpVal`, `Table`, `iscollectable()`, `gcvalue()`
- **lstate.h**: `lua_State`, `global_State`, `G()` macro for accessing global state from thread

# Source_Files/Lua/llex.h
## File Purpose
Defines the lexical analyzer interface for Lua. Declares token types (reserved words and operators), the LexState structure that maintains lexer state during tokenization, and the public API for lexical analysis and token manipulation.

## Core Responsibilities
- Define token type constants for all Lua reserved words and operators (RESERVED enum)
- Maintain lexer state including current/lookahead tokens, input position, and line tracking (LexState)
- Initialize and configure lexer state for input streams
- Advance token stream and provide lookahead capability
- Intern strings to ensure single-copy representation
- Report syntax errors with source location information
- Convert token codes to human-readable strings for diagnostics

## External Dependencies
- **`lobject.h`**: TString (interned strings), lua_Number (numeric values), type system macros
- **`lzio.h`**: ZIO (buffered input stream), Mbuffer (growable token buffer)
- **`lua.h`** (implicit): lua_State, LUAI_FUNC (visibility macro), lua_Reader callback type

# Source_Files/Lua/llimits.h
## File Purpose
Internal Lua header defining platform-specific limits, portable type aliases, and debug/safety macros. Handles compiler-specific features (MSVC, GCC) and supports multiple number-to-integer conversion strategies.

## Core Responsibilities
- Define portable integer/memory types (lu_int32, lu_mem, lu_byte, Instruction)
- Enforce safe limits (MAX_SIZET, MAX_LUMEM, MAX_INT, MAXSTACK, MAXUPVAL)
- Provide type-casting macros with safety checks
- Supply debug assertions (lua_assert, api_check, check_exp)
- Implement numberΓåöinteger conversions with platform-specific optimizations (IEEE754, MSVC assembler)
- Provide thread locking/yielding hooks
- Define user-state lifecycle callbacks for threads

## External Dependencies
- **Standard C:** `<limits.h>` (INT_MAX, UCHAR_MAX), `<stddef.h>` (size_t), `<float.h>` (DBL_MAX_EXP, conditionally), `<math.h>` (frexp, floor, conditionally), `<assert.h>` (conditionally)
- **Lua internals:** `lua.h` (defines lua_Number, lua_Integer, lua_Unsigned); `luaconf.h` (user customization macros: LUAI_UMEM, LUAI_MEM, LUAI_MAXCCALLS, LUA_IEEE754TRICK, MS_ASMTRICK, etc.)
- **Referenced but not defined here:** luaD_reallocstack, luaC_fullgc, G() macro (defined elsewhere in Lua internals)

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

## External Dependencies
- **Includes:** `<stddef.h>` (size_t), `llimits.h` (MAX_SIZET, l_noret, cast macro), `lua.h` (lua_State)
- **External symbols used:** `lua_State` (opaque handle), `MAX_SIZET` (size limit constant)

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

## External Dependencies

- **`<stdarg.h>`**: For `va_list` in format functions
- **`llimits.h`**: Basic types (`lu_byte`, `l_mem`), type limits, cast macros, assertion macros (`check_exp`, `lua_longassert`)
- **`lua.h`**: Type constants (`LUA_TNIL`, `LUA_TFUNCTION`, etc.), version info, public API declarations
- **Defined elsewhere**: 
  - `lua_State`: Opaque Lua execution state (declared in `lua.h`)
  - `G(L)`: Macro to access global state from `lua_State` (defined elsewhere, likely `lstate.h`)
  - `isdead()`: Check if GC object is marked for deletion (defined in GC module, likely `lgc.h`)

# Source_Files/Lua/lopcodes.h
## File Purpose
Defines the instruction format and complete opcode set for the Lua virtual machine. Provides macros for encoding, decoding, and manipulating 32-bit VM instructions with opcode and argument fields. Specifies all 38 opcodes and their argument modes.

## Core Responsibilities
- Define instruction bit layout (6-bit opcode + A/B/C/Bx/sBx/Ax argument fields)
- Encode/decode instruction fields via bit manipulation macros
- Define all Lua VM opcodes (arithmetic, table, control flow, function calls, etc.)
- Specify argument constraints (size limits, signed vs. unsigned, register vs. constant encoding)
- Provide instruction construction and field extraction helpers
- Declare opcode metadata tables and name strings (extern)

## External Dependencies
- **Includes**: `llimits.h` (for `Instruction` type = `lu_int32`, `lu_byte`, size/alignment definitions)
- **Defined elsewhere**: 
  - `luaP_opmodes[]` ΓÇö typically in `lopcodes.c`
  - `luaP_opnames[]` ΓÇö typically in `lopcodes.c`
  - `LUAI_BITSINT`, `LUAI_DDEC`, `MAX_INT` ΓÇö from `llimits.h` / config


# Source_Files/Lua/lparser.h
## File Purpose
Header for the Lua parser, defining data structures used to parse source code and generate bytecode. Establishes the interface between the lexer and code generator during compilation.

## Core Responsibilities
- Define expression descriptor types (`expkind`, `expdesc`) to represent values and control flow during parsing
- Track active local variables and their stack positions (`Vardesc`, `Dyndata`)
- Manage label and goto statements for code generation (`Labeldesc`, `Labellist`)
- Provide per-function compilation state (`FuncState`) for bytecode generation
- Export the main parser entry point (`luaY_parser`)

## External Dependencies
- **llimits.h**: Type definitions (`lu_byte`, `lu_mem`, basic macros)
- **lobject.h**: Lua object structures (`Proto`, `Table`, `TString`, `Closure`, `UpVal`, instruction type)
- **lzio.h**: Buffered I/O (`ZIO` stream, `Mbuffer`)

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

## External Dependencies
- **Includes**: `lua.h` (base API), `lobject.h` (type definitions and GC header), `ltm.h` (tag methods/metatables), `lzio.h` (buffered I/O)
- **Defined elsewhere**:
  - `lua_longjmp` (in `ldo.c`) ΓÇö exception handling jump buffer
  - `GCObject`, `TValue`, `TString`, `Udata`, `Closure`, `UpVal`, `Proto`, `Table` (in `lobject.h`)
  - `Mbuffer` (in `lzio.h`) ΓÇö temporary buffer for string concatenation
  - GC internals (`luaC_*` functions) ΓÇö garbage collection implementation in `lgc.c`

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

## External Dependencies
- **lgc.h**: Garbage collection marking, `FIXEDBIT` and bit manipulation macros
- **lobject.h**: `TString`, `Udata`, `GCObject` type definitions; type tag constants (`LUA_TSHRSTR`, `LUA_TLNGSTR`)
- **lstate.h**: `lua_State`, `global_State` for accessing `strt` (string hash table) and memory allocator

# Source_Files/Lua/ltable.h
## File Purpose
Public header for Lua table (hash table) operations. Declares the interface for creating, accessing, modifying, and destroying tables, and provides convenience macros for accessing table node fields.

## Core Responsibilities
- Define macros for safe access to table node components (`gnode`, `gkey`, `gval`, `gnext`)
- Declare public table management functions (creation, deletion, resizing)
- Declare table lookup and insertion functions (integer keys, string keys, generic TValue keys)
- Declare table iteration and length queries
- Invalidate tagmethod cache when table structure changes

## External Dependencies
- **Includes**: `lobject.h` (provides TValue, Table, Node, TKey, StkId definitions)
- **Transitively**: `lua.h` (Lua core API types), `llimits.h` (type limits)
- **Used symbols** (defined elsewhere): `lua_State`, `TValue`, `Table`, `Node`, `TString`, `StkId`, `LUAI_FUNC` macro

# Source_Files/Lua/ltm.h
## File Purpose
Defines Lua's tag methods (metamethods) systemΓÇöthe mechanism for customizing behavior of operations on values. Provides the enumeration of all tag method types, fast lookup macros, and function declarations for querying metamethods from tables and objects.

## Core Responsibilities
- Define `TMS` enum enumerating all tag method types (INDEX, NEWINDEX, GC, ADD, SUB, CONCAT, etc.)
- Provide macros for fast tag method lookup with cache checking
- Declare functions to retrieve tag methods by event type
- Supply type name lookup macros for debugging/introspection
- Initialize tag method infrastructure at engine startup

## External Dependencies
- **Includes:** `lobject.h` (TValue, Table, TString types; LUA_TOTALTAGS constant)
- **External symbols:** `luaT_gettm`, `luaT_gettmbyobj`, `luaT_init` (implementations in ltm.c); `G()` macro (global state accessor); type macros from lobject.h; `LUAI_FUNC`, `LUAI_DDEC` (visibility macros from llimits.h)

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

## External Dependencies

- **Standard C headers:** `<stdarg.h>` (va_list), `<stddef.h>` (size_t, ptrdiff_t)
- **Local configuration:** `"luaconf.h"` ΓÇö platform-specific settings, type definitions (LUA_NUMBER, LUA_INTEGER), API export macros (LUA_API, LUALIB_API), allocator signatures
- **Optional user header:** `LUA_USER_H` (if defined at compile-time) ΓÇö allows embedding custom extensions
- **Defined elsewhere:** All function implementations (state.c, stack.c, vm.c, etc.), `lua_ident` string, `lua_Debug` struct full definition

# Source_Files/Lua/lua_ephemera.cpp
## File Purpose
Implements Lua bindings for ephemeraΓÇötransient visual objects in the game world. Provides Lua scripts with the ability to query, create, and modify ephemera properties (position, appearance, animation behavior).

## Core Responsibilities
- Register ephemera and ephemera quality as Lua classes/enums
- Expose ephemera property getters (position, facing, shape, collection, scale flags, animation flags)
- Expose ephemera property setters (for mutable contexts only)
- Implement ephemera creation (`new`) and deletion methods
- Implement position/polygon reassignment with spatial updates
- Gate write operations based on mutability context (world_mutable / ephemera_mutable)

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h`
- **Game world:** `ephemera.h` (core ephemera functions), `map.h`, `world.h` (coordinate/polygon types)
- **Lua bindings:** `lua_templates.h` (L_Class, L_Enum, L_Container, L_TableFunction), `lua_map.h` (Lua_Polygon, Lua_Collection)
- **Preferences:** `preferences.h` (graphics_preferences ΓåÆ ephemera_quality)
- **Rendering:** `render.h` (TEST_RENDER_FLAG, _polygon_is_visible)
- **Engine state:** Implicit extern functions `remove_ephemera`, `get_ephemera_data`, `new_ephemera`, `set_ephemera_shape`, `remove_ephemera_from_polygon`, `add_ephemera_to_polygon`, `get_dynamic_limit`; macros `DESCRIPTOR_SHAPE_BITS`, `DESCRIPTOR_COLLECTION_BITS`, `MAXIMUM_SHAPES_PER_COLLECTION`, `MAXIMUM_CLUTS_PER_COLLECTION`, `WORLD_ONE`, `FULL_CIRCLE`, `GET_DESCRIPTOR_*`, `GET_OBJECT_SCALE_FLAGS`, `SLOT_IS_USED`

# Source_Files/Lua/lua_ephemera.h
## File Purpose
Defines Lua binding classes for ephemera (temporary visual effects/objects) in the Aleph One game engine. Provides a Lua-accessible interface to create, access, and iterate over ephemera collections.

## Core Responsibilities
- Define `Lua_Ephemera` class template wrapping individual ephemera objects for Lua
- Define `Lua_Ephemeras` container template for iteration and collection access
- Declare registration function to expose ephemera classes to Lua state
- Bridge C++ ephemera objects with Lua scripting environment

## External Dependencies
- **Lua 5.2 headers** (`lua.h`, `lauxlib.h`, `lualib.h`) ΓÇö scripting runtime
- **`lua_templates.h`** ΓÇö `L_Class<>` and `L_Container<>` template definitions
- **`cseries.h`** ΓÇö engine-wide types and macros
- **Undefined:** `LuaMutabilityInterface` class (defined elsewhere; controls access permissions)

# Source_Files/Lua/lua_hud_objects.cpp
## File Purpose
Implements Lua bindings for HUD (heads-up display) objects in the Aleph One game engine, exposing game world state, player information, rendering primitives, and interface configuration to Lua scripts. Provides 40+ Lua classes and enums for scripting custom HUDs and UI overlays.

## Core Responsibilities
- Define and register Lua wrapper classes for HUD-related engine objects (images, shapes, fonts, screen state)
- Expose game state and player data to Lua (ticks, difficulty, scoring mode, kill limits, player roster)
- Provide color and geometry accessors for HUD interface elements
- Implement drawing functions for text, images, and shapes
- Register enum types for game constants (renderer types, masking modes, texture types, player colors, etc.)
- Initialize and validate Lua type hierarchy during engine startup

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (FFI layer)
- **Game engine core:** `world_view`, `dynamic_world`, `static_world`, `current_player`, `local_player_index` (game state)
- **HUD/Render:** `HUDRenderer.h`, `HUDRenderer_Lua.h`, `Image_Blitter.h`, `OGL_Blitter.h`, `Shape_Blitter.h`, `FontHandler.h` (drawing primitives)
- **Game world:** `items.h`, `player.h`, `motion_sensor.h`, `screen.h`, `map.h` (object definitions)
- **Game constants:** `alephversion.h`, `collection_definition.h` (defines accessed via extern functions)
- **Utilities:** `std::unordered_map`, `std::algorithm`, `std::cmath` (C++ stdlib)

# Source_Files/Lua/lua_hud_objects.h
## File Purpose

Header file declaring Lua bindings for HUD objects in the Aleph One game engine. Provides Lua script access to HUD player state, game information, screen properties, and UI-related enumerations (colors, fonts, rectangles, inventory sections, renderer types, sensor blips, textures).

## Core Responsibilities

- Declare extern name strings for each Lua class and enumeration type
- Define typedef aliases for L_Class template instances (HUDPlayer, HUDGame, HUDScreen)
- Define typedef aliases for L_Enum and L_EnumContainer template instances for UI/game enums
- Provide the registration function to bind all HUD types to a Lua state
- Export types for use by Lua binding implementations

## External Dependencies

- **cseries.h** ΓÇô Platform abstraction and core types
- **lua.h, lauxlib.h, lualib.h** ΓÇô Lua 5.2 C API
- **items.h, map.h** ΓÇô Game world definitions (for context; not directly used in bindings)
- **lua_templates.h** ΓÇô Template classes (L_Class, L_Enum, L_EnumContainer, L_ObjectClass) that implement Lua-to-C++ binding machinery

# Source_Files/Lua/lua_hud_script.cpp
## File Purpose
Implements Lua HUD state management and lifecycle for the Aleph One game engine. Manages loading, initialization, and execution of Lua scripts that customize the heads-up display, including trigger callbacks for init, draw, resize, and cleanup events.

## Core Responsibilities
- Create and maintain a singleton Lua VM instance for HUD scripting
- Load Lua HUD scripts from plugin files or buffers
- Execute script trigger callbacks (init, draw, resize, cleanup) at appropriate lifecycle points
- Register game engine functions accessible to Lua scripts (via `Lua_HUDObjects_register`)
- Track which game asset collections are required by the Lua script
- Manage script lifecycle: load ΓåÆ initialize ΓåÆ run ΓåÆ draw ΓåÆ cleanup

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö VM creation, state management, stack operations, library loading
- **Game engine:** `interface.h`, `mouse.h` ΓÇö view/input state (defined elsewhere)
- **Plugins:** `Plugins.h` ΓÇö discover HUD script in plugin metadata
- **Logging:** `Logging.h` ΓÇö error/warning output
- **Preferences:** `preferences.h` ΓÇö game configuration (not directly used in this file)
- **Boost iostreams:** `<boost/iostreams/*.hpp>` ΓÇö included but not used in visible code
- **File I/O:** `FileSpecifier`, `OpenedFile` ΓÇö load script from disk (defined elsewhere)
- **HUD objects:** `lua_hud_objects.h` ΓÇö `Lua_HUDObjects_register()` binds game functions to Lua (defined elsewhere)

# Source_Files/Lua/lua_hud_script.h
## File Purpose
Header file declaring the public interface for Lua-based HUD (Heads-Up Display) scripting in Aleph One. Provides lifecycle callbacks for HUD initialization, drawing, and cleanup, along with script loading and management functions.

## Core Responsibilities
- Declare HUD lifecycle callbacks (Init, Cleanup, Draw, Resize)
- Declare script loading and execution functions
- Provide HUD script path management (set/get)
- Expose Lua garbage collection marking for HUD objects
- Query HUD script runtime state

## External Dependencies
- `cseries.h` ΓÇö platform abstraction, SDL2 integration, type definitions
- `<string>` ΓÇö C++ STL for path management
- Lua C API ΓÇö Not directly included; used by implementation (defined elsewhere)

# Source_Files/Lua/lua_map.cpp
## File Purpose
Implements Lua bindings for game map entities and world properties. Provides a C API bridge between Lua scripts and the engine's map/level data, including platforms, polygons, lines, lights, media, terminals, and annotations.

## Core Responsibilities
- Define and register Lua wrapper classes for ~25 map entity types using the Lua template system
- Implement property getters and setters for all map-accessible game world objects
- Manage read/write access control via `LuaMutabilityInterface` (conditionally register mutators)
- Validate indices and enforce type constraints for all Lua Γåö C++ conversions
- Provide compatibility layer for legacy map scripts via embedded Lua code
- Bridge between Lua's dynamic typing and C++'s type-safe world data structures

## External Dependencies
- **Engine headers:** `interface.h` (game state), `network.h` (game_info), `map.h` (world data), `lightsource.h` (light_data), `platforms.h`, `player.h`, `projectiles.h`, `OGL_Setup.h` (OpenGL fog), `SoundManager.h`
- **Lua API:** `lua.h`, `lauxlib.h`, `lualib.h` (standard Lua C API)
- **Template system:** `lua_templates.h` (L_Class, L_Enum, L_Container)
- **Other Lua modules:** `lua_monsters.h`, `lua_objects.h`, `lua_player.h` (related entity bindings)
- **Imported symbols (defined elsewhere):**
  - `get_platform_data()`, `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `get_media_data()`, `get_light_data()`
  - `EndpointList`, `LineList`, `MediaList`, `LightList`, `MapAnnotationList`
  - `dynamic_world`, `static_world` (global world state)
  - `set_platform_state()`, `adjust_platform_endpoint_and_line_heights()`, `update_one_media()`
  - `OGL_GetFogData()`, `OGL_Fog_AboveLiquid`, `OGL_Fog_BelowLiquid`

# Source_Files/Lua/lua_map.h
## File Purpose
This header declares Lua bindings for the game engine's map system, enabling Lua scripts to interact with map geometry and environmental properties. It defines type-safe wrapper classes around core game structures (polygons, lines, platforms, lights, media, etc.) via template instantiation. The file serves as the primary interface layer between Lua scripts and the native C++ game world.

## Core Responsibilities
- Declare Lua class wrappers for all major map entity types (geometric and semantic)
- Define enum containers for mutable map properties (damage types, fog modes, transfer modes, etc.)
- Expose map geometry collections as iterable Lua tables
- Provide the single registration entry point for map module initialization
- Support read-only access to map state via Lua-friendly interfaces

## External Dependencies

**Notable includes**:
- `cseries.h` ΓÇô Cross-platform utilities and SDL2 integration
- `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Lua C API (wrapped in `extern "C"` block)
- `map.h` ΓÇô Defines `polygon_data`, `line_data`, `side_data`, `object_data`, etc.
- `lightsource.h` ΓÇô Defines `light_data`, `static_light_data`
- `lua_templates.h` ΓÇô Provides `L_Class<>`, `L_Enum<>`, `L_Container<>`, `L_EnumContainer<>` templates for Lua binding metaprogramming

**External symbols (defined elsewhere)**:
- `lua_State`, Lua C API functions ΓÇô Lua interpreter
- `L_Class<N,T>::Register()`, `L_Enum<...>::Register()` ΓÇô Template methods from lua_templates.h
- `polygon_data`, `line_data`, `side_data`, `light_data` ΓÇô Game structures from map.h and lightsource.h
- `LuaMutabilityInterface` ΓÇô Permission control interface (definition not provided in bundled headers)

# Source_Files/Lua/lua_mnemonics.h
## File Purpose
Defines string-to-integer mnemonic lookup tables for Lua scripting integration. Maps human-readable identifiers (e.g., "water", "missile") to numeric constant values across 42 game engine categories, enabling Lua scripts to reference game engine concepts using symbolic names.

## Core Responsibilities
- Define `lang_def` struct as a key-value pair container (string name ΓåÆ int32 value)
- Provide 42 global mnemonic arrays covering:
  - Audio (ambient sounds, sound effects)
  - Game mechanics (damage types, difficulty, game modes, scoring modes)
  - Game objects (items, weapons, monsters, projectiles, scenery, platforms)
  - Rendering (fade types, light states, transfer modes, textures, interface colors/fonts/rects)
  - Physics (media types, polygon types, control panel classes)
  - Player/monster state (colors, actions, modes, sensor blip types)
- Enable Lua scripts to reference engine enumerations by string names instead of raw numeric values

## External Dependencies
- **`#include "lua_script.h"`** ΓÇô Declares Lua integration functions and enumerations (e.g., `_game_of_most_points`). The bundled header shows this module controls script loading, cleanup, and engine callbacks.
- **Copyright/License** ΓÇô Dual GPL v3 and author attribution (Gregory Smith, 2008).


# Source_Files/Lua/lua_monsters.cpp
## File Purpose
Implements Lua scripting bindings for monster systems in a game engine (appears to be Marathon/Aleph One). Exposes monster types, individual monsters, their properties, relationships, and actions to the Lua scripting environment.

## Core Responsibilities
- **Lua type registration** for monsters, monster types, monster classes, modes, and actions
- **Property access** (getters/setters) for monster attributes (health, position, facing, active state, etc.)
- **Relationship management** between monster types (enemies, friends, alliances)
- **Damage interactions** (immunities, weaknesses for specific damage types)
- **Instance operations** (movement, acceleration, targeting, sound playback, deletion)
- **Backward compatibility** layer providing old API through Lua wrapper functions

## External Dependencies
- **`monsters.h`** ΓÇö `monster_data` struct, `monster_definition` struct, monster constants, activation/movement/damage functions
- **`player.h`** ΓÇö player data access, `monster_index_to_player_index()`
- **`flood_map.h`** ΓÇö pathfinding API (`new_path()`, `delete_path()`, `monster_pathfinding_cost_function`)
- **`lua_templates.h`** ΓÇö `L_Class`, `L_Enum`, `L_Container`, `L_EnumContainer` template classes for Lua binding
- **`lua_map.h`** ΓÇö `Lua_Polygon`, `Lua_Collection`, `Lua_DamageType` (cross-references for spatial and damage data)
- **`lua_objects.h`** ΓÇö `Lua_EffectType`, `Lua_Sound`, `Lua_ItemType` (cross-references for effects and items)
- **`lua_player.h`** ΓÇö `Lua_Player` (cross-reference for player instances)
- **Lua C API** ΓÇö `lua.h`, `lauxlib.h`, `lualib.h`
- **Game engine core** ΓÇö object management, world coordinate system, effect/sound playback

# Source_Files/Lua/lua_monsters.h
## File Purpose
Declares Lua bindings for monster management in the Aleph One game engine. Exposes individual monster instances, monster collections, and enumerations (monster types and actions) to Lua scripts via template-based C++ wrapper classes.

## Core Responsibilities
- Declare Lua-facing type aliases for monsters (single instances and containers)
- Define enumeration bindings for monster actions and types
- Provide a registration function to initialize Lua bindings with mutability constraints
- Bridge game engine monster data structures to Lua scripting API

## External Dependencies
- **Lua C API** (`lua.h`, `lauxlib.h`, `lualib.h`): Core scripting engine
- **cseries.h**: Game engine common utilities
- **map.h** (defined elsewhere): World/map data structures referenced by monsters
- **monsters.h** (defined elsewhere): Native `monster_data` structure and monster constants (types, actions, modes)
- **lua_templates.h** (defined elsewhere): Template classes (`L_Class`, `L_Enum`, `L_Container`) that provide the Lua binding infrastructure
- **LuaMutabilityInterface** (defined elsewhere): Struct controlling script-side mutation of monsters

**Not inferable from this file:**
- Monster property getters/setters (registered via `L_Class` mechanisms)
- Enum mnemonic strings (registered via `L_Enum` mechanisms)
- Whether the implementation is thread-safe or singleton-based

# Source_Files/Lua/lua_music.cpp
## File Purpose
Lua binding layer for music management, exposing C++ music control functionality (play, stop, fade, volume) to Lua scripts. Registers two Lua APIs: `music` (individual music slot control) and `Music` (manager-level operations like playlist management).

## Core Responsibilities
- Wrap `Music` singleton and `Slot` methods as Lua-callable C functions
- Parse and validate Lua arguments (file paths, numeric parameters, booleans)
- Manage music slot indices, accounting for reserved slots (intro/level)
- Handle file resolution using search paths (`L_Get_Search_Path`)
- Conditionally register mutable vs. read-only APIs based on `LuaMutabilityInterface`
- Push/get music slot indices to/from Lua stack

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö stack manipulation and error reporting
- **Music system:** `Music.h` ΓÇö singleton music manager and slot operations
- **Lua templates:** `lua_templates.h` ΓÇö `L_Class`, `L_Container`, `L_TableFunction` template helpers
- **File I/O:** `FileSpecifier` (defined elsewhere) ΓÇö path resolution
- **Audio decoding:** `StreamDecoder` (defined elsewhere) ΓÇö validates audio file formats
- **Utility:** `cseries.h` ΓÇö common types and macros
- **Helper function:** `L_Get_Search_Path()` (defined elsewhere) ΓÇö retrieves Lua script search path

# Source_Files/Lua/lua_music.h
## File Purpose
Header file that defines Lua bindings for the game's music system. Exposes individual music objects and a music manager container to Lua scripts via C++ template wrappers.

## Core Responsibilities
- Declare Lua class and container type wrappers for music objects
- Define extern string identifiers used as Lua metatable names
- Provide a registration function to bind music functionality to a Lua state
- Bridge the C++ music system with Lua scripting via template-based type marshalling

## External Dependencies
- **Lua C API** (`lua.h`, `lauxlib.h`, `lualib.h`): Standard Lua 5.2 C binding library; provides `lua_State`, stack manipulation, and registry access
- **lua_templates.h**: Template infrastructure (`L_Class`, `L_Container`) for wrapping C++ objects as Lua types
- **cseries.h**: Engine-wide utilities and platform abstractions
- **LuaMutabilityInterface**: Defined elsewhere; controls whether Lua-bound objects can be modified

# Source_Files/Lua/lua_objects.cpp
## File Purpose
Implements Lua bindings for game world objects (effects, items, scenery). Provides scripting interface to manipulate map objects, query their properties, and control their lifecycle through registered getter/setter methods and object containers.

## Core Responsibilities
- Register Lua bindings for object types (Effect, Item, Scenery) and their enumerations with conditional mutability
- Provide getter/setter methods for object properties (position, facing, visibility, polygon, type)
- Implement object lifecycle operations (creation via constructors, deletion, teleportation)
- Expose object containers (Effects, Items, Sceneries) as Lua tables with iteration and indexing
- Support type lookups and enumeration (ItemType, SceneryType, EffectType with mnemonics)
- Maintain backward compatibility with deprecated Lua APIs

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h`
- **Engine templates:** `lua_templates.h` (L_Class, L_Container, L_Enum infrastructure)
- **Map bindings:** `lua_map.h` (Lua_Polygon, Lua_Tags, etc.)
- **Game world modules:** `effects.h`, `items.h`, `monsters.h`, `scenery.h`, `player.h` (core data structures)
- **Sound system:** `SoundManager.h` 

**Defined elsewhere:**
- `remove_map_object()`, `get_object_data()`, `add_object_to_polygon_object_list()`, `remove_object_from_polygon_object_list()`
- `new_effect()`, `remove_effect()`, `get_effect_data()`, `teleport_object_in/out()`
- `new_item()`, `new_scenery()`, `damage_scenery()`, `deanimate_scenery()`, `randomize_scenery_shape()`
- `play_object_sound()`, `get_dynamic_limit()`, `get_placement_info()`, `get_item_definition_external()`
- Lua template method implementations and mnemonics tables (included from headers)

# Source_Files/Lua/lua_objects.h
## File Purpose
Declares Lua scripting bindings for game world objects (effects, items, scenery, sounds). Provides template-based wrapper classes that expose game data structures to Lua, along with registration function for initializing these bindings at engine startup.

## Core Responsibilities
- Declare typedef aliases mapping C++ game objects to Lua-accessible wrappers
- Define containers (L_Container) and enumerations (L_Enum) for object collections
- Expose extern string identifiers used as Lua metatable names
- Declare the registration function to initialize all Lua object bindings
- Support lazy-evaluation for sound objects via L_LazyEnum template

## External Dependencies
- **Lua C API:** lua.h, lauxlib.h, lualib.h (Lua 5.2 scripting interface)
- **Game data:** items.h (item definitions), map.h (world geometry)
- **Lua infrastructure:** lua_templates.h (template wrapper classes L_Class, L_Container, L_Enum, L_LazyEnum, L_EnumContainer)
- **Cross-platform:** cseries.h (includes SDL2, standard types)
- **LuaMutabilityInterface:** defined elsewhere; controls script permissions

# Source_Files/Lua/lua_player.cpp
## File Purpose
Implements Lua bindings for all player-related game objects in Aleph One, exposing player state (position, velocity, orientation, inventory, weapons, energy) to Lua scripts. Provides a template-based bridge between C++ game code and Lua scripting through reader/writer function registration.

## Core Responsibilities
- Register player, game state, camera, overlay, and compass objects with Lua VM
- Provide getter/setter methods for player properties (position, velocity, orientation, energy, oxygen, items, weapons)
- Manage action flag (input) queues accessible to Lua during `idle()` callbacks
- Implement camera path animation system with waypoint and angle keyframe management
- Expose player HUD overlays (text, icons, colors) to script control
- Provide inventory and weapon management with accounting (item creation/destruction tracking)
- Support game state queries and mutations (difficulty, game type, scoring mode, time remaining)
- Serialize/deserialize game state to/from binary format
- Maintain backwards compatibility with legacy Lua scripts via compatibility wrapper functions

## External Dependencies
- **ActionQueues.h**: `GetGameQueue()`, `countActionFlags()`, `peekActionFlags()`, `modifyActionFlags()` for input queue management
- **game state**: `dynamic_world`, `static_world`, `current_player_index`, `local_player_index` globals
- **player.h** (implied): `player_data`, `get_player_data()`, `damage_player()`, `mark_player_inventory_as_dirty()`
- **item_definitions.h**: `get_item_definition_external()`, `try_and_add_player_item()`, `item_definition` struct
- **monsters.h / projectiles.h**: Monster/projectile entity queries
- **map.h**: Polygon validation
- **lua_templates.h**: Template base classes (`L_Class<>`, `L_Container<>`, `L_Enum<>`) for Lua binding abstraction
- **screen.h**: `screen_printf()` for console output
- **Crosshairs.h**: `Crosshairs_IsActive()`, `Crosshairs_SetActive()`
- **game_window.h**: `mark_shield_display_as_dirty()`, `draw_panels()`
- **SoundManager.h**: `play_teleport_sound()`
- **fades.h**: `start_fade()`
- **ViewControl.h**: `resync_virtual_aim()`, `SetTunnelVision()`
- **network.h** / **network_games.h**: Networking state, team data
- **lua_map.h, lua_monsters.h, lua_objects.h**: Map/monster/object Lua bindings
- **Boost iostreams**: Memory-based I/O for serialization

# Source_Files/Lua/lua_player.h
## File Purpose
Declares Lua bindings for the Player class and player collections in Aleph One. Provides type-safe wrappers (`L_Class` templates) to expose in-game player objects and colors to Lua scripts via the C API.

## Core Responsibilities
- Define Lua-accessible `Player` class wrapper with metatable support
- Expose player collections (container of all players)
- Define player color enumeration and its container for script access
- Declare registration function to integrate these types into Lua VM
- Establish naming conventions for Lua-side access ("player", "Players", "player_color", "PlayerColors")

## External Dependencies
- **Lua C API** (`lua.h`, `lauxlib.h`, `lualib.h`): Extern "C" declarations for FFI.
- **cseries.h**: Platform abstraction and common utilities.
- **map.h** (included indirectly): Game world state (where player data likely resides).
- **lua_templates.h**: Template infrastructure (`L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer`) that implements metatable mechanics, getter/setter dispatch, and mnemonic lookup for Lua bindings.

# Source_Files/Lua/lua_projectiles.cpp
## File Purpose
Implements Lua C API bindings for projectile game objects, allowing game scripts to inspect and manipulate projectiles in the game world. Provides getters/setters for projectile properties, factory methods for creation, and a backward-compatibility layer for legacy script APIs.

## Core Responsibilities
- Register individual projectile accessors (position, orientation, ownership, targeting) with Lua
- Register the Projectiles container for indexed access to all active projectiles
- Provide projectile creation and deletion through Lua scripting
- Expose projectile type definitions and damage properties to scripts
- Convert between game engine units (fixed-point, angles) and Lua number formats
- Maintain backward-compatible Lua functions wrapping the new property-based API

## External Dependencies

- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h`
- **Game World:** `map.h` (world coordinates, polygons, objects), `monsters.h`, `player.h`, `projectiles.h` (core data)
- **Templates:** `lua_templates.h` (L_Class, L_Enum, L_Container, L_EnumContainer, L_TableFunction)
- **Limits:** `dynamic_limits.h` (MAXIMUM_PROJECTILES_PER_MAP via `get_dynamic_limit()`)
- **Utilities:** `projectile_definitions.h` (DONT_REPEAT_DEFINITIONS flag for static data)
- **Defined Elsewhere:**
  - `remove_projectile()`, `new_projectile()`, `get_projectile_data()`, `get_projectile_definition()`
  - `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
  - `get_object_data()`, `get_player_data()`, `play_object_sound()`
  - Lua wrapper types: `Lua_Monster`, `Lua_Player`, `Lua_ProjectileType`, `Lua_Polygon`, `Lua_Sound`, `Lua_DamageType`

# Source_Files/Lua/lua_projectiles.h
## File Purpose
Declares Lua/C++ binding wrappers for projectile objects and projectile type enumerations. Exposes these game entities to Lua scripts through the Lua VM via template-based wrapper classes and registration functions.

## Core Responsibilities
- Define Lua-accessible wrapper class for individual projectiles (indexing into game's projectile collection)
- Define Lua-accessible container class for iterating all projectiles
- Define Lua-accessible enum wrapper for projectile type classifications
- Define Lua-accessible container for projectile type enumerations (with string lookup via mnemonics)
- Declare registration function to bind these classes to the Lua VM

## External Dependencies
- **Includes**: 
  - `cseries.h` ΓÇö platform abstraction, standard types
  - `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.2 C API
  - `lua_templates.h` ΓÇö template wrappers (L_Class, L_Container, L_Enum, L_EnumContainer)
- **Defined elsewhere**: 
  - `LuaMutabilityInterface` ΓÇö defined in lua_script.h (or similar); controls read/write access
  - Implementation of `Lua_Projectiles_register()` ΓÇö in a .cpp file (e.g., lua_projectiles.cpp)
  - Actual game projectile data structures ΓÇö in core game engine (likely projectiles.h/projectiles.cpp)

# Source_Files/Lua/lua_saved_objects.cpp
## File Purpose
Implements Lua bindings for read-only access to saved map objects (goals, item/monster/player spawn points, sound emitters) defined in map files. Allows Lua scripts to query their positions, orientations, types, and flags without modifying them.

## Core Responsibilities
- Expose five map object types (Goals, ItemStarts, MonsterStarts, PlayerStarts, SoundObjects) as Lua classes
- Provide read-only getter methods for each object type (position, facing, polygon, type-specific metadata)
- Register Lua containers that allow iteration and indexing of all objects of each type
- Convert internal engine units (facing angles, world coordinates) to Lua-friendly formats
- Validate object indices and handle invalid object access

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Lua state and stack manipulation
- **Game Engine Core:** `map.h` ΓÇô `map_object` struct and polygon definitions; `monsters.h`, `items.h` ΓÇô type enumerations
- **Lua Template System:** `lua_templates.h` ΓÇô `L_Class`, `L_Container`, `L_Enum` metaprogramming
- **Sound Manager:** `SoundManagerEnums.h` ΓÇô sound constants (e.g., `MAXIMUM_SOUND_VOLUME`)
- **Included Files:** `lua_monsters.h`, `lua_objects.h`, `lua_player.h`, `lua_map.h` ΓÇô Lua bindings for related subsystems; `lua_saved_objects.h` ΓÇô declarations

# Source_Files/Lua/lua_saved_objects.h
## File Purpose
Declares Lua bindings for game map objects (goals, items, monsters, player starts, sounds) by typedef'ing template-based wrapper classes. Provides the registration entry point to expose these map structures to Lua scripts.

## Core Responsibilities
- Export extern char arrays defining Lua class/container names for map object types
- Define typedef wrappers combining `L_Class` and `L_Container` templates for six object types
- Declare the registration function that sets up all Lua bindings in the Lua state
- Act as a header-only bridge between C++ game objects and Lua

## External Dependencies
- **Includes:**
  - `cseries.h` ΓÇö common C series utilities (macros, platform abstractions)
  - `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.2 C API headers
  - `map.h` ΓÇö game map structure definitions (these objects are map-related)
  - `lua_templates.h` ΓÇö template implementations for `L_Class` and `L_Container` wrapper classes
- **Defined Elsewhere:**
  - `LuaMutabilityInterface` ΓÇö type passed to registration function; defines mutability constraints (not inferable from this file)
  - Implementations of all the `L_Class` and `L_Container` member functions (in `lua_templates.h`)

# Source_Files/Lua/lua_script.cpp
## File Purpose
Implements Lua script loading, execution, and engine integration for Aleph One. Manages multiple script contexts (embedded, solo, netscript, stats, achievements) and dispatches game events to registered Lua callbacks.

## Core Responsibilities
- Load Lua scripts from buffers and files into isolated state instances
- Initialize Lua with standard/sandbox libraries and engine function registrations
- Dispatch game events (player/monster actions, item/projectile interactions, level state) to Lua trigger callbacks
- Manage player control, weapon wielding, compass state, and HUD visibility via Lua
- Serialize/deserialize Lua state persistence across level transitions
- Provide interactive command execution for console/debugging
- Handle resource collection management and camera animation via Lua

## External Dependencies
- **Lua C API**: `lua.h`, `lauxlib.h`, `lualib.h` for state management and stack operations.
- **Lua modules** (this file): `lua_music.h`, `lua_ephemera.h`, `lua_map.h`, `lua_monsters.h`, `lua_objects.h`, `lua_player.h`, `lua_projectiles.h` for object registration.
- **Game engine**: `world.h`, `player.h`, `monsters.h`, `items.h`, `weapons.h`, `platforms.h`, `render.h`, `physics_models.h`.
- **Extern symbols**: `dynamic_world`, `static_world`, `world_view` (global engine state); `local_player_index` (active player); `local_random()` (RNG); `ShootForTargetPoint()`, `get_physics_constants_for_model()`, `instantiate_physics_variables()` (physics).

# Source_Files/Lua/lua_script.h
## File Purpose
Header for Lua scripting subsystem in Aleph One (Marathon game engine). Provides event callbacks for game world interactions, script lifecycle management, state persistence, and camera/cutscene control via Lua scripts.

## Core Responsibilities
- Script lifecycle (load, execute, cleanup, reset)
- Event dispatch for game interactions (damage, kills, switches, terminals, projectiles, items)
- State invalidation tracking (effects, monsters, projectiles, objects, ephemera)
- Lua state serialization/deserialization for save/load
- Camera path management for scripted cutscenes
- Game rule control (scoring modes, end conditions)
- Mutability enforcement (restricting Lua access to game systems)
- Achievement and stats collection
- Action queue management for input handling

## External Dependencies
- `cseries.h` ΓÇô platform/utility definitions
- `world.h` ΓÇô world coordinates (`world_point3d`), angles
- `ActionQueues.h` ΓÇô input queue management (`ActionQueues` class)
- `shape_descriptors.h` ΓÇô sprite/image descriptors
- Standard C++: `<map>`, `<string>`

# Source_Files/Lua/lua_serialize.cpp
## File Purpose
Serializes and deserializes Lua objects to/from binary streams for game state persistence. Supports all core Lua types with reference deduplication to handle circular references and shared objects. Based on Pluto but with simplified design.

## Core Responsibilities
- Recursively serialize Lua values (nil, number, boolean, string, table, userdata) to binary format
- Recursively deserialize binary data back into Lua values with proper type reconstruction
- Maintain reference tables during save/restore to deduplicate objects and preserve identity
- Validate table keys and filter unsupported types (functions, light userdata)
- Handle userdata reconstruction via metatable `__new` callbacks
- Implement version checking for forward compatibility

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (stack manipulation, type checking, metatable access)
- **BStream.h:** `BOStreamBE`, `BIStreamBE` (big-endian binary I/O)
- **Logging.h:** `logWarning()` macro (error reporting)
- **cseries.h:** Base types (`uint32`, `uint16`, `uint8`, `int8`, `double`)

# Source_Files/Lua/lua_serialize.h
## File Purpose
Declares the serialization interface for Lua objects in the game engine. Provides stream-based persistence functions to save and restore Lua state to/from buffers, enabling game save/load functionality.

## Core Responsibilities
- Export serialization API for Lua object persistence
- Provide stream abstraction via C++ standard library (`std::streambuf`)
- Wrap Lua C API for object marshalling

## External Dependencies
- `cseries.h` ΓÇö Project common header
- Lua 5.2 C API: `lua.h`, `lauxlib.h`, `lualib.h`
- C++ standard: `<streambuf>` (abstract I/O stream interface)

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

## External Dependencies
- **lua.h, lauxlib.h, lualib.h**: Lua 5.2 C API (lua_State, lua_CFunction, luaL_Reg, etc.)
- **cseries.h**: Engine type definitions (int16, uint16, uint8, OSErr, etc.)
- **lua_script.h**: Game-facing Lua declarations (L_Error, L_Call_Init, L_Persistent_Table_Key())
- **lua_mnemonics.h**: lang_def struct; extern mnemonic arrays (Lua_ItemType_Mnemonics, etc.)
- **<sstream>, <map>, <new>, <functional>**: C++ standard library
- **luaL_typerror()**: Helper defined in this file; wraps lua_pushfstring + luaL_argerror
- **Defined elsewhere (extern)**: L_Set_Search_Path(), L_Get_Search_Path(), L_Get_Proper_Item_Accounting(), L_Set_Proper_Item_Accounting(), L_Get_Nonlocal_Overlays(), L_Set_Nonlocal_Overlays(), L_Persistent_Table_Key()

# Source_Files/Lua/luaconf.h
## File Purpose

Configuration header for the Lua interpreter/library. Defines platform-specific feature detection, module paths, type sizes, API visibility macros, and numeric type formatting. All definitions are compile-time configuration with no runtime behaviorΓÇötuning is done by editing macros before compilation.

## Core Responsibilities

- **Platform detection and feature enablement** ΓÇö conditionally enable POSIX, Linux, macOS, or Windows-specific functionality
- **Module and library paths** ΓÇö define default search paths for Lua and C modules
- **API visibility and export** ΓÇö mark functions for DLL export, visibility ("hidden"), or static linkage
- **Numeric type definition** ΓÇö configure whether Lua uses `double`, `float`, or custom numeric types; define integer types
- **Number formatting and conversion** ΓÇö specify scanf/printf format strings and conversion functions for numbers
- **Stack and buffer limits** ΓÇö constrain Lua stack depth and auxiliary library buffer sizes
- **Math operation macros** ΓÇö define fast paths for arithmetic (+, ΓêÆ, ├ù, ├╖, mod, pow) conditional on compilation unit
- **Backward compatibility** ΓÇö provide deprecated function replacements and old API bindings via `LUA_COMPAT_*` flags
- **Error output routing** ΓÇö configure where `print()` and error messages are sent (stdout, stderr)
- **IEEE754 and endianness tricks** ΓÇö select fast double-to-integer conversion strategies per CPU architecture

## External Dependencies

### Includes (Conditional)
| Header | Condition | Purpose |
|--------|-----------|---------|
| `<limits.h>` | Always | `INT_MAX` for bit-width detection |
| `<stddef.h>` | Always | `size_t`, `ptrdiff_t` |
| `<stdio.h>` | `LUA_LIB` or `lua_c` | `fwrite()`, `fprintf()` for print and error output |
| `<math.h>` | `lobject_c` or `lvm_c` | `floor()`, `pow()` for Lua math |
| `"config.h"` | `HAVE_CONFIG_H` | Build-system-generated feature macros (Aleph One) |

### Platform/Compiler Symbols Used (Defined Elsewhere)
- `__STRICT_ANSI__`, `_WIN32`, `_WIN32_WCE` ΓÇö compiler/platform macros
- `__GNUC__`, `__ELF__` ΓÇö GCC visibility attributes
- `_MSC_VER`, `_M_IX86` ΓÇö MSVC/architecture detection
- `__i386__`, `__x86_64`, `__POWERPC__` ΓÇö CPU architecture macros

### Relationship to Aleph One (from config.h)
The packaged config.h names this as **Aleph One 20140104** (a Marathon game engine remake). Lua is configured with file I/O (`<pwd.h>`), graphics (`OPENGL`, `SDL_image`), networking (`SDL_net`, `miniupnpc`), audio (`SDL_ttf`), and compression (`zlib`, `zzip`) supportΓÇötypical for embedding Lua in a game engine for scripting and modding.

# Source_Files/Lua/lualib.h
## File Purpose
Public header declaring the Lua standard library module initialization API. Provides function signatures and constants for loading built-in libraries (coroutine, table, I/O, OS, string, bitwise, math, debug, package) into a Lua state.

## Core Responsibilities
- Declare module initialization functions (`luaopen_*`) for each standard library
- Define canonical module name constants (e.g., `LUA_TABLIBNAME`, `LUA_MATHLIBNAME`)
- Declare bulk initialization function (`luaL_openlibs`) to register all libraries at once
- Provide assertion macro for runtime validation
- Include the core Lua API header (`lua.h`)

## External Dependencies
- **`lua.h`**: Defines core Lua API types and functions (`lua_State`, `LUAMOD_API`, `LUALIB_API` macros, etc.)
- All module implementations ("defined elsewhere")

# Source_Files/Lua/lundump.h
## File Purpose
Header file for Lua's binary chunk serialization and deserialization system. Declares interfaces to load precompiled Lua bytecode from streams and dump Lua function prototypes to binary format, with associated binary format constants.

## Core Responsibilities
- Declare binary chunk loader for deserializing Lua bytecode
- Declare binary chunk dumper for serializing Lua prototypes
- Define binary format constants (magic tail bytes, header size)
- Declare header generation for binary files

## External Dependencies
- **lobject.h** ΓÇô Closure, Proto, Instruction types
- **lzio.h** ΓÇô ZIO (stream), Mbuffer (temporary buffer)
- Implicit: lua_State, lua_Writer callback type (defined in lua.h)
- Implicit: LUA_SIGNATURE constant (defined elsewhere, referenced in header generation)

# Source_Files/Lua/lvm.h
## File Purpose
Declares the core Lua virtual machine interface, including value conversion, comparison, table access, and arithmetic operations. Central to executing Lua bytecode at runtime.

## Core Responsibilities
- **Value conversion**: Convert between Lua types (string, number conversions)
- **Comparison operations**: Equality, less-than, less-equal checks with proper type handling
- **Table operations**: Read and write table elements with key lookup
- **VM execution**: Main interpreter loop and opcode execution
- **Arithmetic & metamethods**: Perform arithmetic operations, handle tagging system (tag methods)
- **String operations**: String concatenation with type coercion
- **Object introspection**: Get length of objects (via tag methods)

## External Dependencies
- **ldo.h**: Stack management (`luaD_call`, `luaD_precall` ΓÇö execution context)
- **lobject.h**: Object type definitions (TValue, StkId, type tags, metamethods)
- **ltm.h**: Tag method management (`TMS` enum, `luaT_gettm` ΓÇö metamethod lookup)
- **lua.h** (implicit): Core Lua C API types and constants

# Source_Files/Lua/lzio.h
## File Purpose
Defines buffered stream (ZIO) and memory buffer (Mbuffer) abstractions for Lua's I/O subsystem. Provides efficient character-by-character reading with user-supplied reader callbacks and dynamic buffer management for the lexer and chunk loading.

## Core Responsibilities
- Stream abstraction (`ZIO`) wrapping arbitrary reader functions with a refill buffer
- Macro-based fast path for character consumption (`zgetc`)
- Dynamic memory buffer allocation, resizing, and freeing (`Mbuffer`)
- Buffer initialization, space allocation, and state queries
- Stream initialization and batch reading interface

## External Dependencies
- **lua.h**: `lua_State`, `lua_Reader` callback typedef (`const char * (*lua_Reader)(lua_State *L, void *ud, size_t *sz)`)
- **lmem.h**: Memory allocation macros (`luaM_reallocvector`, `luaM_realloc_`)


