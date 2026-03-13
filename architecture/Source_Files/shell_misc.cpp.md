# Source_Files/shell_misc.cpp

## File Purpose
Miscellaneous shell utilities for the game engine, including global idle processing during event loops and memory allocation strategies for level transitions. Created to reduce code duplication across Mac and SDL ports.

## Core Responsibilities
- Manage idle-time processing for audio subsystems (Music, SoundManager)
- Provide specialized memory allocation for level transitions with progressive fallback recovery
- Global state for chat input mode
- Platform-agnostic game loop support

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `chat_input_mode` | bool | global | Tracks whether the game is in chat input mode |

## Key Functions / Methods

### global_idle_proc
- **Signature:** `void global_idle_proc(void)`
- **Purpose:** Called regularly during event loops to allow audio subsystems to perform idle-time work
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Music::instance()->Idle()` and `SoundManager::instance()->Idle()`
- **Calls:** `Music::instance()->Idle()`, `SoundManager::instance()->Idle()`
- **Notes:** Runs every frame/event loop tick; does not block; uses singleton accessor pattern

### level_transition_malloc
- **Signature:** `void *level_transition_malloc(size_t size)`
- **Purpose:** Allocate memory during level transitions, with automatic fallback recovery mechanisms when memory is tight
- **Inputs:** `size` ΓÇô bytes to allocate
- **Outputs/Return:** Allocated pointer, or NULL if allocation ultimately fails
- **Side effects:** If initial `malloc()` fails, unloads all sounds, then all collections, attempting allocation again after each recovery step
- **Calls:** `malloc()`, `SoundManager::instance()->UnloadAllSounds()`, `unload_all_collections()`
- **Notes:** Three-stage allocation: (1) try malloc directly, (2) unload sounds and retry, (3) unload all collections and retry. Returns NULL if all stages fail.

## Control Flow Notes
- `global_idle_proc()` is called every frame during event processing loops (`main_event_loop()` and equivalents)
- `level_transition_malloc()` is expected to be called during level loading/unloading transitions when memory pressure may be high
- The fallback mechanism is a form of garbage collection: sounds are lighter weight than full collections, so they're freed first

## External Dependencies
- **Notable includes:** `cseries.h` (platform compatibility), `Music.h`, `interface.h`, `world.h`, `screen.h`, `map.h`, `shell.h`, `preferences.h`, `vbl.h`, `player.h`, `items.h`, `TextStrings.h`, `InfoTree.h`
- **External symbols used but not defined here:** 
  - `SoundManager::instance()` ΓÇô singleton accessor (defined elsewhere, likely SoundManager.cpp)
  - `Music::instance()` ΓÇô singleton accessor (defined elsewhere, likely Music.cpp)
  - `unload_all_collections()` ΓÇô defined in shapes/collection loading code
  - `malloc()` ΓÇô standard C runtime
