# Subsystem Overview

## Purpose
The Steam parent shim process acts as a Steam API bridge between the Steamworks SDK and the game child process. It initializes the Steam SDK, spawns and manages the game as a child process, and relays Steam API calls, callbacks, and workshop operations through bidirectional pipes using a binary protocol.

## Key Files
| File | Role |
|------|------|
| steamshim_parent.cpp | Parent process that manages Steam SDK lifecycle, spawns child game process, and implements binary protocol command parsing and async callback relay |

## Core Responsibilities
- Spawn child game process with platform-specific mechanisms (Windows CreateProcessW / Unix fork+execvp) and establish bidirectional pipe communication
- Initialize and maintain Steamworks SDK interface pointers and state
- Parse and dispatch binary protocol commands received from child process to appropriate Steam API methods
- Handle Steamworks async callbacks and relay results back to child process
- Manage workshop item lifecycle operations (upload, update, query, delete)
- Persist achievement and statistics data through Steam UGC/UserStats APIs
- Monitor game overlay activation state
- Set up environment variables for child process integration with Steam runtime

## Key Interfaces & Data Flow
**Exposes to child process:**
- Binary protocol command/response channel via pipes
- Steam async callback results and relay mechanism
- Steam API state (achievements, stats, workshop, user data)

**Consumes from Steamworks SDK:**
- Steam API callbacks and async results
- Workshop item metadata and upload status
- User statistics and achievement states
- Game overlay events

## Runtime Role
- **Initialization**: Calls SteamAPI_InitEx(), sets up environment variables, spawns child process, establishes bidirectional pipes
- **Frame**: Continuously calls SteamAPI_RunCallbacks() to process async Steam events, parses incoming binary commands from child, dispatches relay responses
- **Shutdown**: Cleans up process handles and pipes on child process exit

## Notable Implementation Details
- Implements both Windows (CreatePipe/WriteFile/ReadFile) and POSIX (pipe/fork/execvp) code paths for cross-platform process spawning and I/O
- Uses binary protocol over pipes to avoid text serialization overhead for frequent child-parent communication
- Maintains Boost filesystem utilities for locating program binary location and workshop directory paths
- Manages callback queue internally and flushes results to child on polling intervals
