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

## Key Types / Data Structures
None defined in this file. Relies on external:

| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Music` | typedef (L_Class) | Lua userdata wrapper for individual music slots |
| `Lua_MusicManager` | typedef (L_Container) | Lua container wrapping all music slots |
| `FileSpecifier` | class (external) | File path resolution and validation |
| `MusicParameters` | struct (external) | Volume and loop settings |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_MusicManager_Methods[]` | `luaL_Reg[]` | static | Registry of MusicManager methods for Lua |
| `Lua_Music_Get[]` | `luaL_Reg[]` | static | Registry of Music getters (volume, active) |
| `Lua_Music_Get_Mutable[]` | `luaL_Reg[]` | static | Registry of Music mutators (fade, play, stop) |
| `Lua_Music_Set[]` | `luaL_Reg[]` | static | Registry of Music setters (volume) |
| `Lua_Music_Name[]` | `char[]` | global extern | Lua class name: "music" |
| `Lua_MusicManager_Name[]` | `char[]` | global extern | Lua class name: "Music" |

## Key Functions / Methods

### Lua_MusicManager_New
- **Signature:** `static int Lua_MusicManager_New(lua_State* L)`
- **Purpose:** Create a new music slot with a file, returning its index to Lua.
- **Inputs:** L[1] = filename (string), L[2] = volume (number, default 1.0), L[3] = loop (boolean, default true)
- **Outputs/Return:** 1 (pushes adjusted music slot index to Lua stack)
- **Side effects:** Calls `Music::instance()->Add()`, allocates new slot, may fail if file not found or slots full
- **Calls:** `L_Get_Search_Path()`, `FileSpecifier::SetNameWithPath()`, `Music::instance()->Add()`, `Lua_Music::Push()`
- **Notes:** Index returned is adjusted by subtracting `Music::reserved_music_slots` before exposing to Lua

### Lua_MusicManager_Play
- **Signature:** `static int Lua_MusicManager_Play(lua_State* L)`
- **Purpose:** Queue one or more level music files for playback.
- **Inputs:** L[1..n] = filenames (all strings)
- **Outputs/Return:** 0
- **Side effects:** Resolves file paths, calls `Music::instance()->PushBackLevelMusic()` for each valid file
- **Calls:** `L_Get_Search_Path()`, `FileSpecifier::SetNameWithPath()`, `Music::instance()->PushBackLevelMusic()`
- **Notes:** Silently ignores files that fail to resolve; does not fail on individual file errors

### Lua_Music_Fade
- **Signature:** `static int Lua_Music_Fade(lua_State* L)`
- **Purpose:** Fade a specific music slot to a target volume over a duration.
- **Inputs:** L[1] = slot index, L[2] = limit volume (number), L[3] = duration in seconds (number, default 1.0), L[4] = stop on silence (boolean, default true)
- **Outputs/Return:** 0
- **Side effects:** Calls `Slot::Fade()`, modifies internal fade state
- **Calls:** `Lua_Music::Index()`, `Music::instance()->GetSlot()`, `Slot::Fade()`
- **Notes:** Duration is scaled from seconds to milliseconds; error if slot not found

### Lua_Music_Play, Lua_Music_Stop
- **Signature:** `static int Lua_Music_Play(lua_State* L)` / `static int Lua_Music_Stop(lua_State* L)`
- **Purpose:** Play or pause a specific music slot.
- **Inputs:** L[1] = slot index
- **Outputs/Return:** 0
- **Side effects:** Calls `Slot::Play()` or `Slot::Pause()`
- **Calls:** `Lua_Music::Index()`, `Music::instance()->GetSlot()`
- **Notes:** `Stop` actually calls `Pause()`, not stop; error if slot not found

### Lua_Music_Volume_Get, Lua_Music_Volume_Set
- **Signature:** `static int Lua_Music_Volume_Get/Set(lua_State* L)`
- **Purpose:** Get/set volume of a music slot.
- **Inputs:** Get: L[1] = slot index; Set: L[1] = slot index, L[2] = volume (number)
- **Outputs/Return:** Get returns 1 (pushes volume); Set returns 0
- **Side effects:** Set calls `Slot::SetVolume()`
- **Calls:** `Lua_Music::Index()`, `Music::instance()->GetSlot()`, `Slot::GetParameters()` / `Slot::SetVolume()`

### Lua_Music_Active_Get
- **Signature:** `static int Lua_Music_Active_Get(lua_State* L)`
- **Purpose:** Check if a music slot is currently playing.
- **Inputs:** L[1] = slot index
- **Outputs/Return:** 1 (pushes boolean)
- **Side effects:** None
- **Calls:** `Lua_Music::Index()`, `Music::instance()->GetSlot()`, `Slot::Playing()`

### Lua_Music_Valid (static helper)
- **Signature:** `static bool Lua_Music_Valid(int16 index)`
- **Purpose:** Validate a music slot index.
- **Inputs:** index (int16)
- **Outputs/Return:** bool (true if slot exists and is valid)
- **Calls:** `Music::instance()->GetSlot()`
- **Notes:** Assigned to `Lua_Music::Valid` during registration

### Lua_MusicManager_Valid
- **Signature:** `static int Lua_MusicManager_Valid(lua_State* L)`
- **Purpose:** Check if one or more music file paths can be found and decoded.
- **Inputs:** L[1..n] = filenames (all strings)
- **Outputs/Return:** n (pushes boolean for each file)
- **Side effects:** None (read-only file validation)
- **Calls:** `L_Get_Search_Path()`, `FileSpecifier::SetNameWithPath()`, `StreamDecoder::Get()`
- **Notes:** Validates each file independently; returns multiple booleans

### Lua_Music_register
- **Signature:** `int Lua_Music_register(lua_State* L, const LuaMutabilityInterface& m)`
- **Purpose:** Register the music Lua API, conditionally exposing mutable operations based on context.
- **Inputs:** L = Lua state, m = mutability interface (checks `world_mutable()` and `music_mutable()`)
- **Outputs/Return:** 0
- **Side effects:** Registers Lua methods and setters; sets `Lua_Music::Valid` callback and `Lua_MusicManager::Length`
- **Calls:** `Lua_Music::Register()`, `Lua_Music::RegisterAdditional()`, `Lua_MusicManager::Register()`
- **Notes:** Read-only mode omits fade/play/stop/new; mutable mode adds all; max 16 music slots hardcoded

## Control Flow Notes
Initialization phase: `Lua_Music_register()` is called during engine startup with a `LuaMutabilityInterface` controlling whether Lua scripts can modify music. At runtime, Lua scripts call these bound functions to query or control music playback. The actual playback is delegated to the `Music` singleton, which manages frame-by-frame audio updates separately.

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö stack manipulation and error reporting
- **Music system:** `Music.h` ΓÇö singleton music manager and slot operations
- **Lua templates:** `lua_templates.h` ΓÇö `L_Class`, `L_Container`, `L_TableFunction` template helpers
- **File I/O:** `FileSpecifier` (defined elsewhere) ΓÇö path resolution
- **Audio decoding:** `StreamDecoder` (defined elsewhere) ΓÇö validates audio file formats
- **Utility:** `cseries.h` ΓÇö common types and macros
- **Helper function:** `L_Get_Search_Path()` (defined elsewhere) ΓÇö retrieves Lua script search path
