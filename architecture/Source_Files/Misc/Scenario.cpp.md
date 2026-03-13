# Source_Files/Misc/Scenario.cpp

## File Purpose
Manages scenario metadata (name, ID, version, classic gameplay flag) and version compatibility for the Aleph One engine. Implements a singleton accessor and parses scenario configuration from InfoTree (XML/structured data).

## Core Responsibilities
- Provide singleton access to the global scenario instance
- Store and validate compatible scenario versions
- Check version compatibility between scenarios
- Parse scenario metadata from MML/XML configuration structures
- Truncate string fields to engine-defined limits

## Key Types / Data Structures
None (class definition is in header; only uses `std::string` and `std::vector<string>`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `Scenario*` | static (function-local) | Singleton instance, lazily initialized in `instance()` |

## Key Functions / Methods

### Scenario::instance()
- **Signature:** `static Scenario* instance()`
- **Purpose:** Returns the singleton scenario instance, initializing it on first call
- **Inputs:** None
- **Outputs/Return:** Pointer to the global `Scenario` instance
- **Side effects:** Allocates a new `Scenario` object on heap on first invocation (one-time)
- **Calls:** `new Scenario()` (constructor)
- **Notes:** Lazy singleton; not thread-safe; instance is never freed

### Scenario::AddCompatible()
- **Signature:** `void AddCompatible(const string Compatible)`
- **Purpose:** Records a scenario version as compatible with the current scenario
- **Inputs:** `Compatible` ΓÇô scenario version string
- **Outputs/Return:** None
- **Side effects:** Appends truncated string to `m_compatibleVersions` vector
- **Calls:** `vector::push_back()`
- **Notes:** Truncates input to 24 characters (indices 0ΓÇô23)

### Scenario::IsCompatible()
- **Signature:** `bool IsCompatible(const string Compatible)`
- **Purpose:** Checks if a given scenario version is compatible
- **Inputs:** `Compatible` ΓÇô scenario version string to check
- **Outputs/Return:** `true` if compatible, `false` otherwise
- **Side effects:** None
- **Calls:** `vector::size()`, equality operators
- **Notes:** Returns `true` if either string is empty, if they match exactly, or if `Compatible` is in the compatibility list; linear search

### parse_mml_scenario()
- **Signature:** `void parse_mml_scenario(const InfoTree& root)`
- **Purpose:** Parses scenario metadata from an XML/structured config tree
- **Inputs:** `root` ΓÇô InfoTree node containing scenario attributes and children
- **Outputs/Return:** None
- **Side effects:** Updates singleton instance's name, ID, version, classic gameplay flag, and compatible versions
- **Calls:** `InfoTree::read_attr()`, `InfoTree::children_named()`, `Scenario::instance()`, setter methods
- **Notes:** Iterates over child nodes named `"can_join"` to populate compatible versions; attributes are optional (fail-safe reads)

### reset_mml_scenario()
- **Signature:** `void reset_mml_scenario()`
- **Purpose:** Reset hook for scenario state (currently unused)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** None
- **Notes:** Empty stub; likely reserved for future MML reload cleanup

## Control Flow Notes
This file is part of the **initialization / configuration** phase, not the frame loop. `parse_mml_scenario()` is called by the MML parser when loading scenario metadata from disk or config files. The singleton pattern suggests scenario is accessed globally and remains constant after initialization.

## External Dependencies
- **cseries.h** ΓÇô engine platform/utility macros and types
- **Scenario.h** ΓÇô `Scenario` class definition
- **InfoTree.h** ΓÇô XML/property-tree parsing interface (`read_attr()`, `children_named()`)
- **std::string, std::vector** ΓÇô STL containers
