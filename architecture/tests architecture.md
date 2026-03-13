# Subsystem Overview

## Purpose
Provides testing infrastructure for Aleph One via Catch2 test framework integration. Validates replay film loading, playback, and random seed consistency to ensure replay determinism across engine updates.

## Key Files
| File | Role |
|------|------|
| `tests/main.cpp` | Catch2 test runner entry point; parses and filters engine shell options before framework initialization |
| `tests/replay_film_test.cpp` | Replay film validation suite; loads, executes, and verifies replay seeding and determinism |

## Core Responsibilities
- Parse and filter engine-specific shell options from command-line arguments before Catch2 initialization
- Load replay files from configurable directory trees and validate format integrity
- Execute replays at maximum speed and verify random seed consistency against filename expectations
- Extract and validate random number state from replay execution
- Provide auto-seeding utility to rename and seed malformed replay files
- Manage test-wide application lifecycle (initialize/shutdown)

## Key Interfaces & Data Flow
**Consumes from:**
- `shell`: Application lifecycle (`initialize_application()`, `shutdown_application()`, `main_event_loop()`, `handle_open_document()`)
- `world`: Random number state (`set_random_seed()`, `get_random_seed()`, `global_random()`)
- `FileHandler`: Cross-platform file I/O (`FileSpecifier`, `dir_entry`)
- `shell_options`: Configuration struct with `replay_directory` field
- `interface`: Replay control (`set_replay_speed()`)
- `preferences`: Graphics preferences reference

**Exposes:**
- Test cases and validation results via Catch2 test session runner

## Runtime Role
- At engine startup: Parse and extract shell options before Catch2 processes command-line arguments
- During test execution: Initialize application, execute replay test suite at maximum playback speed, validate replay determinism
- At engine shutdown: Clean application state after test completion

## Notable Implementation Details
- Filters consumed engine arguments from `argv` to prevent Catch2 errors on unrecognized flags
- Sets replay speed to `INT16_MAX` to execute replays at maximum speed during test execution
- Validates replay consistency by comparing extracted random seeds against filename-embedded expectations
- Manages graphics and preferences state during replay validation
