# Source_Files/Misc/DefaultStringSets.cpp

## File Purpose

Provides compiled-in fallback text strings for the Marathon Aleph One game engine when MML (Marathon Markup Language) resources are unavailable. Contains original string data captured from the Marathon resource fork, organized into categorized string sets covering errors, UI labels, weapon/item names, network messages, and game configuration options.

## Core Responsibilities

- Define static string arrays for error messages, filenames, UI text, and game metadata
- Populate the TextStrings repository with fallback string sets during engine initialization
- Support 25+ distinct string sets covering game errors, networking, preferences, items, weapons, and difficulty levels
- Provide team color names and game-type labels for multiplayer setup
- Act as a source-controlled alternative to binary resource files

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sStringSetNumber128` through `sStringSetNumber200` | static const char* array | Game-specific string sets (errors, filenames, UI, network messages, item/weapon names, statistics) |
| `sDifficultyLevelsStrings` | static const char* array | Difficulty level names for game setup |
| `sNetworkGameTypesStrings` | static const char* array | Game mode labels (Every Man for Himself, Cooperative, CTF, etc.) |
| `sEndConditionTypeStrings` | static const char* array | Game end-condition types (Unlimited, Time Limit, Score Limit) |
| `sSingleOrNetworkStrings` | static const char* array | Game classification options |
| `sTeamColorNamesStrings` | static const char* array | Team color identifiers (Slate, Red, Violet, Yellow, White, Orange, Blue, Green) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sStringSetNumber128` ΓÇô `sStringSetNumber200` | const char* array[] | static | Hardcoded string set data for various game subsystems |
| `sDifficultyLevelsStrings` | const char* array[] | static | Difficulty labels used in game setup widgets |
| `sNetworkGameTypesStrings` | const char* array[] | static | Multiplayer game mode names |
| `sEndConditionTypeStrings` | const char* array[] | static | Game duration/scoring condition labels |
| `sSingleOrNetworkStrings` | const char* array[] | static | Game type classification |
| `sTeamColorNamesStrings` | const char* array[] | static | Team color display names |

## Key Functions / Methods

### BuildStringSet
- **Signature:** `static inline void BuildStringSet(short inStringSetID, const char** inStrings, short inNumStrings)`
- **Purpose:** Helper function that registers a single string set into the TextStrings repository
- **Inputs:** `inStringSetID` (set identifier), `inStrings` (string array), `inNumStrings` (count of strings)
- **Outputs/Return:** void
- **Side effects:** Calls `TS_PutCString()` for each string in the array; populates the TextStrings global repository
- **Calls:** `TS_PutCString()` (from TextStrings.h)
- **Notes:** Trivial loop; used via `BUILD_STRINGSET()` macro

### InitDefaultStringSets
- **Signature:** `void InitDefaultStringSets()`
- **Purpose:** Initialization entry point; populates the TextStrings repository with all default string sets at engine startup
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Registers 25+ string sets via `BUILD_STRINGSET()` macro calls; modifies global TextStrings state
- **Calls:** `BUILD_STRINGSET()` macro (which calls `BuildStringSet()`)
- **Notes:** Called once during engine initialization. String sets include error messages (128), filenames (129), UI labels (130ΓÇô131), network errors (132), key codes (133), preferences advice (134), computer interface text (135), join dialogs (136), weapon names (137ΓÇô138), preferences groupings (139), network game stats (140ΓÇô141), new join dialogs (142), progress strings (143), item/weapon names (150ΓÇô151), statistics strings (153), color picker prompts (200), and team color names (152).

## Control Flow Notes

**Initialization phase:** `InitDefaultStringSets()` is called once at engine startup to populate the TextStrings global registry with fallback strings. These strings are then accessed throughout the game via `TS_GetCString()` calls from other modules (UI, networking, game errors).

**No update/frame/render logic:** This file is purely data and initialization; it has no per-frame or per-render responsibilities.

## External Dependencies

- **TextStrings API** (TextStrings.h): `TS_PutCString()` ΓÇö registers strings into the global repository
- **network_dialogs.h**: Provides stringset ID constants (`kDifficultyLevelsStringSetID`, `kNetworkGameTypesStringSetID`, `kEndConditionTypeStringSetID`, `kSingleOrNetworkStringSetID`)
- **player.h**: Provides `kTeamColorsStringSetID` constant (defined as 152)
- **cseries.h**: Cross-platform compatibility headers
