# Source_Files/Misc/alephversion.h

## File Purpose
Compile-time configuration header for the Aleph One game engine. Defines version strings, platform detection, and external service endpoints (update server, metaserver, leaderboard).

## Core Responsibilities
- Define application name, version, and release date
- Detect target platform (Windows, macOS, Linux, BSD variants) via preprocessor conditionals
- Provide platform-specific version strings and update URLs
- Configure external service endpoints (metaserver host, leaderboard, stats server)
- Compose a unified version display string

## Key Types / Data Structures
None.

## Global / File-Static State
None (all definitions are compile-time macros, not runtime state).

## Key Functions / Methods
None.

## Control Flow Notes
This file is pure configurationΓÇöincluded at compile time to embed version and platform constants into the binary. Not involved in frame/update/render/init/shutdown cycles.

## External Dependencies
- No includes or imports
- References external URLs as string constants (alephone.lhowon.org, metaserver.lhowon.org, stats.lhowon.org) but does not call them
- Uses platform-specific preprocessor symbols: `_WIN32`, `__APPLE__`, `__MACH__`, `linux`, `__NetBSD__`, `__OpenBSD__`
