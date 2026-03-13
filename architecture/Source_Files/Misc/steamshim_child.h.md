# Source_Files/Misc/steamshim_child.h

## File Purpose
Header file defining the interface between the game engine and a Steam API shim child process. Provides data structures for serializing/deserializing Steam Workshop items, achievements, stats, and event types for inter-process communication.

## Core Responsibilities
- Define enums for item types, content categories, event types, and upload status codes
- Declare serializable structs for Steam Workshop queries and game information
- Provide serialization/deserialization methods for binary IPC communication
- Declare public API functions for Steam initialization, stats, achievements, and Workshop operations
- Define event structure for polling Steam operation results

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ItemType` | enum class | Workshop item categories (Scenario, Plugin, Map, Physics, Script, Sounds, Shapes) |
| `ContentType` | enum class | Workshop content type groupings with range markers (START_PLUGIN, START_OTHER, etc.) |
| `steam_game_information` | struct | Game installation path and Workshop scenario support flag |
| `item_upload_data` | struct | Metadata for uploading a Workshop item (ID, types, paths, required scenario) |
| `item_owned_query_result` | struct | Batch result for owned Workshop items with title and compatibility info |
| `item_subscribed_query_result` | struct | Batch result for subscribed Workshop items with install paths |
| `STEAMSHIM_EventType` | enum | Event type codes (achievements, stats, Workshop uploads/queries, overlay, game info) |
| `STEAM_EItemUpdateStatus` | enum | Workshop item upload lifecycle states (preparing config ΓåÆ committing changes) |
| `STEAMSHIM_Event` | struct | Union-like event container with all possible result data (okay flag, int/float values, items, game info) |

## Global / File-Static State
None.

## Key Functions / Methods

### STEAMSHIM_pump
- Signature: `const STEAMSHIM_Event *STEAMSHIM_pump(void)`
- Purpose: Poll for next pending event from Steam operations
- Inputs: None
- Outputs/Return: Pointer to event struct (null if no events pending)
- Notes: Event pointer valid only until next pump call; main update loop function

### STEAMSHIM_init / STEAMSHIM_deinit
- Signature: `int STEAMSHIM_init(void)` / `void STEAMSHIM_deinit(void)`
- Purpose: Initialize/shutdown Steam API connection
- Return: Non-zero on successful init, zero on failure (deinit void)

### Workshop Query Functions
- `STEAMSHIM_queryWorkshopItemOwned(scenario_name)` ΓÇô Query owned items for scenario
- `STEAMSHIM_queryWorkshopItemMod(scenario_name)` ΓÇô Query subscribed mods for scenario
- `STEAMSHIM_queryWorkshopItemScenario()` ΓÇô Query all scenarios

Results returned asynchronously via `STEAMSHIM_pump()` with `SHIMEVENT_WORKSHOP_QUERY_*_RESULT` event types.

### Stats / Achievement Functions
- `STEAMSHIM_setStatI/F(name, value)` ΓÇô Set integer or float stat
- `STEAMSHIM_getStatI/F(name)` ΓÇô Query integer or float stat
- `STEAMSHIM_setAchievement(name, enable)` ΓÇô Unlock/lock achievement
- `STEAMSHIM_getAchievement(name)` ΓÇô Query achievement status
- `STEAMSHIM_storeStats()` ΓÇô Flush all stat changes to Steam
- `STEAMSHIM_resetStats(bAlsoAchievements)` ΓÇô Reset stats and optionally achievements

### Other Functions
- `STEAMSHIM_uploadWorkshopItem(item_upload_data)` ΓÇô Upload/update Workshop item
- `STEAMSHIM_getGameInfo()` ΓÇô Retrieve game installation folder and Workshop support flags
- `STEAMSHIM_alive()` ΓÇô Check if Steam connection is active

## Control Flow Notes
Request-response pattern via event pump:
1. Game calls a `STEAMSHIM_*()` function to request an operation
2. Game calls `STEAMSHIM_pump()` periodically (typically per frame) to retrieve completed events
3. Events in `STEAMSHIM_Event` contain result codes, values, and item batches from async operations

Initialization must succeed before calling other functions; teardown via `STEAMSHIM_deinit()`.

## External Dependencies
- `<sstream>`, `<string>`, `<vector>` ΓÇô Standard C++ libraries
- `uint8`, `uint64_t` ΓÇô Integer types (defined elsewhere, likely platform header)
- Steam API enums/types inferred but not included (e.g., `EResult` via `result_code` comment)
