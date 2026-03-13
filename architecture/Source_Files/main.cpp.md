# Source_Files/main.cpp

## File Purpose
Entry point for the Aleph One game engine (a Marathon source port). Prints startup credits and version information, parses command-line options, orchestrates application initialization and shutdown, and runs the main game loop with exception handling.

## Core Responsibilities
- Application entry point (main function)
- Print version banner and credits to console
- Parse command-line arguments via `ShellOptions`
- Invoke engine initialization and shutdown
- Execute main event loop (blocking)
- Handle and log uncaught exceptions gracefully
- Route command-line-specified files to document handler

## Key Types / Data Structures
None defined in this file.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| shell_options | ShellOptions | extern global | Singleton holding parsed CLI arguments and option flags |

## Key Functions / Methods

### main
- Signature: `int main(int argc, char** argv)`
- Purpose: Program entry point; orchestrates startup, runtime, and graceful shutdown
- Inputs: `argc` (argument count), `argv` (command-line arguments)
- Outputs/Return: Exit code (0 on success, 1 on unhandled exception)
- Side effects:
  - Writes banner to stdout (credits, version, platform info, licensing notes)
  - Parses CLI arguments into global `shell_options`
  - Calls `initialize_application()` to set up engine state
  - Calls `main_event_loop()` (blocking; runs until user quits)
  - Logs fatal exceptions via `logFatal()` macro
  - Calls `shutdown_application()` for cleanup
- Calls (visible in this file):
  - `expand_app_variables()` ΓÇö substitutes `$appLongVersion$` and similar macros in banner
  - `shell_options.parse()` ΓÇö parses argc/argv into ShellOptions
  - `initialize_application()` ΓÇö engine initialization (defined elsewhere)
  - `handle_open_document()` ΓÇö opens each file in `shell_options.files` (breaks on first success)
  - `main_event_loop()` ΓÇö main game update/render loop (defined elsewhere)
  - `shutdown_application()` ΓÇö cleanup (defined elsewhere)
  - `logFatal()` ΓÇö error logging macro
- Notes:
  - Exception-safe: catches both `std::exception` and generic `...` catches
  - Shutdown is guaranteed via outer try-catch even if initialization or loop throws
  - Multiple nested try-catch blocks ensure logging and shutdown are always attempted
  - Exit code is 1 if any exception occurs
  - Iterates through CLI files sequentially and stops at first successful open

## Control Flow Notes
Initialization ΓåÆ Main Loop ΓåÆ Shutdown pattern:
1. Print credits/version banner (conditionally platform-specific notes)
2. Parse command-line options into `shell_options`
3. Call `initialize_application()` (engine init; blocks until complete)
4. Iterate through CLI-specified files and attempt to open each
5. Enter `main_event_loop()` (blocks; only returns when game exits)
6. Call `shutdown_application()` (cleanup; always runs in outer try-catch)
7. Return exit code

All rendering, input handling, audio, game logic, and networking are delegated to functions defined elsewhere in the engine.

## External Dependencies
- **shell_options.h**: `ShellOptions` struct and `shell_options` extern global
- **shell.h**: `initialize_application()`, `main_event_loop()`, `shutdown_application()`, `handle_open_document()` function declarations
- **csstrings.h**: `expand_app_variables()` for string template substitution
- **Logging.h**: `logFatal()` macro and logging infrastructure
- **alephversion.h**: Version/platform macros (`A1_DISPLAY_VERSION`, `A1_DISPLAY_NAME`, `A1_HOMEPAGE_URL`, etc.)
- **SDL2/SDL_main.h**: SDL entry point (included but not explicitly used in this file)
- Standard library: `<cstdio>` (printf), `<string>`, `<vector>`, `<exception>`
