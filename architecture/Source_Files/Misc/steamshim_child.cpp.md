# Source_Files/Misc/steamshim_child.cpp

## File Purpose
Child-process side of a Steam API shim that communicates with a parent process via named pipes (IPC). Provides a lightweight game-side interface to Steam features (achievements, stats, workshop) without linking directly to the Steam SDK.

## Core Responsibilities
- Cross-platform named pipe I/O (Windows HANDLE / Unix file descriptor abstraction)
- Command serialization: game calls ΓåÆ binary protocol ΓåÆ parent process
- Event deserialization: parent responses ΓåÆ STEAMSHIM_Event structures
- Connection lifecycle: init via environment variables, maintain pipes, graceful shutdown
- Public API: achievement/stat get/set, workshop queries, game info, event pump

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ShimCmd` | enum | Command opcodes sent to parent (SHIMCMD_PUMP, SHIMCMD_SETACHIEVEMENT, etc.) |
| `STEAMSHIM_EventType` | enum | Event types received from parent (SHIMEVENT_STATSSTORED, SHIMEVENT_GETACHIEVEMENT, etc.) |
| `STEAMSHIM_Event` | struct | Event payload: type, status flags, values (int/float), achievement name, workshop result data |
| `item_upload_data` | struct | Workshop item metadata to upload (ID, type, content type, paths) |
| `PipeType` | typedef | Platform abstraction: HANDLE (Windows) or int (Unix) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `GPipeRead` | PipeType | static | Pipe handle for reading parent responses |
| `GPipeWrite` | PipeType | static | Pipe handle for sending commands to parent |
| `buf[]` in STEAMSHIM_pump | uint8[65536] | static | Receive buffer for event data |
| `br` in STEAMSHIM_pump | int | static | Bytes accumulated in receive buffer |

## Key Functions / Methods

### STEAMSHIM_init
- **Signature:** `int STEAMSHIM_init(void)`
- **Purpose:** Connect to parent shim process by retrieving pipe handles from environment variables
- **Inputs:** Environment variables `STEAMSHIM_READHANDLE`, `STEAMSHIM_WRITEHANDLE` (as decimal strings)
- **Outputs/Return:** Non-zero on success, zero on failure
- **Side effects:** Sets `GPipeRead`, `GPipeWrite` globals
- **Calls:** `initPipes()`, debug printf
- **Notes:** Parent process sets environment variables before spawning child; sscanf parses handles as `unsigned long long`

### STEAMSHIM_deinit
- **Signature:** `void STEAMSHIM_deinit(void)`
- **Purpose:** Close pipes and signal shutdown to parent
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sends SHIMCMD_BYE, closes both pipe handles, nullifies globals
- **Calls:** `writeBye()`, `closePipe()`
- **Notes:** Safe to call multiple times (checks `NULLPIPE`); commented signal handling for SIGPIPE

### STEAMSHIM_pump
- **Signature:** `const STEAMSHIM_Event *STEAMSHIM_pump(void)`
- **Purpose:** Main event loop; poll for parent responses and deserialize them
- **Inputs:** None
- **Outputs/Return:** Pointer to next available event, or NULL if none pending
- **Side effects:** Reads from `GPipeRead`, consumes buffer, modifies static `buf` and `br`
- **Calls:** `pipeReady()`, `readPipe()`, `processEvent()`, `STEAMSHIM_deinit()` on read failure
- **Notes:** Protocol: [2-byte length][event data]. Sends SHIMCMD_PUMP if buffer empty and no pending event. Memmove shifts buffer after event consumed.

### processEvent
- **Signature:** `static const STEAMSHIM_Event *processEvent(const uint8 *buf, size_t buflen)`
- **Purpose:** Deserialize binary event data into STEAMSHIM_Event struct
- **Inputs:** Buffer pointer, length
- **Outputs/Return:** Pointer to static event struct, NULL on parsing error
- **Side effects:** Modifies static event struct; calls shim_deserialize on workshop/game info
- **Calls:** debug printf, struct member deserialization
- **Notes:** Switch on type; validates buffer length before reading fields; strcpy for achievement/stat names (buffer overflow risk if name > 256)

### STEAMSHIM_setAchievement
- **Signature:** `void STEAMSHIM_setAchievement(const char *name, const int enable)`
- **Purpose:** Send achievement set/unlock command to parent
- **Inputs:** Achievement name, enable flag
- **Outputs/Return:** None
- **Side effects:** Serializes and writes to `GPipeWrite`
- **Calls:** `isDead()`, `writePipe()`, strlen, strcpy
- **Notes:** Builds buffer: [length][SHIMCMD_SETACHIEVEMENT][enable_flag][name\0]. Length computed after strcpy.

### STEAMSHIM_pump (event pump loop)
- **Signature:** Called repeatedly in game main loop
- **Purpose:** Non-blocking check for pending Steam events
- **Inputs:** None
- **Outputs/Return:** Next event or NULL
- **Notes:** Critical for async Steam integration; must be called every frame

**Other stat/workshop commands** (STEAMSHIM_setStatI, STEAMSHIM_getStatI, STEAMSHIM_setStatF, STEAMSHIM_getStatF, STEAMSHIM_uploadWorkshopItem, STEAMSHIM_queryWorkshopItemOwned, etc.):
- All follow similar pattern: check `isDead()`, serialize command + data, write to pipe
- `writeStatThing()` helper builds stat command buffers (handles int32 / float binary encoding)
- Workshop commands use `std::ostringstream` for struct serialization

## Control Flow Notes
**Initialization ΓåÆ Frame Loop ΓåÆ Shutdown:**
1. `STEAMSHIM_init()` called on startup; retrieves pipe handles from env vars set by parent
2. Game main loop calls `STEAMSHIM_pump()` every frame to drain parent events
3. Game calls command functions (setAchievement, setStatI, etc.) as needed; these queue to write pipe
4. On shutdown, `STEAMSHIM_deinit()` sends BYE and closes pipes

**Event Processing:**
- Parent writes: [2-byte length][event type][event data]
- Child reads into 64KB buffer; accumulates until complete event available
- `processEvent()` deserializes by type; returns event or NULL on error
- Buffer is rotated after event consumed (memmove)

## External Dependencies
- **Windows:** `<windows.h>` (PeekNamedPipe, WriteFile, ReadFile, CloseHandle, GetEnvironmentVariableA)
- **Unix/Linux:** `<poll.h>` (poll), `<unistd.h>` (read, write, close), `<errno.h>`, `<signal.h>` (SIGPIPE)
- **C Standard:** `<cstring>` (strcpy, strlen, memcpy, memmove), `<cstdio>` (printf, sscanf)
- **C++ Standard:** `<sstream>` (ostringstream for workshop serialization)
- **Header:** `"steamshim_child.h"` ΓÇö event/command enums, API signatures, struct definitions (item_upload_data, item_owned_query_result, etc. with shim_serialize/deserialize methods)
