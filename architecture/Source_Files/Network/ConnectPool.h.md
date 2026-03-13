# Source_Files/Network/ConnectPool.h

## File Purpose
Provides a singleton pool for managing non-blocking outbound TCP connections in a multi-threaded context. Allows asynchronous connection establishment with status polling and connection reuse via pooling.

## Core Responsibilities
- Manage individual non-blocking TCP connection attempts (`NonblockingConnect`) with per-connection status tracking
- Provide thread-safe asynchronous connection establishment (address resolution + TCP connect)
- Maintain a singleton pool of reusable `NonblockingConnect` instances (max 20)
- Track connection lifecycle (Connecting ΓåÆ Connected or failed states)
- Release completed connections as `CommunicationsChannel` pointers for use by callers

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `NonblockingConnect::Status` | Enum | Connection state: Connecting, Connected, ResolutionFailed, ConnectFailed |
| `NonblockingConnect` | Class | Encapsulates a single non-blocking connection attempt with threading |
| `ConnectPool` | Class | Singleton manager for a pool of `NonblockingConnect` instances |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ConnectPool::m_instance` | `ConnectPool*` | Static (singleton) | Holds the single global instance |
| `ConnectPool::m_pool` | `std::pair<NonblockingConnect*, bool>[20]` | Instance member | Array of connection slots; `bool` false = in use, true = available |

## Key Functions / Methods

### NonblockingConnect::NonblockingConnect (address + port)
- **Signature:** `NonblockingConnect(const std::string& address, uint16 port)`
- **Purpose:** Construct a non-blocking connection attempt to a named host/IP and port.
- **Inputs:** Address string (hostname or IP), port number (host byte order).
- **Outputs/Return:** None (constructor).
- **Side effects:** Initializes `m_status` to `Connecting`, spawns connection thread via `SDL_Thread`.
- **Calls:** `connect()` (private helper).
- **Notes:** Address is stored; actual resolution happens in background thread.

### NonblockingConnect::NonblockingConnect (IPaddress)
- **Signature:** `NonblockingConnect(const IPaddress& ip)`
- **Purpose:** Construct a non-blocking connection to a pre-resolved IP address (skip DNS).
- **Inputs:** Prepared `IPaddress` struct.
- **Outputs/Return:** None (constructor).
- **Side effects:** Sets `m_ipSpecified = true`, spawns connection thread.
- **Calls:** `connect()` (private helper).

### NonblockingConnect::status()
- **Signature:** `Status status() { return m_status; }`
- **Purpose:** Query current connection state (poll for completion).
- **Inputs:** None.
- **Outputs/Return:** Current `Status` enum value.
- **Side effects:** None.

### NonblockingConnect::done()
- **Signature:** `bool done() { return m_status != Connecting; }`
- **Purpose:** Convenience predicate: connection attempt has completed (failed or succeeded).
- **Inputs:** None.
- **Outputs/Return:** `true` if not actively connecting.
- **Side effects:** None.

### NonblockingConnect::address()
- **Signature:** `const IPaddress& address()`
- **Purpose:** Retrieve the IP address (only valid after connection attempt completes).
- **Inputs:** None.
- **Outputs/Return:** Reference to `m_ip`.
- **Side effects:** None.
- **Notes:** Has assertions: caller must not call during `Connecting` or `ResolutionFailed` states.

### NonblockingConnect::release()
- **Signature:** `CommunicationsChannel* release()`
- **Purpose:** Extract the underlying communications channel and transfer ownership to caller.
- **Inputs:** None.
- **Outputs/Return:** Raw pointer to `CommunicationsChannel` (ownership transferred).
- **Side effects:** Clears internal `m_channel` via `unique_ptr::release()`.
- **Notes:** Assertion: only valid if `m_status == Connected`. Called to claim a successfully connected channel.

### ConnectPool::instance()
- **Signature:** `static ConnectPool* instance()`
- **Purpose:** Singleton accessor; creates instance on first call.
- **Inputs:** None.
- **Outputs/Return:** Pointer to global `ConnectPool` instance.
- **Side effects:** First call allocates and stores the singleton.
- **Notes:** Double-checked locking pattern (basic, not thread-safe in modern C++).

### ConnectPool::connect (address + port)
- **Signature:** `NonblockingConnect* connect(const std::string& address, uint16 port)`
- **Purpose:** Initiate a non-blocking connection from the pool.
- **Inputs:** Address string, port.
- **Outputs/Return:** Pointer to a `NonblockingConnect` from the pool (caller must poll).
- **Side effects:** Marks a pool slot as in-use; spawns background connection thread.
- **Notes:** Caller must later call `abandon()` to return the object to the pool.

### ConnectPool::connect (IPaddress)
- **Signature:** `NonblockingConnect* connect(const IPaddress& ip)`
- **Purpose:** Initiate a non-blocking connection to a pre-resolved address from the pool.
- **Inputs:** Prepared IP address.
- **Outputs/Return:** Pointer to a `NonblockingConnect` from the pool.
- **Side effects:** Marks a pool slot as in-use.

### ConnectPool::abandon()
- **Signature:** `void abandon(NonblockingConnect*)`
- **Purpose:** Return a connection object to the pool for reuse.
- **Inputs:** Pointer to `NonblockingConnect` obtained from `connect()`.
- **Outputs/Return:** None.
- **Side effects:** Marks the pool slot as available; likely cleans up the object.

## Control Flow Notes
- **Initialization phase:** Game/networking layer calls `ConnectPool::instance()->connect()` to initiate server connections asynchronously.
- **Main loop / update phase:** Caller polls `NonblockingConnect::done()` or `status()` in each frame.
- **Completion:** Once `status() == Connected`, caller calls `release()` to obtain the `CommunicationsChannel` for messaging, then `abandon()` to return the pool entry.
- **Non-blocking:** Actual resolution and TCP connect happen in background `SDL_Thread`, allowing the caller's thread to continue.

## External Dependencies
- **"cseries.h":** Platform abstraction, type definitions (`uint16`).
- **"CommunicationsChannel.h":** `CommunicationsChannel` class and `IPaddress` type.
- **SDL2:** `SDL_Thread` for background connection threads.
- **Standard library:** `<string>`, `<memory>` (for `std::unique_ptr`).
