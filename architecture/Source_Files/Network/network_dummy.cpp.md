# Source_Files/Network/network_dummy.cpp

## File Purpose

Provides stub/dummy implementations of the network subsystem API. Used when building without networking support or as a fallback, allowing the game to run in single-player mode while maintaining the same function signatures as the real network implementation.

## Core Responsibilities

- Stub synchronization functions for network game state
- Return safe single-player defaults for player/game queries
- Prevent network-specific cheats from being enabled
- Provide no-op cleanup and state management
- Maintain API compatibility with the real network module (network.cpp)

## Key Types / Data Structures

None.

## Global / File-Static State

None.

## Key Functions / Methods

### NetExit
- **Signature:** `void NetExit(void)`
- **Purpose:** Cleanup network subsystem on shutdown
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** None (no-op)
- **Calls:** None
- **Notes:** Dummy implementation; real version would close sockets and release resources

### NetSync
- **Signature:** `bool NetSync(void)`
- **Purpose:** Synchronize game state with network players before advancing frame
- **Inputs:** None
- **Outputs/Return:** `true` (always synchronized)
- **Side effects:** None
- **Calls:** None
- **Notes:** In real networking, would block until all players are ready; here always succeeds immediately

### NetGetLocalPlayerIndex
- **Signature:** `short NetGetLocalPlayerIndex(void)`
- **Purpose:** Get the local player's index in the player array
- **Inputs:** None
- **Outputs/Return:** `0` (always player 0 in single-player)
- **Side effects:** None
- **Calls:** None
- **Notes:** Fixed to 0; real version would track position in multiplayer topology

### NetGetNumberOfPlayers
- **Signature:** `short NetGetNumberOfPlayers(void)`
- **Purpose:** Query total number of active players in the game
- **Inputs:** None
- **Outputs/Return:** `1` (single player only)
- **Side effects:** None
- **Calls:** None
- **Notes:** Enforces single-player-only constraint

### NetGetPlayerData / NetGetGameData
- **Signature:** `void *NetGetPlayerData(short player_index)` / `void *NetGetGameData(void)`
- **Purpose:** Retrieve serialized player or game state for network transmission
- **Inputs:** `player_index` (ignored)
- **Outputs/Return:** `NULL`
- **Side effects:** None
- **Calls:** None
- **Notes:** No network serialization in dummy mode

### network_gather / network_join
- **Signature:** `bool network_gather(void)` / `int network_join(void)`
- **Purpose:** Initiate gathering of players or joining an existing game
- **Inputs:** None
- **Outputs/Return:** `false` (operations fail)
- **Side effects:** None
- **Calls:** None
- **Notes:** Networking disabled; these UI flow functions are not available

### NetAllowBehindview / NetAllowCrosshair / NetAllowTunnelVision
- **Signature:** `bool NetAllow*(void)`
- **Purpose:** Check if network game rules permit cheat overlays/modes
- **Inputs:** None
- **Outputs/Return:** `false` (all cheats disabled)
- **Side effects:** None
- **Calls:** None
- **Notes:** Even in single-player mode (this dummy), cheats are gated through the network subsystem API; real network would enforce these per game_data flags

### display_net_game_stats / current_game_has_balls / NetChangeMap / NetGetNetTime
- **Purpose:** Placeholder implementations for game state queries and stats display
- **Outputs/Return:** Empty or zero values
- **Notes:** Summarized here; each is a trivial stub returning safe defaults

## Control Flow Notes

This file is part of the **initialization and per-frame update cycle**:
- **Startup:** `NetExit()` stub ensures no cleanup errors
- **Gather phase:** `network_gather()` and `network_join()` return `false`, forcing single-player path
- **Per-frame:** `NetSync()` always returns `true`, allowing game loops to proceed without waiting for other players
- **Shutdown:** `NetExit()` (no-op) completes cleanly

When real networking is disabled, the game engine calls these dummy functions instead of network.cpp's real implementations, allowing the same build to support both networked and non-networked modes.

## External Dependencies

- **Included headers:**
  - `cseries.h` ΓÇö Base types and platform definitions
  - `map.h` ΓÇö `entry_point` struct definition (for `NetChangeMap` parameter)
  - `network.h` ΓÇö Function declarations / API contract
  - `network_games.h` ΓÇö Game-mode-specific declarations
- **Defined elsewhere:** All structures and game state globals referenced in declarations (e.g., `entry_point`)
