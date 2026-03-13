# Source_Files/Network/SSLP_limited.cpp

## File Purpose
Implements the Simple Service Location Protocol (SSLP) for discovering and advertising network services in Aleph One. Uses a non-threaded, pump-based design where the application's main thread periodically calls `SSLP_Pump()` to perform discovery and advertisement work. Designed to be lightweight, with automatic timeout and garbage collection of stale service entries.

## Core Responsibilities
- **Service Discovery**: Locate remote services by broadcasting FIND packets and processing HAVE responses
- **Service Advertisement**: Respond to discovery requests and allow local services to be found by peers
- **Packet Processing**: Receive, parse, and interpret SSLP protocol messages (FIND, HAVE, LOST)
- **Instance Tracking**: Maintain linked list of discovered service instances with timestamp-based timeout
- **Callback Notification**: Invoke registered callbacks when services are found, lost, or renamed
- **Resource Management**: Initialize/shutdown UDP socket and clean up found instances

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SSLPint_FoundInstance` | struct | Local tracking of discovered services with timestamp; forms linked list |
| `SSLP_ServiceInstance` | struct (from API) | Public-facing service instance with type, name, and address |
| `SSLP_Packet` | struct (from protocol) | Wire format for UDP datagrams (magic, version, message type, port, service metadata) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sBehaviorsDesired` | int | static | Flags (`SSLPINT_LOCATING`, `SSLPINT_RESPONDING`) tracking which modes are active |
| `sSocketDescriptor` | `std::unique_ptr<UDPsocket>` | static | UDP socket for all multicast/broadcast communication |
| `sReceivingPacket` | `UDPpacket` | static | Buffer for incoming packets |
| `sFindPacket` | `UDPpacket` | static | Pre-formatted FIND broadcast packet template |
| `sResponsePacket` | `UDPpacket` | static | Pre-formatted HAVE response packet template |
| `sFoundInstances` | `SSLPint_FoundInstance*` | static | Linked-list head of discovered service instances |
| `sFoundCallback`, `sLostCallback`, `sNameChangedCallback` | function pointers | static | Client callbacks for service lifecycle events |

## Key Functions / Methods

### SSLP_Pump
- **Signature:** `void SSLP_Pump()`
- **Purpose:** Main pump function called by application to process discovery and advertisement work
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Broadcasts FIND packets (every 5 sec if in LOCATING mode), receives and processes incoming packets, removes timed-out instances
- **Calls:** `sSocketDescriptor->broadcast_send()`, `sSocketDescriptor->check_receive()`, `sSocketDescriptor->receive()`, `SSLPint_ReceivedPacket()`, `SSLPint_RemoveTimedOutInstances()`, `machine_tick_count()`
- **Notes:** Early-exit if `sBehaviorsDesired == SSLPINT_NONE`; throttles FIND broadcasts to 5-second intervals to avoid flooding

### SSLP_Locate_Service_Instances
- **Signature:** `void SSLP_Locate_Service_Instances(const char* inServiceType, callback, callback, callback)`
- **Purpose:** Start locating service instances of a given type
- **Inputs:** Service type string, three optional callback function pointers (found, lost, name-changed)
- **Outputs/Return:** None
- **Side effects:** Sets `sBehaviorsDesired |= SSLPINT_LOCATING`, initializes UDP socket if not already open, prepares FIND packet template
- **Calls:** `SSLPint_Enter()`, `PackPacket()`
- **Notes:** Asserts that no other location is in progress; only one service type can be searched at a time

### SSLP_Stop_Locating_Service_Instances
- **Signature:** `void SSLP_Stop_Locating_Service_Instances(const char* inServiceType)`
- **Purpose:** Stop locating services (argument ignored in this limited implementation)
- **Inputs:** Service type (ignored)
- **Outputs/Return:** None
- **Side effects:** Clears `SSLPINT_LOCATING` flag, nulls callbacks, flushes found instances, exits SSLP if no other mode active
- **Calls:** `SSLPint_FlushAllFoundInstances()`, `SSLPint_Exit()`
- **Notes:** All discovered `SSLP_ServiceInstance` pointers become invalid after this call

### SSLP_Allow_Service_Discovery
- **Signature:** `void SSLP_Allow_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance)`
- **Purpose:** Advertise a local service for discovery by peers
- **Inputs:** Service instance with type, name, and port
- **Outputs/Return:** None
- **Side effects:** Sets `sBehaviorsDesired |= SSLPINT_RESPONDING`, initializes UDP socket if needed, prepares HAVE response packet, broadcasts one HAVE packet immediately
- **Calls:** `SSLPint_Enter()`, `PackPacket()`, `sSocketDescriptor->broadcast_send()`
- **Notes:** Only one service instance can be advertised at a time

### SSLP_Disallow_Service_Discovery
- **Signature:** `void SSLP_Disallow_Service_Discovery(const SSLP_ServiceInstance* inInstance)`
- **Purpose:** Stop advertising a service
- **Inputs:** Instance pointer (ignored in limited implementation)
- **Outputs/Return:** None
- **Side effects:** Clears `SSLPINT_RESPONDING` flag, broadcasts a LOST packet as courtesy, exits SSLP if no other mode active
- **Calls:** `UnpackPacket()`, `PackPacket()`, `sSocketDescriptor->broadcast_send()`, `SSLPint_Exit()`
- **Notes:** Argument ignored; stops the single advertised service

### SSLPint_ReceivedPacket
- **Signature:** `static void SSLPint_ReceivedPacket()`
- **Purpose:** Process a single received SSLP packet (FIND, HAVE, or LOST)
- **Inputs:** Implicitly reads from `sReceivingPacket` global
- **Outputs/Return:** None
- **Side effects:** Validates packet magic/version, dispatches to message-type handler, may invoke client callbacks or send responses
- **Calls:** `UnpackPacket()`, message-type handlers for FIND/HAVE/LOST, `SSLPint_FoundAnInstance()`, `SSLPint_LostAnInstance()`, `sSocketDescriptor->send()`
- **Notes:** Validates fixed header fields and drops malformed packets; compares service types as null-terminated strings

### SSLPint_FoundAnInstance
- **Signature:** `static SSLP_ServiceInstance* SSLPint_FoundAnInstance(SSLP_ServiceInstance* inInstance)`
- **Purpose:** Record discovery of a service or update an existing entry
- **Inputs:** Service instance structure (temporary copy from packet)
- **Outputs/Return:** Pointer to newly-added instance (caller's responsibility is passed to it), or NULL if already known
- **Side effects:** Searches linked list; if new, allocates `SSLPint_FoundInstance` and `SSLP_ServiceInstance`, prepends to list, updates timestamp; if existing, updates timestamp and name if changed
- **Calls:** `logTrace()`, `strncmp()`, `strncpy()`, `machine_tick_count()`, `malloc()`, callback `sNameChangedCallback()`
- **Notes:** Caller must free returned instance on lifecycle end; comparison is by address only (service_type not checked if multiple types supported)

### SSLPint_LostAnInstance
- **Signature:** `static SSLP_ServiceInstance* SSLPint_LostAnInstance(SSLP_ServiceInstance* inInstance)`
- **Purpose:** Remove a service from tracking
- **Inputs:** Service instance (used for address lookup)
- **Outputs/Return:** Pointer to removed instance (caller must free), or NULL if not found
- **Side effects:** Searches linked list, unlinks and frees `SSLPint_FoundInstance` wrapper, returns the wrapped `SSLP_ServiceInstance`
- **Calls:** `logTrace()`, `malloc()`
- **Notes:** Symmetric to `SSLPint_FoundAnInstance`; caller responsible for notifying callbacks and freeing returned instance

### SSLPint_RemoveTimedOutInstances
- **Signature:** `static void SSLPint_RemoveTimedOutInstances()`
- **Purpose:** Garbage-collect service instances that have not been heard from recently
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Iterates linked list, removes entries older than `SSLPINT_INSTANCE_TIMEOUT` (20 seconds), invokes `sLostCallback()` for each removed instance, frees memory
- **Calls:** `machine_tick_count()`, `logContext()`, callback `sLostCallback()`, `free()`
- **Notes:** Called during `SSLP_Pump()` in LOCATING mode; maintains two pointers to safely unlink while iterating

### SSLPint_Enter / SSLPint_Exit
- **Signature:** `static bool SSLPint_Enter()` / `static int SSLPint_Exit()`
- **Purpose:** Initialize and shut down SSLP subsystem
- **Inputs:** None
- **Outputs/Return:** `SSLPint_Enter()` returns success/failure; `SSLPint_Exit()` returns 0
- **Side effects:** `Enter` opens UDP socket on SSLP_PORT, enables broadcast flag; `Exit` closes socket
- **Calls:** `NetGetNetworkInterface()->udp_open_socket()`, `sSocketDescriptor->broadcast()`
- **Notes:** Reference-counted via `sBehaviorsDesired`; only first call to Enter creates socket, only last call to Exit destroys it

## Control Flow Notes
- **Initialization:** Client calls `SSLP_Locate_Service_Instances()` or `SSLP_Allow_Service_Discovery()` ΓåÆ `SSLPint_Enter()` opens socket
- **Main Loop:** Application calls `SSLP_Pump()` regularly (ideally ~1/sec, must be Γëñ5 sec intervals)
  - If LOCATING: broadcasts FIND every 5 sec, prunes timed-out instances
  - Always: drains incoming packet queue, processes each via `SSLPint_ReceivedPacket()`
- **Callbacks:** Invoked from `SSLP_Pump()` context (same thread as caller)
- **Shutdown:** Client calls `SSLP_Stop_Locating_Service_Instances()` or `SSLP_Disallow_Service_Discovery()` ΓåÆ `SSLPint_Exit()` closes socket if no other mode active

## External Dependencies
- **SDL2:** `SDL_SwapBE32()` for endian conversion, `SDL_thread.h` (included but not used in limited implementation)
- **Network Interface:** `NetGetNetworkInterface()`, `UDPsocket` abstraction, `UDPpacket`, `IPaddress` with `.address()` and `.port()` methods
- **Logging:** `logTrace()`, `logContext()`, `logNote()` macros from `Logging.h`
- **Time:** `machine_tick_count()` from `csmisc.h` (millisecond precision)
- **C Standard:** `memcpy()`, `strncpy()`, `strncmp()`, `malloc()`, `free()`
