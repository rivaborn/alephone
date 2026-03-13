# Source_Files/Network/SSLP_API.h

## File Purpose
Public API header for SSLP (Simple Service Location Protocol), a custom service discovery protocol for locating and advertising networked services. Designed for the Aleph One project to enable cross-network player discovery, replacing AppleTalk-based service location.

## Core Responsibilities
- Expose service discovery API for client-side service location
- Expose service advertisement API for server-side service exposure
- Define callback mechanism for service lifecycle events (found, lost, renamed)
- Coordinate periodic processing of discovery and advertisement logic
- Provide service instance representation with type, name, and network address

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SSLP_ServiceInstance` | struct | Represents a discoverable network service with type, name, and address |
| `SSLP_Service_Instance_Status_Changed_Callback` | function pointer typedef | Callback signature invoked when service discovery status changes |

## Global / File-Static State
None.

## Key Functions / Methods

### SSLP_Pump
- Signature: `void SSLP_Pump()`
- Purpose: Yield processing time to SSLP for handling incoming packets and timeout logic
- Inputs: None
- Outputs/Return: None
- Side effects: May invoke registered callbacks; processes pending network packets
- Calls: (Not visible in this file)
- Notes: Must be called frequently (suggested ΓëÑ every 5 seconds) in non-threaded implementations. In threaded implementations, internal threads handle pumping. More frequent calls increase responsiveness.

### SSLP_Locate_Service_Instances
- Signature: `void SSLP_Locate_Service_Instances(const char* inServiceType, SSLP_Service_Instance_Status_Changed_Callback inFoundCallback, SSLP_Service_Instance_Status_Changed_Callback inLostCallback, SSLP_Service_Instance_Status_Changed_Callback inNameChangedCallback)`
- Purpose: Initiate service discovery for instances of a given service type
- Inputs: Service type string; three optional callbacks
- Outputs/Return: None (status reported via callbacks)
- Side effects: Broadcasts discovery requests; invokes callbacks from arbitrary threads when services are found/lost/renamed; caches service pointers
- Calls: (Not visible in this file)
- Notes: Only one service-location attempt may be active at a time. Callbacks may execute in different threads than the caller. ServiceInstance pointers remain valid until `inLostCallback` is invoked; caller may cache references. Duplicate calls without `Stop_Locating_Service_Instances` is undefined/fatal.

### SSLP_Stop_Locating_Service_Instances
- Signature: `void SSLP_Stop_Locating_Service_Instances(const char* inServiceType)`
- Purpose: Terminate an active service location search
- Inputs: Service type (currently ignored; only one search permitted)
- Outputs/Return: None
- Side effects: Halts discovery broadcasts; invalidates all previously found ServiceInstance pointers; no further callbacks issued
- Calls: (Not visible in this file)
- Notes: `inServiceType` argument is ignored in current implementation. Calling without an active search is undefined/fatal.

### SSLP_Allow_Service_Discovery
- Signature: `void SSLP_Allow_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance)`
- Purpose: Advertise a local service instance for remote discovery
- Inputs: ServiceInstance with type, name, and port (host address ignored)
- Outputs/Return: None
- Side effects: Copies service data; begins responding to discovery broadcasts for this service; initiates optional hinting
- Calls: (Not visible in this file)
- Notes: Only one service may be advertised at a time. Input struct is copied, so caller retains ownership. Duplicate calls without `Disallow_Service_Discovery` is undefined/fatal.

### SSLP_Disallow_Service_Discovery
- Signature: `void SSLP_Disallow_Service_Discovery(const SSLP_ServiceInstance* inServiceInstance)`
- Purpose: Stop advertising a local service instance
- Inputs: ServiceInstance pointer (currently ignored; only one service permitted)
- Outputs/Return: None
- Side effects: Stops responding to discovery broadcasts; halts service hinting; frees internal service data
- Calls: (Not visible in this file)
- Notes: `inServiceInstance` argument is ignored in current implementation. Calling without an active advertised service is undefined/fatal.

## Control Flow Notes
SSLP operates in a requestΓÇôresponse discovery pattern:

**Client side** (locate): Application calls `SSLP_Locate_Service_Instances()` to initiate discovery, then calls `SSLP_Pump()` periodically to process incoming advertisements; callbacks fire when services appear/disappear/rename. Terminates with `SSLP_Stop_Locating_Service_Instances()`.

**Server side** (advertise): Application calls `SSLP_Allow_Service_Discovery()` to publish a service, then calls `SSLP_Pump()` periodically to respond to discovery requests. Terminates with `SSLP_Disallow_Service_Discovery()`.

In single-threaded mode, callbacks execute synchronously within `SSLP_Pump()`. In multi-threaded mode, callbacks may execute asynchronously. Only one simultaneous discovery search and one advertised service are supported.

## External Dependencies
- **NetworkInterface.h**: Provides `IPaddress` type (encapsulates host and port in network byte order)
- **Implicit C runtime**: `NULL` pointer constant
