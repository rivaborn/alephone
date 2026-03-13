# Source_Files/Network/network_udp.cpp

## File Purpose
Implements UDP socket communication layer for the Aleph One game engine, providing a thin wrapper around UDP sockets with an asynchronous receiving thread. Corresponds functionally to AppleTalk DDP on legacy systems. Handles opening/closing sockets, dispatching received packets via callback, and sending frames to remote machines.

## Core Responsibilities
- Create and manage a UDP socket bound to a configurable port
- Spawn and manage an asynchronous receiving thread that polls for incoming packets
- Dispatch received packets to a registered callback handler (with mutex synchronization)
- Validate and send UDP frames to remote addresses
- Cleanly shutdown socket and thread on close

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| UDPsocket | class (defined elsewhere) | Wraps UDP socket operations; supports async send/receive |
| UDPpacket | struct (defined elsewhere) | Contains packet data, size, and destination address |
| IPaddress | struct (defined elsewhere) | Network address for routing packets |
| PacketHandlerProcPtr | typedef (defined elsewhere) | Function pointer type for packet dispatch callbacks |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sSocket | std::unique_ptr<UDPsocket> | static | Holds the single UDP socket instance |
| sPacketHandler | PacketHandlerProcPtr | static | Registered callback invoked when packets arrive |
| sKeepListening | std::atomic_bool | static | Signal flag for receiving thread to continue/exit |
| sReceivingThread | SDL_Thread* | static | Handle to the asynchronous packet listener thread |

## Key Functions / Methods

### receive_thread_function
- **Signature:** `static int receive_thread_function(void*)`
- **Purpose:** Thread entry point; continuously polls for incoming UDP packets and invokes the packet handler
- **Inputs:** Unused void pointer (SDL thread convention)
- **Outputs/Return:** 0 (thread exit code)
- **Side effects:** Calls sPacketHandler under mutex protection; modifies packet state in sSocket
- **Calls:** `sSocket->receive_async()`, `sSocket->register_receive_async()`, `take_mytm_mutex()`, `sPacketHandler()`, `release_mytm_mutex()`
- **Notes:** Loops while `sKeepListening` is true; uses 1000ms timeout on receive; takes mutex only *after* receiving to avoid blocking I/O under lock

### NetDDPOpenSocket
- **Signature:** `bool NetDDPOpenSocket(uint16_t ioPortNumber, PacketHandlerProcPtr packetHandler)`
- **Purpose:** Initializes UDP socket on specified port and starts listening thread
- **Inputs:** Port number (network byte order), packet handler callback
- **Outputs/Return:** `true` on success; `false` if socket creation fails
- **Side effects:** Creates socket via network interface; spawns SDL thread; sets all static state variables; attempts thread priority boost
- **Calls:** `NetGetNetworkInterface()->udp_open_socket()`, `SDL_CreateThread()`, `BoostThreadPriority()`, `fdprintf()`
- **Notes:** Assigns `sPacketHandler` twice (redundant); emits warning if priority boost fails but continues anyway

### NetDDPCloseSocket
- **Signature:** `bool NetDDPCloseSocket()`
- **Purpose:** Gracefully shuts down socket and receiving thread
- **Inputs:** None
- **Outputs/Return:** `true`
- **Side effects:** Signals thread exit via `sKeepListening`; waits for thread termination; closes and destroys socket
- **Calls:** `SDL_WaitThread()`, `sSocket.reset()`
- **Notes:** Uses atomic bool for thread-safe signaling; always returns true (error path not modeled)

### NetDDPSendFrame
- **Signature:** `bool NetDDPSendFrame(UDPpacket& frame, const IPaddress& address)`
- **Purpose:** Sends a UDP frame to remote address
- **Inputs:** Packet (reference, modified); destination address
- **Outputs/Return:** `true` if send succeeded, `false` otherwise
- **Side effects:** Sets `frame.address` before transmission
- **Calls:** `sSocket->send()`, `assert()`
- **Notes:** Validates frame size against `ddpMaxData` constant; address parameter allows per-packet routing

## Control Flow Notes
**Initialization:** `NetDDPOpenSocket()` is called once at network startup to open socket and spawn listener thread.  
**Runtime:** Receiving thread runs continuously in background, invoking packet handler under mutex protection. Send operations are asynchronous and independent.  
**Shutdown:** `NetDDPCloseSocket()` signals thread exit, waits for completion, then closes socket. Must be called before cleanup.

## External Dependencies
- **SDL2/SDL_thread.h** ΓÇô Thread creation and management
- **thread_priority_sdl.h** ΓÇô `BoostThreadPriority()` for elevating listener thread priority
- **cseries.h** ΓÇô Utility macros (e.g., `fdprintf`)
- **network_private.h** ΓÇô Network constants and type definitions
- **mytm.h** ΓÇô `take_mytm_mutex()`, `release_mytm_mutex()` for synchronization
- **Defined elsewhere:** `NetGetNetworkInterface()`, `UDPsocket` class, `ddpMaxData` constant, `UDPpacket`, `IPaddress`
