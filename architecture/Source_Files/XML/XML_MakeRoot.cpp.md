# Source_Files/XML/XML_MakeRoot.cpp

## File Purpose
Root XML parser dispatcher for Marathon's modular XML configuration system (MML). Loads and validates Marathon XML files, then routes parsed elements to appropriate subsystem parsers. Provides a centralized reset mechanism for all MML-configured values.

## Core Responsibilities
- Reset all MML-modified game subsystems to hardcoded defaults
- Load XML from files or memory buffers with comprehensive error handling
- Traverse XML tree and dispatch child elements to ~30 subsystem-specific parsers
- Support conditional parsing (menu-only vs. full) based on context
- Log parse errors with file path and exception details

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| InfoTree | class | XML document tree (external); provides load_xml() and children_named() |
| FileSpecifier | class | File path/specification abstraction (external) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_ParseAllMML` | function | static | Internal dispatcher; routes XML elements to subsystem parsers |

## Key Functions / Methods

### ResetAllMMLValues
- **Signature:** `void ResetAllMMLValues()`
- **Purpose:** Reset all game subsystems to hardcoded MML defaults
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls 27+ `reset_mml_*()` functions across subsystems (interface, weapons, items, monsters, textures, sounds, etc.)
- **Calls:** reset_mml_stringset, reset_mml_interface, reset_mml_motion_sensor, reset_mml_player, reset_mml_weapons, reset_mml_opengl, etc.
- **Notes:** Master reset for entire configuration state; typically called before loading a new map or reloading MML

### _ParseAllMML
- **Signature:** `static void _ParseAllMML(const InfoTree& fileroot, bool load_menu_mml_only)`
- **Purpose:** Recursively traverse XML tree and dispatch child elements to subsystem parsers
- **Inputs:** fileroot (XML document), load_menu_mml_only (skip non-menu-safe parsing if true)
- **Outputs/Return:** None
- **Side effects:** Modifies global state via parse_mml_*() calls
- **Calls:** parse_mml_stringset, parse_mml_interface, parse_mml_player, parse_mml_scenario, parse_mml_weapons, parse_mml_opengl, etc.
- **Notes:** Partitions parsing by flagΓÇöalways parses menu-safe items (stringset, interface, player_name); skips level-specific items (weapons, monsters, textures) when flag is true. Uses range-based for loop over `children_named("marathon")` and child iterators.

### ParseMMLFromFile
- **Signature:** `bool ParseMMLFromFile(const FileSpecifier& FileSpec, bool load_menu_mml_only)`
- **Purpose:** Load XML from file and parse with error recovery
- **Inputs:** FileSpec, load_menu_mml_only
- **Outputs/Return:** bool (true on success, false on exception)
- **Side effects:** File I/O; modifies global state via _ParseAllMML
- **Calls:** InfoTree::load_xml(), _ParseAllMML(), logError()
- **Exceptions caught:** InfoTree::parse_error, path_error, data_error, unexpected_error
- **Notes:** Returns false on any exception; logs detailed error with file path and exception message

### ParseMMLFromData
- **Signature:** `bool ParseMMLFromData(const char *buffer, size_t buflen)`
- **Purpose:** Parse XML from memory buffer instead of file
- **Inputs:** buffer (char*), buflen (size)
- **Outputs/Return:** bool (true on success, false on error)
- **Side effects:** Modifies global state; always loads full MML (not menu-only)
- **Calls:** InfoTree::load_xml(strm), _ParseAllMML(), logError()
- **Creates:** std::istringstream from buffer
- **Exceptions caught:** Same four InfoTree exception types
- **Notes:** Used for parsing embedded or dynamically-generated XML; always calls _ParseAllMML with false

## Control Flow Notes
Fits into init/shutdown and level-change flow: ResetAllMMLValues is called before loading MML, then ParseMMLFromFile/Data loads user/level-specific configurations. The load_menu_mml_only flag supports menu initialization (which needs interface/player name but not weapon balance).

## External Dependencies
- **Headers:** cseries.h (platform/base), 30+ subsystem headers (interface.h, weapons.h, monsters.h, items.h, etc.), InfoTree.h
- **External symbols:** InfoTree class, FileSpecifier class, ~60 parse_mml_*() and reset_mml_*() functions (defined elsewhere), logError() function
