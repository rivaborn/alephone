# Source_Files/Network/network_capabilities.h

## File Purpose
Defines a versioning and capability-negotiation system for network protocol compatibility in Aleph One. Maps feature names (strings) to version numbers to enable gatherers (servers) and joiners (clients) to verify mutual protocol support during connection handshakes.

## Core Responsibilities
- Define version constants for network protocols (gameworld, star, Lua, zipped data, network stats, rugby)
- Provide a map-based container (`Capabilities`) for storing and querying capability versions
- Support capability lookup with bounds checking on key names
- Enable cross-version compatibility checking in network initialization

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `capabilities_t` | typedef | `std::map<string, uint32>` alias; maps capability names to version numbers |
| `Capabilities` | class | Inherits from `capabilities_t`; wraps the map with version constants and bounds-checked `operator[]` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kMaxKeySize` | enum { int } | static | Max length for capability key names (1024 chars); used in bounds checking |
| `kGameworldVersion` | static const int | static | Version 6 for core physics/PRNG protocol |
| `kGameworldM1Version` | static const int | static | Version 4 for Marathon 1 compatibility variant |
| `kStarVersion` | static const int | static | Version 6 for star network protocol |
| `kLuaVersion` | static const int | static | Version 2 for Lua scripting support |
| `kGatherableVersion` | static const int | static | Version 1 for joiner gatherable capability |
| `kZippedDataVersion` | static const int | static | Version 1 for compressed map/lua/physics data |
| `kNetworkStatsVersion` | static const int | static | Version 1 for latency/jitter/error reporting |
| `kRugbyVersion` | static const int | static | Version 1 for sane score limits |
| `kGameworld`, `kGameworldM1`, `kStar`, etc. | static const string | static | Capability names corresponding to version constants |

## Key Functions / Methods

### Capabilities::operator[]
- **Signature:** `uint32& operator[](const string& k)`
- **Purpose:** Provide safe subscript access to the underlying map with key-length validation.
- **Inputs:** Key string `k`
- **Outputs/Return:** Reference to the mapped `uint32` value (or newly inserted default if key absent).
- **Side effects:** May insert new map entry if key does not exist; asserts key length < `kMaxKeySize`.
- **Calls:** `capabilities_t::operator[]` (parent class map subscript)
- **Notes:** Bounds check prevents oversized keys; relies on parent map semantics for insertion.

## Control Flow Notes
Used during network session establishment: clients and servers exchange `Capabilities` instances to negotiate compatible feature versions before gameplay begins. Version mismatches indicate incompatible protocol revisions.

## External Dependencies
- `cseries.h` ΓÇô platform abstraction and type definitions (e.g., `uint32`)
- `<string>` ΓÇô STL `std::string`
- `<map>` ΓÇô STL `std::map` container
