# Source_Files/Network/network.h - Enhanced Analysis

## Architectural Role

This header is the **public API boundary** of the network subsystem, isolating the rest of the engine from transport layer details (TCP, UDP, message framing). It defines a clean state machine for multiplayer session lifecycle and hooks for UI integration, allowing GameWorld (`marathon2.cpp`), rendering systems, and the Shell to orchestrate multiplayer games without knowing about `CommunicationsChannel` or `Pinger` internals. The subsystem follows a clear host-gatherer / client-joiner topology, with special support for remote hub relay servers (NAT traversal) that diverged from the original peer-to-peer design.

## Key Cross-References

### Incoming (who depends on this)

- **GameWorld** (`Source_Files/GameWorld/marathon2.cpp`): Calls `NetSync()`, `NetStart()`, `NetProcessMessagesInGame()`, `NetDistributeGameDataToAllPlayers()`, `NetReceiveGameData()` to synchronize game state each frame and handle in-game messages
- **Shell/Interface** (`Source_Files/misc/interface.cpp`, preference dialogs): Calls `NetEnter()`, `NetExit()`, `NetGather()`, `NetGameJoin()`, `NetCancelJoin()`, `NetChangeColors()` for multiplayer setup flow
- **Rendering/UI** (dialog code): Registers `GatherCallbacks` and `ChatCallbacks` to display joining players and receive chat messages in real-time
- **Network diagnostics** (HUD, stats display): Polls `NetGetLatency()`, `NetGetStats()`, `NetState()`, `NetGetNumberOfPlayers()` for client-side UI feedback

### Outgoing (what this calls)

- **TCPMess subsystem** (`Source_Files/TCPMess/CommunicationsChannel.h`, `MessageInflater.h`): Used internally for TCP message transport; shared pointer ownership suggests handoff of channel management
- **Files subsystem** (`Source_Files/Files/wad.h`, game_wad.cpp): `NetDistributeGameDataToAllPlayers()` marshals map/physics via WAD format; `NetReceiveGameData()` unmarshals received WADs
- **Pinger** (`Source_Files/Network/Pinger.h`): Created via `NetCreatePinger()`, provides ICMP-based latency diagnostics
- **network_capabilities.h**: Runtime capability negotiation for protocol versioning (gameworld updates, Lua state, zipped data support)

## Design Patterns & Rationale

**1. Callback Pattern (Observer):**  
`GatherCallbacks` and `ChatCallbacks` are abstract base classes; UI code implements them and registers via `NetSetGatherCallbacks()` / `NetSetChatCallbacks()`. This decouples network event dispatch from UI concerns ΓÇö the network subsystem never depends on rendering or dialog code.

**2. State Machine with Pseudo-States:**  
The session lifecycle is a clean enumeration (`netGathering` ΓåÆ `netWaiting` ΓåÆ `netActive` ΓåÆ `netDown`), but `NetUpdateJoinState()` returns pseudo-states like `netPlayerAdded` or `netChatMessageReceived` **without assigning them to the true state**. This is an elegant pattern: callers get fine-grained event feedback for UI updates without polluting the actual state machine, avoiding separate event queues.

**3. Opaque Data Encapsulation:**  
`NetGather()` accepts `void* game_data` and `void* player_data` with sizes; the network subsystem treats them as blobs. This is C-style flexibility ΓÇö the engine defines its own `game_info` and `player_info` structs, but the network layer is agnostic to their format. Trades type safety for protocol independence.

**4. Separation of Concerns:**  
Network lifecycle (`NetEnter`/`NetExit`) is orthogonal to game state management (`NetSync`/`NetStart`). A multiplayer session can be "down" while the game world is still loaded, or vice versa.

## Data Flow Through This File

**Setup Phase (Host):**
```
NetEnter() ΓåÆ NetGather(game_data, player_data, ...) 
  ΓåÆ Opens socket, listens for joiners
  ΓåÆ NetCheckForNewJoiner() polled repeatedly
  ΓåÆ GatherCallbacks::JoiningPlayerArrived() invoked per joiner
  ΓåÆ NetGatherPlayer() called by UI to validate joiner
  ΓåÆ Transitions netGathering ΓåÆ netWaiting
```

**Setup Phase (Client):**
```
NetEnter() ΓåÆ NetGameJoin(player_data, host_address_string)
  ΓåÆ Connects to host, transitions netConnecting ΓåÆ netJoining
  ΓåÆ NetUpdateJoinState() polled, returns pseudo-states
  ΓåÆ Upon host's NetSync(), transitions to netWaiting then netStartingUp
```

**In-Game Sync:**
```
NetSync() ΓåÆ Validates topology, pre-game coordination
NetStart() ΓåÆ Game begins, transitions netStartingUp ΓåÆ netActive
NetProcessMessagesInGame() called per frame
  ΓåÆ Dispatches received messages to Lua/GameWorld
NetDistributeGameDataToAllPlayers() broadcasts map WAD
NetReceiveGameData() unmarshals remote WAD updates
NetGetUnconfirmedActionFlags() / NetUpdateUnconfirmedActionFlags()
  ΓåÆ Support client-side prediction: lookahead buffer for extrapolation
```

**Diagnostics:**
```
Pinger (if created) measures ICMP latency continuously
NetGetLatency() / NetGetStats() return per-player metrics
Used for rollback thresholds and client-side interpolation tuning
```

## Learning Notes

- **Callback-driven architecture was standard in 1994ΓÇô2003**: Synchronous invocation (no event queue) reflects the constraints of Mac OS and early network stacks. Modern engines would use async callbacks or event queues.
- **Opaque void* data**: The engine chose protocol flexibility over type safety. Compare to modern APIs that serialize to JSON or protobuf ΓÇö this avoids those marshalling costs.
- **Two-phase joiner acceptance**: `NetCheckForNewJoiner()` identifies candidates; `NetGatherPlayer()` validates them before topology commitment. This allows UI-driven filtering (e.g., skill level, color availability) without network retransmission.
- **Remote Hub as an afterthought**: The `use_remote_hub` flag and `NetConnectRemoteHub()` / `NetRemoteHubSendCommand()` suggest NAT traversal was grafted on post-hoc (note the V2 in the protocol string). Original design was likely pure P2P.
- **Pseudo-states as a clever compromise**: Rather than designing a separate event system, the authors reused the state enum to signal transient events. Avoids callback explosion for fine-grained events.

## Potential Issues

1. **Callback Lifetime**: Who owns the `GatherCallbacks` and `ChatCallbacks` pointers? If UI code is deallocated before `NetSetGatherCallbacks(nullptr)` is called, dangling pointers result. No mention of reference counting or weak pointers.

2. **Remote Hub Disconnection**: `NetConnectRemoteHub()` sets up a relay, but what happens if the remote hub crashes mid-game? No explicit error state or graceful degradation to P2P visible in the header.

3. **Type Safety of Opaque Data**: `game_data` and `player_data` blobs are serialized/deserialized by the network subsystem, but there's no versioning or schema validation ΓÇö corruption or version mismatch silently produces garbage.

4. **Unconfirmed Action Flags**: `NetGetUnconfirmedActionFlagsCount()` supports client-side prediction, but the header doesn't explain the rollback mechanism. Misprediction recovery likely happens in GameWorld, creating tight coupling.

5. **Gathering Flag Semantics**: `prospective_joiner_info::gathering` is unclear ΓÇö is it set by the host or the joiner? Used to distinguish "interested in joining" vs. "already gathered"?
