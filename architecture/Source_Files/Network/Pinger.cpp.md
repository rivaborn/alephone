# Source_Files/Network/Pinger.cpp

## File Purpose
Implements network ping functionality to measure latency to registered IP addresses. Sends UDP-based ping requests and collects response times with timeout support. Part of the Aleph One game engine's networking subsystem for connection quality monitoring.

## Core Responsibilities
- Register IPv4 addresses for ping monitoring with unique identifiers
- Send UDP ping request packets with configurable retry counts
- Store ping timing metadata (sent tick, received tick)
- Collect and return response time measurements with timeout
- Thread-safe access via mutex protection during packet transmission

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PingAddress | struct (in header) | Per-address state: IPaddress, ping_sent_tick (uint64_t), pong_received_tick (atomic_uint32_t) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _ping_identifier_counter | static uint16_t | static | Monotonic counter for unique ping request IDs |

## Key Functions / Methods

### Register
- Signature: `uint16_t Register(const IPaddress& ipv4)`
- Purpose: Register an IP address to be pinged; returns a unique identifier for later correlation.
- Inputs: `ipv4` ΓÇô target IP address
- Outputs/Return: Unique uint16_t identifier (pre-incremented counter)
- Side effects: Inserts into `_registered_ipv4s` map; increments `_ping_identifier_counter`
- Calls: `std::unordered_map::emplace()`
- Notes: Non-thread-safe registration; assumes called before Ping/GetResponseTime.

### Ping
- Signature: `void Ping(uint8_t number_of_tries, bool unpinged_addresses_only)`
- Purpose: Construct and send UDP ping request packets to all (or only unpinged) registered addresses.
- Inputs: `number_of_tries` (ΓëÑ1 after clamping), `unpinged_addresses_only` (skip if already sent)
- Outputs/Return: None
- Side effects: Sets `ping_sent_tick` on each address; updates ticks; sends packets via UDP; swallows all exceptions.
- Calls: `AOStreamBE::operator<<()`, `calculate_data_crc_ccitt()`, `take_mytm_mutex()`, `NetDDPSendFrame()`, `release_mytm_mutex()`, `machine_tick_count()`
- Notes: Serializes ping ID into packet payload; writes CRC to bytes [2ΓÇô3]; mutex held during transmission only; exception-safe (tryΓÇôcatch all).

### GetResponseTime
- Signature: `std::unordered_map<uint16_t, uint16_t> GetResponseTime(uint16_t timeout_ms)`
- Purpose: Poll for ping responses with optional timeout; return response times (ms) indexed by ping identifier.
- Inputs: `timeout_ms` (0 = no timeout, block until all responses or indefinite)
- Outputs/Return: Map of identifier ΓåÆ response time in ticks; unmeasured addresses return `UINT16_MAX`.
- Side effects: Calls `machine_tick_count()`, `sleep_for_machine_ticks(1)` in loop; reads `pong_received_tick` (atomic).
- Calls: `std::unordered_map::find()`, `std::unordered_map::insert()`, `machine_tick_count()`, `sleep_for_machine_ticks()`
- Notes: Busy-waits with 1-tick sleep; returns early if all addresses responded; fills missing responses with `UINT16_MAX` at end.

### StoreResponse
- Signature: `void StoreResponse(uint16_t identifier, const IPaddress& address)`
- Purpose: Record receipt of a ping response; called (presumably) by network receive handler when pong arrives.
- Inputs: `identifier` (ping request ID), `address` (sender's IP)
- Outputs/Return: None
- Side effects: Sets `pong_received_tick` (atomic) on matching address if not yet received and IP matches.
- Calls: `std::unordered_map::find()`, `machine_tick_count()`
- Notes: Address verification prevents spoofed responses; atomic write prevents data race; only accepts first response per ping.

## Control Flow Notes
**Initialization ΓåÆ Ping ΓåÆ Response ΓåÆ Query**
1. `Register()` populates `_registered_ipv4s` before any pinging.
2. `Ping()` is called to initiate requests; records `ping_sent_tick`.
3. Network receive thread calls `StoreResponse()` when a pong arrives (asynchronous).
4. `GetResponseTime()` blocks/polls until timeout or all responses received.
5. Fit within frame-based timing: tick counts from `machine_tick_count()` (not wall-clock).

## External Dependencies
- **Pinger.h**: Class declaration, PingAddress struct.
- **network_star.h**: Packet constants (`kPingRequestPacket`, `kStarPacketHeaderSize`), UDPpacket type, `NetDDPSendFrame()`.
- **network_private.h**: Networking infrastructure.
- **crc.h**: `calculate_data_crc_ccitt()` ΓÇô computes 16-bit CCITT CRC.
- **mytm.h**: `machine_tick_count()`, `take_mytm_mutex()`, `release_mytm_mutex()`, `sleep_for_machine_ticks()`.
- **AStream.h**: `AOStreamBE` ΓÇô big-endian output serialization stream.
- **Standard library**: `<unordered_map>`, `<atomic>`, `<algorithm>` (std::max).
