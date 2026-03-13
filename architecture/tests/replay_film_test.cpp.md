# tests/replay_film_test.cpp

## File Purpose
Catch2-based test suite for validating replay film loading and playback. Tests verify that recorded game replays execute correctly and produce consistent random seeds. Includes utility mode to auto-seed malformed replay filenames.

## Core Responsibilities
- Load and validate replay files from a configurable directory tree
- Execute replays at maximum speed and verify seeded randomness consistency
- Extract random seeds from replay execution and verify matches filename expectations
- Provide tooling to auto-rename and seed replay files that lack proper naming convention
- Manage test-wide application lifecycle (init/shutdown)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Replay` | typedef (pair) | Encodes replay file path and expected RNG seed |
| `dir_entry` | struct (external) | Directory listing entry with name and metadata |
| `FileSpecifier` | class (external) | Cross-platform file/path abstraction |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `shell_options` | ShellOptions | extern | Provides base directory and replay_directory paths from shell config |
| `graphics_preferences` | graphics_preferences_data* | extern | Pointer to active graphics settings; fps_target modified for replay |

## Key Functions / Methods

### get_seed_from_filename
- **Signature:** `static uint16_t get_seed_from_filename(const std::string& file_name)`
- **Purpose:** Parse RNG seed from replay filename (expects format `name.SEED.filA`)
- **Inputs:** Filename string
- **Outputs/Return:** 16-bit unsigned seed value
- **Side effects:** Throws `std::exception` if seed not found (filename lacks required `.SEED.` pattern)
- **Calls:** `std::string::find_last_of()`, `std::string::substr()`, `stoi()`
- **Notes:** Assumes last dot separates extension; second-to-last dot marks seed boundary. No validation of seed range.

### get_replays (normal mode)
- **Signature:** `static std::vector<Replay> get_replays(std::string& directory_path)`
- **Purpose:** Recursively scan directory for all replay files with valid seeded names
- **Inputs:** Directory path by reference
- **Outputs/Return:** Vector of (path, seed) pairs
- **Side effects:** None (read-only filesystem traversal)
- **Calls:** `FileSpecifier::ReadDirectory()`, `FileSpecifier::IsDir()`, `FileSpecifier::GetType()`, `get_seed_from_filename()`
- **Notes:** Silently skips entries that fail seed extraction; filters by `_typecode_film`. Recurses into subdirectories.

### get_replays (set-seed mode)
- **Signature:** `static std::vector<std::string> get_replays(std::string& directory_path)` (alternate overload)
- **Purpose:** Find all replay files that **lack** a seed in their filename (for bulk seeding operation)
- **Inputs:** Directory path by reference
- **Outputs/Return:** Vector of paths to unseeded replay files
- **Side effects:** None (read-only filesystem traversal)
- **Calls:** Same as normal mode, except uses try-catch to **invert** filter logic
- **Notes:** Returns paths of files where `get_seed_from_filename()` throws (malformed names).

### set_replay_preferences
- **Signature:** `static void set_replay_preferences(void)`
- **Purpose:** Configure graphics settings for deterministic replay execution
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Sets `graphics_preferences->fps_target = 60`
- **Calls:** (none explicit; global assignment)
- **Notes:** Ensures consistent frame timing across replay runs.

### TEST_CASE("Film replay")
- **Signature:** `TEST_CASE("Film replay", "[Replay]")` (Catch2 macro)
- **Purpose:** Main replay validation test; loads and executes each replay, verifying RNG seed determinism
- **Inputs:** None (uses shell_options.replay_directory)
- **Outputs/Return:** Pass/fail per Catch2 assertion semantics
- **Side effects:** Initializes/shuts down the game application; loads game files; executes main event loop for each replay
- **Calls:** `initialize_application()`, `get_replays()`, `set_replay_preferences()`, `handle_open_document()`, `set_replay_speed()`, `main_event_loop()`, `get_random_seed()`, `shutdown_application()`
- **Notes:** Requires non-empty `shell_options.directory` and `shell_options.replay_directory`. Sets replay speed to max (INT16_MAX) for fast iteration. Each replay must exist and load without error.

### TEST_CASE("Film replay set seed")
- **Signature:** `TEST_CASE("Film replay set seed", "[Replay]")` (Catch2 macro, conditional on `REPLAY_SET_SEED_FILENAME`)
- **Purpose:** Utility test to auto-generate and embed RNG seeds in replay filenames
- **Inputs:** None (uses shell_options.replay_directory)
- **Outputs/Return:** Pass/fail per Catch2 assertion semantics
- **Side effects:** **Mutates filesystem**: renames replay files from `name.filA` ΓåÆ `name.SEED.filA` after execution
- **Calls:** `initialize_application()`, `get_replays()`, `handle_open_document()`, `set_replay_speed()`, `main_event_loop()`, `get_random_seed()`, `FileSpecifier::SplitPath()`, `FileSpecifier::Rename()`, `shutdown_application()`
- **Notes:** Only enabled via preprocessor define. Destructive: irreversibly renames files. Extracts seed by executing replay and capturing `get_random_seed()` at frame completion.

## Control Flow Notes

**Normal Mode (TEST_CASE "Film replay"):**
1. App init
2. Recursively load all replays from directory (filters by valid seed in name)
3. For each replay: open ΓåÆ set max playback speed ΓåÆ run main loop to completion ΓåÆ assert final RNG seed matches expected
4. App shutdown

**Set-Seed Mode (TEST_CASE "Film replay set seed"):**
1. App init
2. Recursively load all replays from directory (filters by **missing** seed in name)
3. For each malformed replay: open ΓåÆ play to completion ΓåÆ extract final seed ΓåÆ rename file to include seed
4. App shutdown

Both modes run one-shot after initialization; no per-frame interaction. Replay execution speed is maximized (INT16_MAX) to minimize test runtime.

## External Dependencies
- **Includes (engine systems):**
  - `shell.h`: Application lifecycle (`initialize_application()`, `shutdown_application()`, `main_event_loop()`, `handle_open_document()`)
  - `world.h`: Random number state (`set_random_seed()`, `get_random_seed()`, `global_random()`)
  - `FileHandler.h`: Cross-platform file I/O (`FileSpecifier`, `dir_entry`)
  - `shell_options.h`: Configuration struct (`ShellOptions`, replay_directory field)
  - `interface.h`: Replay control and graphics preferences access (`set_replay_speed()`, `graphics_preferences`)
  - `preferences.h`: Global preferences pointers
  - `catch2/catch_test_macros.hpp`: Testing framework

- **Extern symbols:**
  - `shell_options`: ShellOptions global set by shell initialization
  - `graphics_preferences`: Mutable pointer to active graphics config

- **Engine functions (signatures inferred from usage):**
  - `void initialize_application(void)`
  - `void shutdown_application(void)`
  - `bool handle_open_document(const std::string&)`
  - `void main_event_loop(void)`
  - `void set_replay_speed(short)` ΓÇö takes INT16_MAX for max speed
  - `uint16_t get_random_seed(void)`
