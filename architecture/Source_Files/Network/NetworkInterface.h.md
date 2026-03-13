# Source_Files/Network/NetworkInterface.h

## File Purpose
Provides a C++ abstraction layer over ASIO for network operations in a game engine. Encapsulates UDP and TCP socket management, server listeners, and IP address handling for both client and server networking scenarios.

## Core Responsibilities
- Define `IPaddress` class to wrap IP addresses and ports with conversion utilities
- Implement `UDPsocket` for datagram communication (broadcast, send, receive, async)
- Implement `TCPsocket` for stream-based connections
- Implement `TCPlistener` for server-side connection acceptance
- Provide `NetworkInterface` as the main facade for socket creation and DNS resolution
- Define `UDPpacket` structure for packet data with embedded address information

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `IPaddress` | class | Wraps ASIO IP address and port; provides accessors and mutation methods |
| `UDPpacket` | struct | Contains address, fixed-size buffer (1500 bytes), and data length |
| `UDPsocket` | class | ASIO UDP socket wrapper with sync/async receive and broadcast support |
| `TCPsocket` | class | ASIO TCP socket wrapper for send/receive and remote address queries |
| `TCPlistener` | class | ASIO TCP acceptor for server-side listening and connection acceptance |
| `NetworkInterface` | class | Main facade managing ASIO context, socket factories, and DNS resolution |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ddpMaxData` | `#define` (1500) | global | Maximum UDP packet payload size; mirrors AppleTalk compatibility constant |

## Key Functions / Methods

### IPaddress (constructors & accessors)
- **Signature:** `IPaddress(const std::string& host, uint16_t port)`, `IPaddress(const uint8_t ip[4], uint16_t port)`
- **Purpose:** Construct IP address from hostname string or raw IPv4 bytes
- **Inputs:** Host string or 4-byte IPv4 array; port number
- **Outputs/Return:** IPaddress object
- **Side effects:** None (constructor)
- **Notes:** Private constructors from ASIO endpoints allow friend classes to create IPaddress from socket endpoints

### UDPsocket::broadcast_send / send
- **Signature:** `int64_t broadcast_send(const UDPpacket& packet)`, `int64_t send(const UDPpacket& packet)`
- **Purpose:** Send UDP packet to broadcast or unicast address
- **Inputs:** UDPpacket with address and data
- **Outputs/Return:** Bytes sent (int64_t)
- **Side effects:** Network I/O via ASIO socket
- **Calls:** ASIO `udp::socket::send_to()`
- **Notes:** Broadcast variant enables broadcast flag on socket

### UDPsocket::receive / receive_async
- **Signature:** `int64_t receive(UDPpacket& packet)`, `int64_t receive_async(int timeout_ms)`
- **Purpose:** Receive UDP packet (blocking or async with timeout)
- **Inputs:** Reference to packet struct to fill; timeout in ms (async only)
- **Outputs/Return:** Bytes received; `-1` or error code on timeout/failure
- **Side effects:** Blocks on `receive()`, asynchronous callback scheduling on `receive_async()`
- **Calls:** ASIO `udp::socket::receive_from()`, async handlers
- **Notes:** `register_receive_async()` must be called before `receive_async()`; `_receive_async_endpoint` and `_receive_async_return_value` track pending async state

### TCPsocket::send / receive
- **Signature:** `int64_t send(uint8_t* buffer, size_t size)`, `int64_t receive(uint8_t* buffer, size_t size)`
- **Purpose:** Send/receive raw bytes on established TCP connection
- **Inputs:** Buffer pointer and size
- **Outputs/Return:** Bytes transferred
- **Side effects:** Network I/O via ASIO socket
- **Calls:** ASIO `tcp::socket::send()` / `receive()`

### TCPlistener::accept_connection
- **Signature:** `std::unique_ptr<TCPsocket> accept_connection()`
- **Purpose:** Block until incoming TCP connection, return wrapped socket
- **Outputs/Return:** Unique pointer to new TCPsocket
- **Side effects:** Blocking accept; creates new socket from `_acceptor`
- **Calls:** ASIO `tcp::acceptor::accept()`

### NetworkInterface::udp_open_socket / tcp_connect_socket / tcp_open_listener
- **Signature:** `std::unique_ptr<UDPsocket> udp_open_socket(uint16_t port)`, etc.
- **Purpose:** Factory methods to create and bind sockets
- **Inputs:** Port number (or address for TCP connect)
- **Outputs/Return:** Unique pointer to configured socket object
- **Side effects:** Socket binding/connection via ASIO
- **Calls:** ASIO socket constructors, bind, connect
- **Notes:** Manages ASIO context lifetime; returned sockets hold reference to `_io_context`

### NetworkInterface::resolve_address
- **Signature:** `std::optional<IPaddress> resolve_address(const std::string& host, uint16_t port)`
- **Purpose:** DNS resolution (hostname ΓåÆ IP address)
- **Inputs:** Hostname string; port number
- **Outputs/Return:** Optional IPaddress if resolution succeeds
- **Side effects:** Blocking DNS lookup via ASIO resolver
- **Calls:** ASIO `tcp::resolver::resolve()`

## Control Flow Notes
This header is a utility/initialization layer. Typical usage:
1. Create `NetworkInterface` instance (initializes ASIO context and resolver)
2. Call factory methods (`udp_open_socket`, `tcp_connect_socket`, `tcp_open_listener`) to create sockets
3. Use socket methods (`send`, `receive`, `accept_connection`) during frame/update loops
4. Destruction of `NetworkInterface` cleans up ASIO context

Async UDP receive (`register_receive_async` / `receive_async`) suggests support for non-blocking network events during frame processing.

## External Dependencies
- **`<asio.hpp>`:** Asynchronous I/O library (Boost.ASIO or standalone); provides socket, resolver, and endpoint abstractions
- **`<string>`, `<optional>`, `<array>`:** C++ standard library
- **ASIO namespaces used:** `asio::ip::address`, `asio::ip::udp`, `asio::ip::tcp`, `asio::io_context`
