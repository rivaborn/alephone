# Source_Files/Files/preprocess_map_shared.cpp

## File Purpose
Provides a simplified auto-save wrapper for single-player games. The function delegates to the standard save mechanism, eliminating the need for dialog presentation and using standardized overwrite logic.

## Core Responsibilities
- Provide an entry point for auto-saving in single-player mode
- Delegate auto-save operations to the core `save_game()` function
- Maintain backward compatibility with callers expecting `save_game_full_auto()`

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### save_game_full_auto
- **Signature:** `bool save_game_full_auto(bool inOverwriteRecent)`
- **Purpose:** Perform an automatic game save without presenting a dialog to the player.
- **Inputs:** `inOverwriteRecent` (bool) ΓÇô parameter indicating whether to overwrite the most recent save (currently unused; present for API compatibility)
- **Outputs/Return:** Returns `bool` ΓÇô success/failure status from `save_game()`
- **Side effects:** Triggers game serialization to disk; may modify file system state if overwriting an existing save file
- **Calls:** `save_game()` (defined elsewhere)
- **Notes:** The `inOverwriteRecent` parameter is accepted but not forwarded to `save_game()`, suggesting it was intended for future use or legacy API compatibility. The function is now a thin wrapper with no additional logic.

## Control Flow Notes
This function would be invoked during gameplay state management when automatic saves are triggered (e.g., level transitions, checkpoints in campaign mode). It sits in the save/load pipeline as initiated by higher-level game state handlers.

## External Dependencies
- **Includes:** `interface.h` ΓÇô provides the declaration and (presumed) implementation of `save_game()`
- **Symbols used but defined elsewhere:** `save_game()` function
