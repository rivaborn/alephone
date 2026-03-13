# Source_Files/Misc/PlayerName.cpp

## File Purpose
Manages player name storage and configuration for netgame sessions in Aleph One. Provides a getter function and MML (Marathon Markup Language) parser to load and store the player's name from engine configuration files.

## Core Responsibilities
- Store the active player name in a static buffer
- Provide read-only access to the stored player name via getter
- Parse player name from InfoTree (XML/INI configuration structures)
- Support configuration reset (currently unused)

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PlayerName` | `char[256]` | static | Stores the current player's network name (255 chars max + null terminator) |

## Key Functions / Methods

### GetPlayerName
- **Signature:** `const char *GetPlayerName()`
- **Purpose:** Retrieve the currently stored player name.
- **Inputs:** None.
- **Outputs/Return:** Pointer to static `PlayerName` buffer.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Inline return; safe as long as caller doesn't retain pointer across configuration changes.

### parse_mml_player_name
- **Signature:** `void parse_mml_player_name(const InfoTree& root)`
- **Purpose:** Extract player name from configuration tree and apply UTF-8 conversion.
- **Inputs:** `root` ΓÇô InfoTree node containing optional string value.
- **Outputs/Return:** None (writes to static `PlayerName`).
- **Side effects:** Updates static `PlayerName` buffer; may truncate at 255 chars.
- **Calls:** `root.get_value_optional<std::string>()`, `DeUTF8_C()` (from TextStrings).
- **Notes:** Uses boost::optional for safe extraction; silently ignores missing or invalid values.

### reset_mml_player_name
- **Signature:** `void reset_mml_player_name()`
- **Purpose:** Reset player name to defaults.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** None (currently a no-op).
- **Calls:** None.
- **Notes:** Stub function; no reset logic implemented.

## Control Flow Notes
Part of the engine's configuration loading subsystem, invoked during MML parsing (likely in `parse_mml_*` callback chains). Intended to set the player name once at engine initialization before netgame startup.

## External Dependencies
- **cseries.h** ΓÇô engine common utilities and includes
- **PlayerName.h** ΓÇô public interface
- **TextStrings.h** ΓÇô `DeUTF8_C()` for UTF-8 decoding to C strings
- **InfoTree.h** ΓÇô Boost.PropertyTree wrapper for XML/INI config parsing
- **\<string.h\>** ΓÇô C standard library (included but not visibly used)
- **Boost** ΓÇô property_tree (via InfoTree)
