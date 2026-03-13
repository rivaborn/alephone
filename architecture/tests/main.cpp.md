# tests/main.cpp

## File Purpose
Entry point for the Catch2 test runner. Parses game engine shell options from command-line arguments, filters them out, and passes remaining arguments to Catch2's test session runner.

## Core Responsibilities
- Parse engine-specific shell options before test execution
- Filter consumed arguments from argv to prevent Catch2 errors on unrecognized flags
- Initialize and run Catch2 test session with cleaned argument list
- Act as bootstrap between application configuration and test framework

## Key Types / Data Structures
None (uses types from external headers: `ShellOptions`, `Catch::Session`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `shell_options` | `ShellOptions` | extern (global) | Application-wide configuration parsed from argv |

## Key Functions / Methods

### main
- **Signature:** `int main(int argc, char* argv[])`
- **Purpose:** Test runner entry point; bridges shell option parsing and Catch2 test execution
- **Inputs:** 
  - `argc`: argument count
  - `argv`: argument vector (raw command-line args)
- **Outputs/Return:** Exit code from `Catch::Session().run()`
- **Side effects:** 
  - Parses and modifies global `shell_options` state
  - Constructs filtered `argv_catch` array (stack-allocated, max 64 args)
- **Calls:** 
  - `shell_options.parse(argc, argv, true)` ΓÇö parses engine options with unknown args ignored
  - `Catch::Session().run(argc_catch, argv_catch)` ΓÇö runs test framework
- **Notes:** 
  - `results` map keys are argv indices; value `false` or missing index means argument is passed to Catch2
  - Filters arguments in-place: skips any arg whose index is in `results` and marked `true`
  - Hard-coded 64-argument limit for Catch2 (stack buffer)

## Control Flow Notes
Initialization-only: runs once at program startup before any test discovery or execution. Acts as a preprocessing layerΓÇöshell options are consumed and removed from argv before Catch2 sees them, preventing test framework from failing on unrecognized flags like `--nosound` or `--debug`.

## External Dependencies
- **Includes:**
  - `<catch2/catch_session.hpp>` ΓÇö Catch2 test framework
  - `"shell_options.h"` ΓÇö application configuration struct and parser
- **External symbols:** 
  - `ShellOptions::parse()` ΓÇö defined elsewhere (likely `shell_options.cpp`)
  - `Catch::Session` ΓÇö from Catch2 framework
