# Source_Files/Network/Pinger.h

## File Purpose
Defines the `Pinger` class for managing ICMP ping requests and measuring latency to registered IPv4 addresses. Part of the Aleph One game engine's network layer for health monitoring and connection quality assessment.

## Core Responsibilities
- Register IPv4 addresses for ping monitoring with unique identifiers
- Send ICMP ping requests to registered addresses with configurable retry and filtering
- Track ping transmission and response timestamps to measure round-trip latency
- Store incoming ping responses (called by network receive handler)
- Retrieve response times for all or specific registered addresses with timeout support

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `PingAddress` | struct (nested, private) | Encapsulates per-address state: IPv4, sent timestamp, received timestamp (atomic) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_ping_identifier_counter` | static uint16_t | static | Monotonic counter for generating unique ping request identifiers |
| `_registered_ipv4s` | unordered_map<uint16_t, PingAddress> | member | Maps ping identifier to registered address and timing state |

## Key Functions / Methods

### Register
- Signature: `uint16_t Register(const IPaddress& ipv4);`
- Purpose: Add an IPv4 address to the ping monitoring pool
- Inputs: IPv4 address to monitor
- Outputs/Return: Unique 16-bit identifier for this registered address
- Side effects: Inserts new entry into `_registered_ipv4s`, increments `_ping_identifier_counter`

### Ping
- Signature: `void Ping(uint8_t number_of_tries = 1, bool unpinged_addresses_only = false);`
- Purpose: Send ICMP echo requests to registered addresses
- Inputs: 
  - `number_of_tries`: Number of consecutive pings per address (default: 1)
  - `unpinged_addresses_only`: If true, skip addresses already pinged (smart throttling)
- Side effects: Generates network packets; updates `ping_sent_tick` for each pinged address
- Calls: (Implementation not visible; presumably calls UDP or raw socket API)

### GetResponseTime
- Signature: `std::unordered_map<uint16_t, uint16_t> GetResponseTime(uint16_t timeout_ms = 0);`
- Purpose: Retrieve round-trip latency measurements for all registered addresses
- Inputs: `timeout_ms` ΓÇö wait up to this duration for pending responses (0 = poll immediately)
- Outputs/Return: Map of {identifier ΓåÆ response_time_ms}
- Notes: Computes latency as `(pong_received_tick - ping_sent_tick)`; likely filters out zero/unresponsive addresses

### StoreResponse
- Signature: `void StoreResponse(uint16_t identifier, const IPaddress& address);`
- Purpose: Record receipt of an ICMP echo reply (called by network receive handler)
- Inputs: Ping identifier from response packet and source address (for validation)
- Side effects: Updates `pong_received_tick` atomically in the matching `PingAddress`
- Notes: Uses atomic store for thread-safe updates from async receive handler

## Control Flow Notes
Typically called in a per-frame or periodic network health check loop:
1. `Register()` addresses once at startup/when peers join
2. `Ping()` each frame or interval to probe connectivity
3. Network receive handler calls `StoreResponse()` when ICMP replies arrive (potentially different thread)
4. `GetResponseTime()` queried to display latency or make routing decisions

## External Dependencies
- `NetworkInterface.h`: Provides `IPaddress` type (wraps asio address/port)
- `<unordered_map>`: Hash map for O(1) response lookup by identifier
- `<atomic>`: `std::atomic_uint32_t` for lock-free thread-safe timestamp updates from async receive thread
- Conditioned on `!DISABLE_NETWORKING` preprocessor guard
