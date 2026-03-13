# Source_Files/XML/XML_LevelScript.h

## File Purpose
Header file declaring the interface for loading, parsing, and executing XML-based level scripts in Aleph One. Manages level initialization scripts written in Pfhortran/MML (Marathon Markup Language) and Lua, handles level and end-game movie playback, and provides restoration of default parameters across level transitions.

## Core Responsibilities
- Loading level scripts from map file resources (resource 128)
- Running level-specific MML/Pfhortran scripts during level initialization
- Managing embedded MML and Lua script data (get/set operations)
- Locating and retrieving movie files for level playback and end-game sequences
- Executing end-of-game scripts and restoration scripts for parameter reset
- Parsing and managing MML default level configurations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class (external) | Encapsulates file paths and handles file I/O operations for loading map files |
| `InfoTree` | class (external, forward-declared) | XML/configuration tree structure for parsing MML defaults |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `EndScreenIndex` | extern short | global | Selects the fake level index for end-of-game screen display |
| `NumEndScreens` | extern short | global | Specifies how many end-of-game screens to display (resource IDs in sequence) |

## Key Functions / Methods

### LoadLevelScripts
- **Signature:** `void LoadLevelScripts(FileSpecifier& MapFile)`
- **Purpose:** Loads all level scripts from a map file (resource 128 or equivalent).
- **Inputs:** Map file specifier reference
- **Outputs/Return:** None
- **Side effects:** Loads and caches all level scripts into memory
- **Calls:** FileSpecifier methods (via MapFile argument)
- **Notes:** Runs once during map/game initialization

### RunLevelScript
- **Signature:** `void RunLevelScript(int LevelIndex)`
- **Purpose:** Execute level-specific scriptsΓÇöloads Pfhortran, runs MML for the given level.
- **Inputs:** Level index (0-based)
- **Outputs/Return:** None
- **Side effects:** Modifies game state (parameters, entities) per MML script directives
- **Calls:** MML/Pfhortran interpreter, parameter setters
- **Notes:** Called when entering a new level

### FindLevelMovie
- **Signature:** `void FindLevelMovie(short index)`
- **Purpose:** Locates the movie file for a level and the end-of-game movie.
- **Inputs:** Level index
- **Outputs/Return:** None (result stored internally)
- **Side effects:** Caches movie file specifiers
- **Calls:** File I/O operations
- **Notes:** Populates internal state for `GetLevelMovie()`

### GetLevelMovie
- **Signature:** `FileSpecifier *GetLevelMovie(float& PlaybackSize)`
- **Purpose:** Retrieves pointer to the movie file specifier for playback.
- **Inputs:** Playback size (by reference, updated if specified in script)
- **Outputs/Return:** Pointer to FileSpecifier; NULL if no movie configured
- **Side effects:** Updates playback size parameter if set in script
- **Calls:** None visible in this file
- **Notes:** Returns NULL for no movie; caller should check before use

### SetMMLS / SetLUAS
- **Signature:** `void SetMMLS(uint8* data, size_t length); void SetLUAS(uint8* data, size_t length)`
- **Purpose:** Embed MML and Lua script data (typically from map file resources).
- **Inputs:** Raw byte pointer and length
- **Outputs/Return:** None
- **Side effects:** Stores embedded script data for later execution
- **Calls:** Memory allocation (implicit)
- **Notes:** Caller retains ownership; these functions copy or take ownership as needed

### GetMMLS / GetLUAS
- **Signature:** `uint8* GetMMLS(size_t& length); uint8* GetLUAS(size_t& length)`
- **Purpose:** Retrieve embedded MML/Lua script data.
- **Inputs:** Size reference (output)
- **Outputs/Return:** Pointer to script data; length set by reference
- **Side effects:** None
- **Calls:** None visible
- **Notes:** Pointer validity depends on script lifetime

### RunEndScript
- **Signature:** `void RunEndScript()`
- **Purpose:** Execute scripts designated for game end (e.g., credits, final state changes).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies game state per end-game MML
- **Calls:** MML interpreter
- **Notes:** Intended for end-of-game flow

### RunRestorationScript
- **Signature:** `void RunRestorationScript()`
- **Purpose:** Reset parameter values to defaults by running default MML.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Restores game parameters to baseline configuration
- **Calls:** MML interpreter
- **Notes:** Used to undo level-specific parameter changes (e.g., on level restart)

### parse_mml_default_levels / reset_mml_default_levels
- **Signature:** `void parse_mml_default_levels(const InfoTree& root); void reset_mml_default_levels()`
- **Purpose:** Parse MML default level configurations and reset to baseline.
- **Inputs:** XML config tree (parse) or none (reset)
- **Outputs/Return:** None
- **Side effects:** Populates or clears default level parameters in global state
- **Calls:** InfoTree traversal, parameter setters
- **Notes:** Called during engine initialization and cleanup

## Control Flow Notes
- **Initialization phase:** `LoadLevelScripts()` loads all scripts from the map file; `parse_mml_default_levels()` initializes defaults from config.
- **Level transition:** `FindLevelMovie()` and `RunLevelScript()` execute when entering a level.
- **Frame updates:** `RunScriptChunks()` runs incremental script execution (if scripts support chunking).
- **Game end:** `RunEndScript()` executes end-game logic.
- **Restoration:** `RunRestorationScript()` resets state (e.g., on level restart or error recovery).

## External Dependencies
- **FileHandler.h** ΓÇö `FileSpecifier` class for file path abstraction and I/O
- **Implicit:** Pfhortran/MML interpreter (not declared here; likely in implementation file)
- **Forward-declared:** `InfoTree` class for XML configuration parsing (defined elsewhere)
- **C standard:** `<cstdint>` types (`uint8`, `size_t`); `<cstdio>` indirectly via FileHandler
