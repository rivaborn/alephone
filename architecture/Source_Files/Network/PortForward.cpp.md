# Source_Files/Network/PortForward.cpp

## File Purpose
Implements automatic UPnP port forwarding for network connectivity. Discovers Internet Gateway Devices on the local network, maps a specified port (TCP and UDP) through UPnP to enable external access, and cleans up mappings on shutdown using RAII patterns.

## Core Responsibilities
- Discover UPnP-capable Internet Gateway Devices via `upnpDiscover()`
- Validate and select a valid IGD from discovered devices
- Configure bidirectional TCP and UDP port mappings through the IGD
- Manage lifecycle of port mappings (create on construction, destroy on destruction)
- Handle errors and rollback partial state (e.g., roll back TCP if UDP mapping fails)
- Encapsulate miniupnpc resource management behind unique_ptr deleters

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `PortForwardException` | exception class | Custom exception for port forwarding failures (defined in header) |
| `url_freer_t` | type alias | `unique_ptr<UPNPUrls, &FreeUPNPUrls>` for automatic URL cleanup |
| `devlist_freer_t` | type alias | `unique_ptr<UPNPDev, &freeUPNPDevlist>` for automatic device list cleanup |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor: `PortForward(uint16_t port)`
- **Signature:** `PortForward(uint16_t port)`
- **Purpose:** Initialize port forwarding by discovering an IGD and mapping both TCP and UDP on the specified port
- **Inputs:** 
  - `uint16_t port` ΓÇö network port to map (e.g., 7575)
- **Outputs/Return:** (constructor, no return value)
- **Side effects:**
  - Network I/O: UPnP device discovery (2000ms timeout)
  - Modifies IGD state: adds TCP and UDP port mappings
  - Memory allocation: temporary devlist unique_ptr, url_freer_ member
  - Throws `PortForwardException` on discovery or mapping failure
- **Calls:**
  - `upnpDiscover(2000, nullptr, nullptr, UPNP_LOCAL_PORT_ANY, false, 2, &error)` ΓÇö discover UPnP devices with 2s timeout
  - `UPNP_GetValidIGD(devlist.get(), &urls_, &data_, lanaddr, ...)` ΓÇö validate and populate IGD data (conditional on miniupnpc API version ΓëÑ18)
  - `UPNP_AddPortMapping(...)` ΓÇö twice (TCP, then UDP) to map the port
  - `UPNP_DeletePortMapping(...)` ΓÇö rollback TCP if UDP mapping fails
- **Notes:**
  - Port converted to `std::string` for miniupnpc API calls
  - Version-conditional handling for miniupnpc API version (checks `MINIUPNPC_API_VERSION >= 18`)
  - All mappings use description string `"Aleph One"`
  - Obtains local LAN address from IGD discovery
  - If UDP mapping fails, actively rolls back TCP mapping before throwing
  - Resource management via unique_ptr prevents leaks on exception

### Destructor: `~PortForward()`
- **Signature:** `~PortForward()`
- **Purpose:** Clean up port mappings by removing them from the IGD on object destruction
- **Inputs:** (none)
- **Outputs/Return:** (destructor, no return value)
- **Side effects:**
  - Network I/O: sends UPnP delete commands (TCP then UDP)
  - Allows unique_ptr `url_freer_` to call `FreeUPNPUrls(&urls_)`
- **Calls:**
  - `UPNP_DeletePortMapping(...)` ΓÇö twice (TCP, then UDP)
- **Notes:**
  - Does not check or handle return values; failures are silent
  - Guaranteed to run if constructor completes (RAII guarantee)
  - Cleanup order: manual deletion first, then unique_ptr cleanup

## Control Flow Notes
This is a **one-time setup/teardown class**, not part of the frame loop. Instantiated once during game initialization to enable port forwarding for multiplayer connectivity; destroyed at shutdown to clean up port mappings. Exception-safe: if constructor throws, destructor will not run (no partial state).

## External Dependencies
- **miniupnpc library:**
  - `<miniupnpc/upnpcommands.h>` ΓÇö UPnP command functions (UPNP_AddPortMapping, UPNP_DeletePortMapping, UPNP_GetValidIGD)
  - `<miniupnpc/miniupnpc.h>` ΓÇö UPnP device discovery (upnpDiscover, UPNPUrls, UPNPDev, IGDdatas, freeUPNPDevlist)
- **Standard library:**
  - `<sstream>` ΓÇö ostringstream for error message formatting
  - `<memory>`, `<stdexcept>` ΓÇö unique_ptr, std::runtime_error (via header)
- **Defined elsewhere (miniupnpc):**
  - `upnpDiscover()`, `UPNP_GetValidIGD()`, `UPNP_AddPortMapping()`, `UPNP_DeletePortMapping()`, `FreeUPNPUrls`, `freeUPNPDevlist`
