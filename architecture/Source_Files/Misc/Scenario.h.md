# Source_Files/Misc/Scenario.h

## File Purpose
Declares the `Scenario` singleton class for managing scenario metadata and compatibility information. Provides access to scenario name, version, ID, and classic gameplay support status in the Aleph One game engine.

## Core Responsibilities
- Expose singleton instance of active scenario data
- Store and retrieve scenario metadata (name, version, ID) with field-length truncation
- Manage scenario version compatibility tracking
- Control classic gameplay mode enablement
- Serve as the public interface for scenario properties

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Scenario | class | Singleton managing scenario metadata and compatibility |
| InfoTree | class (forward declared) | Parse tree structure; defined elsewhere |

## Global / File-Static State
None explicit in header; implicit singleton state managed via `instance()` method.

## Key Functions / Methods

### instance
- Signature: `static Scenario *instance()`
- Purpose: Singleton accessor; returns the unique active scenario instance
- Outputs/Return: Pointer to static Scenario instance
- Notes: Classic implementation; caller owns no memory

### GetName / SetName
- Signature: `const string GetName()` / `void SetName(const string name)`
- Purpose: Query and update scenario name
- Side effects: SetName truncates input to 31 characters
- Notes: Getter returns by value; setter copies truncated substring

### GetVersion / SetVersion
- Signature: `const string GetVersion()` / `void SetVersion(const string version)`
- Purpose: Query and update scenario version string
- Side effects: SetVersion truncates input to 7 characters
- Notes: Version likely follows semantic versioning or custom format

### GetID / SetID
- Signature: `const string GetID()` / `void SetID(const string id)`
- Purpose: Query and update unique scenario identifier
- Side effects: SetID truncates input to 23 characters

### IsCompatible / AddCompatible
- Signature: `bool IsCompatible(const string)` / `void AddCompatible(const string)`
- Purpose: Check if a version string is in the compatibility list; add new compatible version
- Inputs: Version string
- Notes: Maintains vector of compatible version strings

### AllowsClassicGameplay / SetAllowsClassicGameplay
- Signature: `bool AllowsClassicGameplay() const` / `void SetAllowsClassicGameplay(bool allow)`
- Purpose: Query and set classic gameplay mode support flag
- Notes: Initialized to `false` in constructor

## Control Flow Notes
Likely called during game initialization when parsing scenario/map files. `parse_mml_scenario()` and `reset_mml_scenario()` populate and clear singleton data respectively.

## External Dependencies
- `<string>`, `<vector>` (STL containers)
- `InfoTree` (forward declared; defined elsewhere, used in `parse_mml_scenario`)
- GPL v3 licensing (per header comment)
