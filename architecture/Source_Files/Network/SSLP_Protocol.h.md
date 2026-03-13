# Source_Files/Network/SSLP_Protocol.h

## File Purpose
Defines the Simple Service Location Protocol (SSLP) specificationΓÇöa lightweight network service discovery protocol for the Aleph One game engine. Specifies packet structure, message types, and protocol semantics for discovering network game instances and players across non-AppleTalk networks.

## Core Responsibilities
- Define the SSLP packet format and wire protocol specification
- Document message types: FIND (service discovery request), HAVE (service advertisement), LOST (service retirement)
- Specify magic numbers, version, and port constants for protocol identification
- Establish naming constraints for service types and instance names
- Document protocol behavior, delivery guarantees, and usage patterns

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SSLP_Packet` | struct | Network packet containing magic, version, message type, target port, and service type/name (80 bytes total) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SSLP_PORT` | uint16_t | compile-time constant | Default port (15367) for SSLP communication |
| `SSLPP_MAGIC` | uint32_t | compile-time constant | Magic value (0x73736c70, "sslp") to identify SSLP packets |
| `SSLPP_VERSION` | uint32_t | compile-time constant | Protocol version (1) for compatibility checking |
| `SSLPP_MESSAGE_FIND` | uint32_t | compile-time constant | Message type identifier (0x66696e64, "find") |
| `SSLPP_MESSAGE_HAVE` | uint32_t | compile-time constant | Message type identifier (0x68617665, "have") |
| `SSLPP_MESSAGE_LOST` | uint32_t | compile-time constant | Message type identifier (0x6c6f7374, "lost") |
| `SSLP_MAX_TYPE_LENGTH` | int | compile-time constant | Max length of service type string (32 bytes) |
| `SSLP_MAX_NAME_LENGTH` | int | compile-time constant | Max length of service instance name (32 bytes) |
| `SIZEOF_SSLP_Packet` | int | compile-time constant | Packed wire size (80 bytes) to avoid compiler dependency |

## Key Functions / Methods
NoneΓÇöthis is a protocol specification header with no function implementations.

## Control Flow Notes
SSLP operates as a best-effort service discovery mechanism on top of UDP:

1. **Discovery Phase**: Host broadcasts FIND messages (typically every 1ΓÇô5 seconds) advertising a desired `service_type` from a known port
2. **Advertisement Phase**: Hosts with matching services respond with HAVE messages to the sender's host/port, providing the service instance `service_name` and contact `sslpp_service_port`
3. **Retirement Phase**: When a service is no longer discoverable, the host broadcasts a LOST message to invalidate cached entries

Hosts may unicast FIND/HAVE if they suspect a specific peer. Clients are expected to age and discard stale service entries if no HAVE is received within a time window (protocol does not guarantee delivery).

## External Dependencies
- Standard C types: `uint32_t`, `uint16_t`, `char` (from `<stdint.h>`)
- No external symbols or library dependencies; pure protocol specification

**Notes**: Service type and name fields are treated as C-strings for comparison (match on first `\0`); NULL termination is not required if all `SSLP_MAX_*_LENGTH` bytes are used. In Aleph One, service types are player identifiers and service names are player join handles.
