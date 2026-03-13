# Source_Files/Network/NetworkInterface.cpp

## File Purpose
Implements a C++ abstraction layer over the ASIO networking library, providing simple wrappers for TCP and UDP socket operations. Part of the Aleph One game engine's network subsystem.

## Core Responsibilities
- Manage IP address objects with ASIO endpoint conversion
- Wrap UDP sockets with methods for unicast, broadcast, and async receive
- Wrap TCP sockets for stream-based communication with non-blocking support
- Implement TCP listener/acceptor for accepting incoming connections
- Provide a central `NetworkInterface` factory that creates and manages all socket types
- Handle DNS resolution of hostnames to IP addresses

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `IPaddress` | class | Encapsulates IP address + port; wraps ASIO address objects and provides conversion between string/byte/endpoint forms |
| `UDPpacket` | struct | Container for UDP datagram: destination address, data buffer (1500 bytes), and actual data size |
| `UDPsocket` | class | ASIO UDP socket wrapper supporting unicast send, broadcast, sync/async receive, and broadcast mode toggling |
| `TCPsocket` | class | ASIO TCP socket wrapper supporting send/receive and non-blocking mode |
| `TCPlistener` | class | ASIO TCP acceptor wrapper that accepts incoming connections and returns `TCPsocket` objects |
| `NetworkInterface` | class | Central manager/factory creating UDP sockets, TCP listeners, TCP client sockets, and resolving hostnames |

## Global / File-Static State
None.

## Key Functions / Methods

### IPaddress (constructors)
- **Signature:** `IPaddress(const asio::ip::tcp::endpoint& endpoint)`, `IPaddress(const asio::ip::udp::endpoint& endpoint)`, `IPaddress(const std::string& host, uint16_t port)`, `IPaddress(const uint8_t ip[4], uint16_t port)`
- **Purpose:** Construct IP address objects from various sources (ASIO endpoints, hostname strings, raw byte arrays)
- **Inputs:** endpoint object, hostname string, or raw bytes + port
- **Outputs/Return:** Initialized `IPaddress` object
- **Side effects:** None
- **Calls:** `set_address()`, `set_port()`
- **Notes:** Private endpoints constructors used internally; public string/bytes constructors used by callers

### IPaddress::operator==, operator!=
- **Purpose:** Compare two IP addresses for equality
- **Inputs:** Reference to another `IPaddress`
- **Outputs/Return:** `bool`
- **Side effects:** None
- **Notes:** Uses `std::tie` for comparison of address and port members

### UDPsocket::send
- **Signature:** `int64_t send(const UDPpacket& packet)`
- **Purpose:** Send a UDP packet to a specific address
- **Inputs:** Const reference to `UDPpacket` (contains destination address and data)
- **Outputs/Return:** Bytes sent on success, -1 on error
- **Side effects:** None
- **Calls:** `_socket.send_to()` (ASIO)
- **Notes:** Uses `send_to` for explicit destination; errors return -1

### UDPsocket::broadcast_send
- **Signature:** `int64_t broadcast_send(const UDPpacket& packet)`
- **Purpose:** Send a UDP packet to the broadcast address on the packet's port
- **Inputs:** Const reference to `UDPpacket`
- **Outputs/Return:** Bytes sent, or -1 on error
- **Side effects:** None
- **Calls:** `_socket.send_to()` with `asio::ip::address_v4::broadcast()`

### UDPsocket::receive, receive_async, register_receive_async
- **Purpose:** Receive UDP packets (blocking or async); `register_receive_async` queues an async receive, `receive_async` runs the I/O context for a timeout
- **Inputs/Outputs:** UDPpacket reference (populated with source address and data), timeout in ms for async
- **Return:** Bytes received or -1 on error
- **Side effects:** Updates `_receive_async_return_value` and packet contents; restarts `_io_context` if stopped
- **Calls:** `_socket.receive_from()`, `_socket.async_receive_from()`, `_io_context.run_for()`
- **Notes:** Async receive uses lambda capturing packet and endpoint; `receive_async` blocks the caller for the specified timeout

### TCPsocket::send, receive
- **Purpose:** Send/receive data on a TCP connection
- **Inputs:** Buffer pointer and size
- **Outputs/Return:** Bytes transferred or -1 on error (non-blocking errors like `would_block` return bytes transferred, not -1)
- **Side effects:** None
- **Calls:** `_socket.send()`, `_socket.read_some()` (ASIO)
- **Notes:** Treats `would_block` as success (returns actual bytes), only returns -1 on genuine errors

### TCPlistener::accept_connection
- **Signature:** `std::unique_ptr<TCPsocket> accept_connection()`
- **Purpose:** Accept one incoming TCP connection
- **Inputs:** None
- **Outputs/Return:** Unique pointer to new `TCPsocket` on success, null on error
- **Side effects:** Resets internal `_socket` member after acceptance
- **Calls:** `_acceptor.accept()` (ASIO)
- **Notes:** Creates a fresh socket after each accept; returns null if acceptor error

### NetworkInterface::udp_open_socket
- **Signature:** `std::unique_ptr<UDPsocket> udp_open_socket(uint16_t port)`
- **Purpose:** Create and bind a UDP socket to a local port
- **Inputs:** Port number
- **Outputs/Return:** Unique pointer to `UDPsocket` or null on error
- **Side effects:** Opens and binds socket via ASIO
- **Calls:** `socket.open()`, `socket.bind()`

### NetworkInterface::tcp_open_listener
- **Signature:** `std::unique_ptr<TCPlistener> tcp_open_listener(uint16_t port)`
- **Purpose:** Create a TCP listener bound to a local port
- **Inputs:** Port number
- **Outputs/Return:** Unique pointer to `TCPlistener`
- **Side effects:** None (construction only; acceptor binds immediately)

### NetworkInterface::tcp_connect_socket
- **Signature:** `std::unique_ptr<TCPsocket> tcp_connect_socket(const IPaddress& address)`
- **Purpose:** Create a TCP socket and connect to a remote address
- **Inputs:** Const reference to target `IPaddress`
- **Outputs/Return:** Unique pointer to connected `TCPsocket` or null on error
- **Side effects:** Opens and connects socket
- **Calls:** `socket.connect()` (ASIO)

### NetworkInterface::resolve_address
- **Signature:** `std::optional<IPaddress> resolve_address(const std::string& host, uint16_t port)`
- **Purpose:** Resolve a hostname to an IPv4 address
- **Inputs:** Hostname string and port
- **Outputs/Return:** Optional `IPaddress` (first IPv4 result or empty)
- **Side effects:** DNS query via resolver
- **Calls:** `_resolver.resolve()` (ASIO)
- **Notes:** Returns only the first IPv4 endpoint; discards IPv6 or other results

## Control Flow Notes
No frame/init/shutdown involvement inferable. This is a library/utility module providing on-demand socket creation and management; called by higher-level game systems that need network communication (e.g., multiplayer/lobby code).

## External Dependencies
- **`<asio.hpp>`**: Asio standalone networking library; provides `ip::tcp::socket`, `ip::udp::socket`, `ip::address`, `endpoint`, and `io_context`
- **`<string>`, `<optional>`, `<array>`**: Standard library utilities
- **Defined elsewhere:** `UDPpacket` struct uses `IPaddress` and buffer; all socket classes reference ASIO objects and the shared `io_context`
