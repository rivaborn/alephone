# Source_Files/RenderOther/TextStrings.cpp

## File Purpose
Implements a hierarchical text string repository system that replaces MacOS STR# resources. Strings are indexed by ID and sub-index, with support for XML/MML parsing and UTF-8 to ASCII/Mac Roman conversion for localization.

## Core Responsibilities
- Store, retrieve, and delete strings in a two-level indexed repository (ID ΓåÆ Index ΓåÆ String)
- Parse string definitions from InfoTree (XML/MML) configuration
- Convert UTF-8 encoded input strings to ASCII/Mac Roman format
- Provide null-pointer safety and bounds checking on all lookups
- Maintain global string state throughout engine lifetime

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `StringSet` | typedef | `std::map<short, std::string>` ΓÇö strings indexed within a single ID |
| `StringSetMap` | typedef | `std::map<short, StringSet>` ΓÇö maps ID to string sets |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `StringSetRoot` | `StringSetMap` | static (file) | Global repository holding all string sets organized by ID |

## Key Functions / Methods

### TS_PutCString
- Signature: `void TS_PutCString(short ID, short Index, const char *String)`
- Purpose: Add or replace a string in the repository
- Inputs: ID (set identifier), Index (position within set), String (C-string pointer)
- Outputs/Return: None
- Side effects: Modifies `StringSetRoot`; creates new set entry if needed
- Calls: `std::map::operator[]`
- Notes: Silently ignores negative indices; no bounds validation

### TS_GetCString
- Signature: `const char *TS_GetCString(short ID, short Index)`
- Purpose: Retrieve a string from the repository
- Inputs: ID, Index
- Outputs/Return: Pointer to string (or NULL if not found)
- Side effects: None
- Calls: `std::map::find()`, iterator dereferencing
- Notes: Returns NULL if ID or Index missing; pointer validity depends on string lifecycle

### TS_IsPresent
- Signature: `bool TS_IsPresent(short ID)`
- Purpose: Check whether a string set exists
- Inputs: ID
- Outputs/Return: true if set exists, false otherwise
- Side effects: None
- Calls: `std::map::count()`

### TS_CountStrings
- Signature: `size_t TS_CountStrings(short ID)`
- Purpose: Get count of strings in a set
- Inputs: ID
- Outputs/Return: String count (0 if set absent)
- Side effects: None
- Calls: `std::map::find()`, `std::map::size()`
- Notes: Does not verify contiguity; comment claims contiguity from index zero but implementation doesn't enforce it

### DeUTF8 & DeUTF8_C
- Signature: `static size_t DeUTF8(const char *InString, size_t InLen, char *OutString, size_t OutMaxLen)`
- Purpose: Decode UTF-8 byte sequences into ASCII/Mac Roman characters
- Inputs: UTF-8 encoded input buffer, input length, output buffer, max output length
- Outputs/Return: Character count written to output
- Side effects: Writes to output buffer
- Calls: `unicode_to_mac_roman()` (defined elsewhere)
- Notes: Handles 1ΓÇô6 byte UTF-8 sequences; malformed bytes become '$'; does not enforce strict UTF-8 (no overlong detection); `DeUTF8_C` null-terminates and wraps `DeUTF8`

### parse_mml_stringset
- Signature: `void parse_mml_stringset(const InfoTree& root)`
- Purpose: Parse string definitions from XML/MML tree structure
- Inputs: `InfoTree` root containing "index" attribute and child "string" elements
- Outputs/Return: None
- Side effects: Populates `StringSetRoot` via UTF-8 conversion
- Calls: `InfoTree::read_attr()`, `InfoTree::children_named()`, `InfoTree::read_indexed()`, `DeUTF8_C()`, `std::map::operator[]`
- Notes: Skips children with invalid indices; allocates fixed 256-byte buffer for UTF-8 conversion (truncates if larger)

### Cleanup helpers
- `TS_DeleteString`, `TS_DeleteStringSet`, `TS_DeleteAllStrings`, `reset_mml_stringset` ΓÇö straightforward wrapper calls around map erase/clear; `reset_mml_stringset` is a no-op.

## Control Flow Notes
- Initialization: Strings are loaded via `parse_mml_stringset()` during engine configuration (called externally)
- Runtime: Strings queried via `TS_GetCString()` and managed via put/delete functions
- Shutdown: Cleared via `TS_DeleteAllStrings()` (assumed called by engine)
- UTF-8 conversion happens only during MML parsing, not during retrieval

## External Dependencies
- `<string>`, `<map>` ΓÇö C++ STL containers
- `cseries.h` ΓÇö platform-abstraction types (`uint8`, `uint16`, `uint32`, `int16`, `size_t`)
- `TextStrings.h` ΓÇö public interface declarations
- `InfoTree.h` ΓÇö XML/MML tree parsing
- `unicode_to_mac_roman()` ΓÇö defined elsewhere; converts Unicode code points to Mac Roman bytes
