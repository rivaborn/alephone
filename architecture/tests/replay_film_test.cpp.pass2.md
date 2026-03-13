# tests/replay_film_test.cpp - Enhanced Analysis

## Architectural Role

This file is a **seam test** that validates the determinism contract of Aleph One's replay system by orchestrating the full application lifecycle (Shell), world state (GameWorld RNG), file I/O (Files), and graphics subsystems in concert. It serves as integration validation that replays can be loaded, executed at maximum speed, and produce identical random seedsΓÇöa critical invariant for competitive multiplayer and campaign replays. The test encodes expected RNG seeds in replay filenames as a lightweight assertion mechanism, avoiding the need for full replay binary comparisons.

## Key Cross-References

### Incoming (who depends on this file)
- **Catch2 test runner** (`tests/main.cpp`) ΓÇö invokes both TEST_CASE blocks via test macros
- **CI/build system** ΓÇö runs test suite to validate replay determinism before releases
- **Manual test execution** ΓÇö developers use `REPLAY_SET_SEED_FILENAME` mode to regenerate seeded replay files after engine changes

### Outgoing (what this file depends on)
- **Shell subsystem** (`shell.h`): lifecycle hooks (`initialize_application`, `shutdown_application`), document handling (`handle_open_document`), event loop orchestration (`main_event_loop`)
- **GameWorld subsystem** (`world.h`): random seed introspection (`get_random_seed`)
- **Files subsystem** (`FileHandler.h`, `crc.h`): cross-platform file abstraction (`FileSpecifier`), directory traversal (`dir_entry`, `ReadDirectory`), file type filtering (`_typecode_film`)
- **Interface/Graphics** (`interface.h`, `preferences.h`): replay speed control (`set_replay_speed`), graphics preferences mutation (`graphics_preferences->fps_target`)
- **Configuration** (`shell_options.h`): base directory paths (`shell_options.directory`, `shell_options.replay_directory`)

## Design Patterns & Rationale

1. **Conditional Compilation for Dual Modes**  
   The `#ifndef REPLAY_SET_SEED_FILENAME` guard creates two distinct behaviors: a production **validation test** (normal) and a maintenance **seed-generation utility** (set-seed). This avoids the complexity of a mutable test fixture or command-line flag while allowing developers to easily toggle between modes by redefining a single preprocessor symbol.

2. **Seam Testing via Public APIs**  
   Rather than mocking subsystems, the test directly invokes the public APIs of Shell, World, and FilesΓÇöexercising the true integration boundary. This validates that the *entire pipeline* preserves determinism, not just individual components in isolation.

3. **Filename as Assertion Contract**  
   Encoding the expected RNG seed in the filename (`name.SEED.filA`) shifts test data (expected values) into the filesystem, making test cases self-documenting and avoiding hard-coded arrays. However, this trades discoverability for implicit coupling to the filename convention.

4. **Recursive Directory Traversal as Utility**  
   Both `get_replays()` overloads use identical recursive logic (filtering by `_typecode_film`, processing subdirectories) but differ only in their success filter (seed present vs. absent). This mirrors the **Strategy pattern**: the filter function is implicit in the exception-based logic.

5. **Application Lifetime as Test Fixture**  
   `initialize_application()` and `shutdown_application()` are called once per test, not per replay. This amortizes initialization overhead but requires that replays don't corrupt persistent state (a strong implicit contract).

## Data Flow Through This File

**Normal Mode (Replay Validation):**
```
Filesystem (replay_directory/)
    Γåô [get_replays recursively scans]
Vector<Replay> (path + seed pairs)
    Γåô [initialize_application once]
    Γö£ΓåÆ [for each replay]
    Γöé   Γö£ΓåÆ handle_open_document(path)
    Γöé   Γö£ΓåÆ set_replay_speed(INT16_MAX)
    Γöé   Γö£ΓåÆ main_event_loop() [runs to completion]
    Γöé   Γö£ΓåÆ get_random_seed()
    Γöé   ΓööΓåÆ CHECK(seed == expected_seed)
    ΓööΓåÆ [shutdown_application]
```

**Set-Seed Mode (Maintenance):**
```
Filesystem (replay_directory/)
    Γåô [get_replays filters files lacking seed in name]
Vector<String> (unseeded replay paths)
    Γåô [initialize_application once]
    Γö£ΓåÆ [for each unseeded replay]
    Γöé   Γö£ΓåÆ handle_open_document(path)
    Γöé   Γö£ΓåÆ main_event_loop()
    Γöé   Γö£ΓåÆ get_random_seed() [captures actual RNG state at completion]
    Γöé   ΓööΓåÆ FileSpecifier::Rename(name.SEED.filA)
    ΓööΓåÆ [shutdown_application + commit to disk]
```

**Key State Transformations:**
- `graphics_preferences->fps_target = 60`: Ensures consistent frame timing (no variable FPS skew)
- `set_replay_speed(INT16_MAX)`: Skips rendering/audio, maximizes iteration speed during test
- `main_event_loop()` advances world state deterministically until replay concludes

## Learning Notes

1. **Determinism as an Engine Invariant**  
   The replay system relies on *fully deterministic* execution: same input replay + same RNG seed = identical world state. This constrains engine design: no uninitialized variables, no floating-point rounding errors in critical paths, no asynchronous updates that depend on wall-clock time.

2. **Filename Conventions Encode Metadata**  
   Rather than storing replay metadata (expected seed, version, author) in WAD headers or separate files, Aleph One embeds the seed in the filename. This is lightweight but brittle: renaming a replay breaks its metadata, and parsing is manual string manipulation.

3. **Replay Speed Control via Interface**  
   The `set_replay_speed(INT16_MAX)` call suggests replays are played back at variable speeds during normal gameplay but at maximum speed during testing. This implies the replay system does frame-by-frame iteration without real-time constraintsΓÇöa key assumption for deterministic execution.

4. **Application Lifetime Coupling**  
   Tests rely on `initialize_application()` / `shutdown_application()` to set up/tear down the world, preferencing, and file systems. This couples tests tightly to the Shell's initialization order, risking cascade failures if that order changes.

5. **Era-Specific Testing Approach**  
   This test predates modern approaches like property-based testing (e.g., Hypothesis) or fuzzing. Instead, it validates a fixed set of human-curated replay files, limiting coverage to scenarios developers thought to record.

## Potential Issues

1. **Overly Generic Exception Handling**  
   `get_seed_from_filename()` throws a bare `std::exception()` with no message. If seed extraction fails, the error is cryptic and difficult to diagnose. Better: `throw std::invalid_argument("Replay file lacks seed in expected format: " + file_name)`.

2. **No Validation of Loaded Replay**  
   `handle_open_document(replay.first)` can fail (return `false`), but the test uses `REQUIRE()` and continues blindly. If a replay file is corrupted, the test may run an uninitialized world and produce spurious seed mismatches.

3. **Destructive Filename Mutation**  
   The set-seed mode renames replay files in-place with no undo mechanism. If a developer runs set-seed mode twice with different engine versions, the first-pass seeds are lost. A safer approach: write seed metadata to a sibling `.seed` file.

4. **No Handling of Replay Execution Errors**  
   `main_event_loop()` is assumed to complete successfully, but if the replay data is malformed or the world state is inconsistent, the loop could crash, hang, or produce an undefined `get_random_seed()` value.

5. **Tight Coupling to Shell Initialization Order**  
   Tests rely on `shell_options` being initialized before the test runs. If Shell initialization order changes or shell_options is not populated, tests fail opaquely. No explicit validation that `shell_options.replay_directory` is valid before use.

6. **FPS Target Hardcoded for Testing**  
   Setting `graphics_preferences->fps_target = 60` assumes graphics prefs are initialized and mutable. If graphics initialization fails, this has no effect, silently invalidating the timing assumption the test relies on.
