# Source_Files/Network/network_capabilities.cpp

## File Purpose
Implements static string constant definitions for the `Capabilities` class, which manages version identifiers and feature capability flags for the Aleph One network layer. These constants represent supported network protocols and features that gatherers (servers) and joiners (clients) negotiate during multiplayer game setup.

## Core Responsibilities
- Defines static string constants that identify specific network capability categories
- Provides human-readable capability names for map access in the inherited `std::map<string, uint32>` container
- Acts as the central registry of feature capability identifiers used throughout the network code
- Supports version negotiation between different network client/server implementations

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Capabilities::kGameworld` | `static const string` | global | Identifier for base game physics/PRNG capability |
| `Capabilities::kGameworldM1` | `static const string` | global | Marathon 1 compatibility variant of gameworld |
| `Capabilities::kStar` | `static const string` | global | Star network protocol capability |
| `Capabilities::kLua` | `static const string` | global | Lua scripting engine support identifier |
| `Capabilities::kGatherable` | `static const string` | global | Joiner's capability to be gathered into game |
| `Capabilities::kZippedData` | `static const string` | global | Compressed map/lua/physics data support |
| `Capabilities::kNetworkStats` | `static const string` | global | Network latency/jitter/error reporting |
| `Capabilities::kRugby` | `static const string` | global | Rugby mode (sane score limit) support |

## Key Functions / Methods
None. (This is a pure definitions file.)

## Control Flow Notes
This file serves as a bootstrap/configuration point for network capability negotiation. The constants defined here are referenced during network connection handshakes to determine feature compatibility between peers. The values are typically inserted into the inherited `std::map<string, uint32>` using the overloaded `operator[]` (defined in the header) with version numbers as values.

## External Dependencies
- `#include "network_capabilities.h"` ΓÇô declares the `Capabilities` class and version constants
- `#include <string>` ΓÇô indirectly via header for `std::string` type
- `std::string` type from STL
