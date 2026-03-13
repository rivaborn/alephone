# Source_Files/XML/QuickSave.h

## File Purpose
Defines a manager for auto-named saved game files ("quick saves") in the Aleph One game engine. Provides structures to represent individual saves and a singleton class to enumerate, manage, and manipulate collections of quick saves.

## Core Responsibilities
- Define `QuickSave` struct to encapsulate metadata (file path, name, level, timestamp, tick count, player count)
- Provide `QuickSaves` singleton to maintain an in-memory collection of discovered quick saves
- Support enumeration of saves from disk, deletion of surplus saves, and iteration over the collection
- Expose global functions for creating, deleting, and loading quick saves
- Track networked save status for multiplayer games

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `QuickSave` | struct | Represents a single quick save with metadata; comparable by `save_time` for sorting |
| `QuickSaves` | class | Singleton container managing a `std::vector<QuickSave>`; provides iteration and lifecycle operations |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `QuickSaves::instance()` | static `QuickSaves*` | singleton | Returns the unique manager instance |
| `m_saves` | `std::vector<QuickSave>` | member (private) | Holds enumerated quick save records |

## Key Functions / Methods

### QuickSaves::instance
- Signature: `static QuickSaves* instance()`
- Purpose: Access the singleton instance
- Inputs: None
- Outputs/Return: Pointer to `QuickSaves` singleton
- Side effects: None (or lazy initialization, not visible in header)
- Calls: Not inferable from this file
- Notes: Friend class `QuickSaveLoader` has special access

### QuickSaves::enumerate
- Signature: `void enumerate()`
- Purpose: Scan disk and populate `m_saves` with discovered quick save files
- Inputs: None
- Outputs/Return: None (modifies `m_saves` in place)
- Side effects: Reads filesystem; populates internal vector
- Calls: Not inferable from this file
- Notes: Must be called before iteration; clears or appends unclear from header alone

### QuickSaves::delete_surplus_saves
- Signature: `void delete_surplus_saves(size_t max_saves)`
- Purpose: Remove oldest quick saves if collection exceeds `max_saves` threshold
- Inputs: `max_saves` ΓÇô maximum number of saves to retain
- Outputs/Return: None
- Side effects: Modifies `m_saves`; may delete files from disk
- Calls: Not inferable from this file
- Notes: Relies on `QuickSave::operator<` for age-based sorting

### QuickSaves::begin / end
- Signature: `iterator begin()`, `iterator end()`
- Purpose: Provide STL-compatible iteration over saves
- Inputs: None
- Outputs/Return: Iterator to first/past-last element of `m_saves`
- Side effects: None
- Calls: `std::vector<QuickSave>::begin()` / `end()`
- Notes: Allows range-based iteration

### create_quick_save, delete_quick_save, load_quick_save_dialog
- Purpose: Global functions (implementation not in this file) to create a new save, delete an existing save, and display a dialog to load a save
- Notes: `load_quick_save_dialog` populates a `FileSpecifier`; `saved_game_was_networked` returns `size_t` (likely player count or flag)

## Control Flow Notes
This header is likely used during:
- **Game load** (UI dialog calls `load_quick_save_dialog`)
- **Game save** (calls `create_quick_save`, then possibly `delete_surplus_saves` to prune old saves)
- **Save menu refresh** (calls `enumerate()` to repopulate list)

The singleton pattern suggests `QuickSaves` is initialized once at startup and persists throughout the game session.

## External Dependencies
- **FileHandler.h**: `FileSpecifier` class for file/path abstraction
- **Standard library**: `<string>`, `<vector>`, `<time.h>` (for `time_t`)
- Implicit: Definition of `QuickSaveLoader` (friend class, defined elsewhere)
