# Source_Files/Network/ConnectPool.cpp

## File Purpose
Implements a non-blocking TCP connection pool for the Aleph One game engine. Manages up to 20 concurrent asynchronous connection attempts, each running on a background SDL thread to avoid blocking game logic during DNS resolution and TCP handshakes.

## Core Responsibilities
- Spawn and manage background threads for non-blocking TCP connections
- Resolve hostnames asynchronously using the network interface
- Maintain a fixed-size pool of reusable connection slots (kPoolSize = 20)
- Track connection state (Connecting, Connected, ResolutionFailed, ConnectFailed)
- Provide lazy cleanup of completed connections
- Release CommunicationsChannel objects to callers on successful connection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NonblockingConnect::Status` | enum | Tracks connection lifecycle state |
| `ConnectPool::m_pool[kPoolSize]` | array of pair | Stores connection slots with availability flags |

## Global / File-Static State
None.

## Key Functions / Methods

### NonblockingConnect::NonblockingConnect (address/port variant)
- Signature: `NonblockingConnect(const std::string& address, uint16 port)`
- Purpose: Create a connection attempt to a remote host via hostname
- Inputs: hostname/address string; port number
- Outputs/Return: None (constructor)
- Side effects: Spawns SDL thread; sets m_status = Connecting or ConnectFailed
- Calls: `connect()`
- Notes: Address resolution deferred to background thread

### NonblockingConnect::NonblockingConnect (IPaddress variant)
- Signature: `NonblockingConnect(const IPaddress& ip)`
- Purpose: Create a connection attempt to a resolved IP address
- Inputs: Pre-resolved IPaddress
- Outputs/Return: None (constructor)
- Side effects: Spawns SDL thread; sets m_status = Connecting or ConnectFailed
- Calls: `connect()`

### NonblockingConnect::Thread
- Signature: `int Thread()`
- Purpose: Main background thread entry point; performs DNS resolution and TCP connection
- Inputs: None (operates on member variables)
- Outputs/Return: int (thread exit code: 0 = success, 1 = resolution failed, 2 = connection failed)
- Side effects: Updates m_ip, m_status, and m_channel; calls external DNS/TCP operations
- Calls: `NetGetNetworkInterface()->resolve_address()`, `channel->connect()`, `channel->isConnected()`
- Notes: If m_ipSpecified is false, resolves hostname; creates CommunicationsChannel on heap; moves channel ownership to m_channel on success

### ConnectPool::connect (address/port variant)
- Signature: `NonblockingConnect* connect(const std::string& address, uint16 port)`
- Purpose: Allocate a pool slot and initiate a connection to hostname:port
- Inputs: hostname string; port number
- Outputs/Return: Pointer to NonblockingConnect (or nullptr if pool full)
- Side effects: Calls fast_free(); marks pool slot in-use; creates NonblockingConnect on heap
- Calls: `fast_free()`, `NonblockingConnect::NonblockingConnect()`
- Notes: Returns nullptr if no available slots

### ConnectPool::connect (IPaddress variant)
- Signature: `NonblockingConnect* connect(const IPaddress& ip)`
- Purpose: Allocate a pool slot and initiate a connection to a resolved IP
- Inputs: IPaddress structure
- Outputs/Return: Pointer to NonblockingConnect (or nullptr if pool full)
- Side effects: Calls fast_free(); marks pool slot in-use; creates NonblockingConnect on heap
- Calls: `fast_free()`, `NonblockingConnect::NonblockingConnect()`

### ConnectPool::abandon
- Signature: `void abandon(NonblockingConnect* nbc)`
- Purpose: Mark a connection as available for cleanup
- Inputs: Pointer to a NonblockingConnect in the pool
- Outputs/Return: None
- Side effects: Sets availability flag to true for matching slot
- Calls: None
- Notes: Called when caller is done with the connection; lazy cleanup happens on next connect call

### ConnectPool::fast_free
- Signature: `void fast_free()`
- Purpose: Clean up completed connections marked as available
- Inputs: None (operates on m_pool)
- Outputs/Return: None
- Side effects: Deletes NonblockingConnect objects; nulls out pointers
- Calls: `NonblockingConnect::done()`
- Notes: Checks both availability flag and done() before deleting

## Control Flow Notes
**Initialization:** ConnectPool singleton is instantiated on first `instance()` call, initializing m_pool with null pointers.

**Connection sequence:** 
1. Caller invokes `connect(address, port)` ΓåÆ searches for available slot
2. Creates NonblockingConnect ΓåÆ spawns SDL thread ΓåÆ returns immediately
3. Background thread resolves DNS (if needed) ΓåÆ attempts TCP connection ΓåÆ updates m_status
4. Caller polls `m_status` or calls `release()` once done() returns true
5. Caller invokes `abandon()` ΓåÆ marks slot available
6. Next `connect()` call triggers `fast_free()` ΓåÆ deletes and reuses slot

**No explicit frame/render participation;** purely asynchronous connection management.

## External Dependencies
- **SDL:** `SDL_CreateThread`, `SDL_WaitThread` (thread lifecycle)
- **CommunicationsChannel:** custom class for TCP I/O and connection management (defined elsewhere)
- **NetworkInterface:** `NetGetNetworkInterface()->resolve_address()` for DNS resolution (defined elsewhere)
- **Standard library:** `<string>`, `<memory>`, `<utility>`
- **Type definitions:** `uint16`, `IPaddress` (from cseries.h)
