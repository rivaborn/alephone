# tests/main.cpp
## File Purpose
Entry point for the Catch2 test runner. Parses game engine shell options from command-line arguments, filters them out, and passes remaining arguments to Catch2's test session runner.

## Core Responsibilities
- Parse engine-specific shell options before test execution
- Filter consumed arguments from argv to prevent Catch2 errors on unrecognized flags
- Initialize and run Catch2 test session with cleaned argument list
- Act as bootstrap between application configuration and test framework

## External Dependencies
- **Includes:**
  - `<catch2/catch_session.hpp>` ΓÇö Catch2 test framework
  - `"shell_options.h"` ΓÇö application configuration struct and parser
- **External symbols:** 
  - `ShellOptions::parse()` ΓÇö defined elsewhere (likely `shell_options.cpp`)
  - `Catch::Session` ΓÇö from Catch2 framework

# tests/replay_film_test.cpp
## File Purpose
Catch2-based test suite for validating replay film loading and playback. Tests verify that recorded game replays execute correctly and produce consistent random seeds. Includes utility mode to auto-seed malformed replay filenames.

## Core Responsibilities
- Load and validate replay files from a configurable directory tree
- Execute replays at maximum speed and verify seeded randomness consistency
- Extract random seeds from replay execution and verify matches filename expectations
- Provide tooling to auto-rename and seed replay files that lack proper naming convention
- Manage test-wide application lifecycle (init/shutdown)

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


