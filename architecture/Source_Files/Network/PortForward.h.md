# Source_Files/Network/PortForward.h

## File Purpose
Provides UPnP port forwarding functionality for both TCP and UDP protocols. Wraps the miniupnpc library with RAII semantics and exception-based error handling to automatically manage port mappings and cleanup.

## Core Responsibilities
- Manage UPnP device discovery and port mapping setup
- Maintain resource lifecycle for UPnP URLs and device lists using smart pointers
- Provide exception-based error signaling for port forwarding operations
- Abstract miniupnpc library details behind a C++ interface
- Support simultaneous TCP and UDP port forwarding for a single port

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PortForwardException | class (exception) | Custom exception for port forwarding errors; extends std::runtime_error |
| PortForward | class | Main port forwarding manager; RAII wrapper around UPnP operations |
| url_freer_t | type alias (unique_ptr) | Smart pointer for UPNPUrls with custom deleter FreeUPNPUrls |
| devlist_freer_t | type alias (unique_ptr) | Smart pointer for UPNPDev with custom deleter freeUPNPDevlist |

## Global / File-Static State
None.

## Key Functions / Methods

### PortForward (constructor)
- Signature: `PortForward(uint16_t port)`
- Purpose: Initialize port forwarding for the given port on both TCP and UDP protocols
- Inputs: `port` ΓÇö port number (uint16_t)
- Outputs/Return: None (RAII object constructed)
- Side effects: Discovers UPnP devices, establishes port mappings via miniupnpc library; may throw PortForwardException
- Calls: miniupnpc functions (not directly visible but implied by member initialization)
- Notes: Supports both TCP and UDP simultaneously via single port parameter

### ~PortForward (destructor)
- Signature: `~PortForward()`
- Purpose: Clean up port mappings and release UPnP resources
- Inputs: None
- Outputs/Return: None
- Side effects: Automatic cleanup via unique_ptr deleters (FreeUPNPUrls, freeUPNPDevlist); removes port mappings
- Calls: FreeUPNPUrls, freeUPNPDevlist (via unique_ptr deleters)
- Notes: Non-explicit; relies on RAII for resource management

## Control Flow Notes
Typical usage: construct during network/multiplayer initialization to open ports; destructor called on shutdown or scope exit to clean up mappings. Not inferable from header alone whether this is used for server listening or client connectivity.

## External Dependencies
- `miniupnpc/miniupnpc.h` ΓÇö UPnP client library (UPNPUrls, UPNPDev, FreeUPNPUrls, freeUPNPDevlist)
- `<stdexcept>` ΓÇö std::runtime_error base class
- `<memory>` ΓÇö std::unique_ptr
- `cseries.h` ΓÇö project common definitions
- Conditional compile: `HAVE_MINIUPNPC` preprocessor guard
