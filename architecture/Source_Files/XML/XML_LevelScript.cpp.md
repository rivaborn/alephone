# Source_Files/XML/XML_LevelScript.cpp

## File Purpose
Manages XML-based scripts embedded in map files for the Aleph One game engine. Orchestrates per-level execution of MML scripts, Lua scripts, music playlists, movies, and load screens. Supports special pseudo-levels for defaults, restoration, and end-of-game sequences.

## Core Responsibilities
- Load and parse `marathon_levels` XML from map file resource 128
- Maintain a registry of level scripts indexed by level number
- Execute appropriate scripts when levels are loaded or game ends
- Manage movie specifications and playback parameters per level
- Coordinate music playlist setup and random-order playback
- Configure load screen appearance with images, colors, and progress indicators
- Support pseudo-levels (Default, Restore, End) for common operations
- Handle embedded MML and Lua script chunks from binary data

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `LevelScriptCommand` | struct | Single command (MML, Music, Movie, Lua, LoadScreen) with resource ID, file spec, and type-specific parameters |
| `LevelScriptHeader` | struct | Container for all commands for one level plus RandomOrder flag |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `LevelScripts` | `std::map<int, LevelScriptHeader>` | static | Registry mapping level indices to their script headers |
| `CurrScriptPtr` | `LevelScriptHeader*` | static | Pointer to currently executing script |
| `MovieFile` | `FileSpecifier` | static | File path for current level's movie |
| `MovieFileExists` | `bool` | static | Whether MovieFile points to an actual file |
| `MovieSize` | `float` | static | Relative playback size for current movie |
| `mmls_chunk` | `std::vector<uint8>` | static | Embedded binary MML script data |
| `luas_chunk` | `std::vector<uint8>` | static | Embedded binary Lua script data |
| `EndScreenIndex`, `NumEndScreens` | `short` (extern) | global | Configuration for end-of-game screens |

## Key Functions / Methods

### LoadLevelScripts
- **Signature:** `void LoadLevelScripts(FileSpecifier& MapFile)`
- **Purpose:** Load all level scripts from a map file's resource 128
- **Inputs:** MapFile ΓÇö the map file to load from
- **Outputs/Return:** None (populates static `LevelScripts` map)
- **Side effects:** Clears `LevelScripts` (except on first call); resets EndScreenIndex and NumEndScreens to defaults; logs parse errors
- **Calls:** `get_text_resource_from_scenario()`, `InfoTree::load_xml()`, `parse_levels_xml()`, `logError()`
- **Notes:** On first invocation, preserves external level scripts. Uses exception handling for XML parse errors.

### RunLevelScript
- **Signature:** `void RunLevelScript(int LevelIndex)`
- **Purpose:** Execute scripts for a level (default settings + level-specific + seed music)
- **Inputs:** LevelIndex ΓÇö the level to run scripts for
- **Outputs/Return:** None (side effects on MML state and Music)
- **Side effects:** Executes MML parsing, configures music playlist, applies load screens
- **Calls:** `GeneralRunScript()`, `Music::instance()->SeedLevelMusic()`
- **Notes:** Always runs Default script first, then level-specific script

### GeneralRunScript
- **Signature:** `void GeneralRunScript(int LevelIndex)`
- **Purpose:** Search for and execute all commands in a specific level's script
- **Inputs:** LevelIndex ΓÇö pseudo-level index (can be -3 Restore, -2 Default, -1 End, or regular level)
- **Outputs/Return:** None
- **Side effects:** Parses MML, loads Lua, configures music, sets load screens
- **Calls:** `ParseMMLFromData()`, `LoadLuaScript()`, `Music::instance()` methods, `OGL_LoadScreen::instance()` methods
- **Notes:** Early return if level not found; sets `CurrScriptPtr`; iterates all commands in order

### RunScriptChunks
- **Signature:** `void RunScriptChunks()`
- **Purpose:** Process embedded binary MML and Lua script chunks (from binary WAD format)
- **Inputs:** None (reads from static `mmls_chunk` and `luas_chunk`)
- **Outputs/Return:** None
- **Side effects:** Parses MML data, loads Lua scripts
- **Calls:** `AIStreamBE` (binary read), `ParseMMLFromData()`, `LoadLuaScript()`
- **Notes:** Parses big-endian binary format with 8-byte header per level; validates buffer bounds before parsing

### FindLevelMovie
- **Signature:** `void FindLevelMovie(short index)`
- **Purpose:** Locate movie specification for a level (checking Default then level-specific)
- **Inputs:** index ΓÇö level index
- **Outputs/Return:** None (updates static `MovieFile`, `MovieFileExists`, `MovieSize`)
- **Side effects:** Resets MovieFileExists and MovieSize; searches scripts
- **Calls:** `FindMovieInScript()`
- **Notes:** Always resets state first; searches Default before level-specific

### GetLevelMovie
- **Signature:** `FileSpecifier *GetLevelMovie(float& Size)`
- **Purpose:** Retrieve the movie file and size for playback
- **Inputs:** Size ΓÇö output parameter for movie size
- **Outputs/Return:** Pointer to `MovieFile` if exists, NULL otherwise
- **Side effects:** Modifies Size output parameter
- **Calls:** None
- **Notes:** Only updates Size if MovieSize >= 0; returns NULL if no movie found

### parse_levels_xml
- **Signature:** `static void parse_levels_xml(InfoTree root)`
- **Purpose:** Top-level XML parser; dispatches to level-specific parsers
- **Inputs:** root ΓÇö the `<marathon_levels>` InfoTree node
- **Outputs/Return:** None
- **Side effects:** Populates `LevelScripts` map via calls to `parse_level_commands()`
- **Calls:** `parse_level_commands()` (multiple times), InfoTree iteration
- **Notes:** Handles `<level>`, `<end>`, `<default>`, `<restore>`, `<end_screens>` elements

### parse_level_commands
- **Signature:** `void parse_level_commands(InfoTree root, int index)`
- **Purpose:** Parse all command elements (`<mml>`, `<lua>`, `<music>`, `<movie>`, `<load_screen>`) for one level
- **Inputs:** root ΓÇö level node; index ΓÇö level number (or pseudo-level constant)
- **Outputs/Return:** None (builds commands in `LevelScripts[index]`)
- **Side effects:** Creates/appends to LevelScriptHeader for given index
- **Calls:** InfoTree accessors; no direct execution
- **Notes:** Used by both regular levels and pseudo-levels; handles OpenGL load screens conditionally

### SetMMLS / GetMMLS
- **Signature:** `void SetMMLS(uint8* data, size_t length)` / `uint8* GetMMLS(size_t& length)`
- **Purpose:** Store and retrieve embedded binary MML script chunks
- **Inputs/Outputs:** data, length for get/set
- **Side effects:** Modifies static `mmls_chunk` vector
- **Calls:** `memcpy()`, vector operations
- **Notes:** Clear if length is 0; return NULL pointer if empty

### SetLUAS / GetLUAS
- **Signature:** `void SetLUAS(uint8* data, size_t length)` / `uint8* GetLUAS(size_t& length)`
- **Purpose:** Store and retrieve embedded binary Lua script chunks
- **Inputs/Outputs:** data, length for get/set
- **Side effects:** Modifies static `luas_chunk` vector
- **Calls:** `memcpy()`, vector operations
- **Notes:** Symmetric with SetMMLS/GetMMLS

## Control Flow Notes
- **Initialization:** `LoadLevelScripts()` called once per map load; populates registry from resource 128
- **Per-level:** `RunLevelScript(index)` called before each level; executes Default + level-specific scripts + music seed
- **End-of-game:** `RunEndScript()` executes Default + End pseudo-level scripts
- **State restoration:** `RunRestorationScript()` executes Default + Restore pseudo-level (used to revert MML changes)
- **Movie lookup:** `FindLevelMovie()` called before playback to locate and cache movie data
- **Binary chunks:** `RunScriptChunks()` processes embedded MML/Lua from WAD format after decompression

## External Dependencies
- **Music.h:** `Music::instance()` for playlist management, fading, playback
- **XML_ParseTreeRoot.h:** `ParseMMLFromData()` to parse MML from memory
- **lua_script.h:** `LoadLuaScript()` to load Lua scripts (with `_embedded_lua_script` context)
- **OGL_LoadScreen.h:** `OGL_LoadScreen::instance()` for load screen rendering (OpenGL)
- **Plugins.h:** `Plugins::instance()->load_mml()` for plugin scripts
- **InfoTree.h:** XML parsing infrastructure (`InfoTree::load_xml()`, node iteration, attribute reading)
- **AStream.h:** `AIStreamBE` for big-endian binary stream parsing
- **Logging.h:** `logError()` for parse error reporting
- **images.h:** `get_text_resource_from_scenario()` to retrieve TEXT resource 128 from map file
