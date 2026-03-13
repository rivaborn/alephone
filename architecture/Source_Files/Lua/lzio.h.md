# Source_Files/Lua/lzio.h

## File Purpose
Defines buffered stream (ZIO) and memory buffer (Mbuffer) abstractions for Lua's I/O subsystem. Provides efficient character-by-character reading with user-supplied reader callbacks and dynamic buffer management for the lexer and chunk loading.

## Core Responsibilities
- Stream abstraction (`ZIO`) wrapping arbitrary reader functions with a refill buffer
- Macro-based fast path for character consumption (`zgetc`)
- Dynamic memory buffer allocation, resizing, and freeing (`Mbuffer`)
- Buffer initialization, space allocation, and state queries
- Stream initialization and batch reading interface

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ZIO` | struct | Buffered input stream with reader callback, current position, remaining bytes, and Lua state context |
| `Mbuffer` | struct | Dynamic resizable character buffer with current length and allocated size |

## Global / File-Static State
None.

## Key Functions / Methods

### luaZ_init
- **Signature:** `void luaZ_init(lua_State *L, ZIO *z, lua_Reader reader, void *data)`
- **Purpose:** Initialize a ZIO stream with a custom reader function.
- **Inputs:** Lua state, ZIO struct to initialize, reader callback, and user-supplied data for the reader.
- **Outputs/Return:** None (void).
- **Side effects:** Modifies the ZIO struct fields; no I/O performed yet.
- **Calls:** Not inferable from header.
- **Notes:** The reader callback (`lua_Reader`) is invoked on-demand by `luaZ_fill()` when the buffer is exhausted.

### luaZ_read
- **Signature:** `size_t luaZ_read(ZIO* z, void* b, size_t n)`
- **Purpose:** Read up to `n` bytes from the ZIO stream into buffer `b`.
- **Inputs:** ZIO stream, destination buffer, number of bytes to read.
- **Outputs/Return:** Number of bytes actually read.
- **Side effects:** Advances stream position, may trigger `luaZ_fill()` refills.
- **Calls:** Not inferable from header.
- **Notes:** May return fewer bytes than requested if stream ends.

### luaZ_openspace
- **Signature:** `char *luaZ_openspace(lua_State *L, Mbuffer *buff, size_t n)`
- **Purpose:** Ensure the buffer has at least `n` bytes of free space, allocating or resizing as needed.
- **Inputs:** Lua state, Mbuffer to expand, required space.
- **Outputs/Return:** Pointer to the buffer start.
- **Side effects:** May reallocate buffer memory via `luaM_reallocvector`.
- **Calls:** Indirectly calls `luaM_reallocvector` (via macro `luaZ_resizebuffer`).
- **Notes:** Used to pre-allocate space before writing into the buffer.

### luaZ_fill (private)
- **Signature:** `int luaZ_fill(ZIO *z)`
- **Purpose:** Refill the ZIO buffer by invoking the reader callback.
- **Inputs:** ZIO stream.
- **Outputs/Return:** Next character read, or `EOZ` (-1) on end of stream.
- **Side effects:** Updates `z->n` (remaining bytes) and `z->p` (current position); invokes the reader callback.
- **Calls:** User-supplied `reader` callback via `z->reader`.
- **Notes:** Called indirectly via the `zgetc` macro when `z->n` reaches 0.

## Control Flow Notes
ZIO fits into the **load/parse phase**: `lua_load()` and the lexer use ZIO to abstract different input sources (file, memory, string). The reader callback pattern allows the Lua VM to load chunks from any source without coupling to specific I/O primitives. Mbuffer is typically used by the lexer to accumulate tokens or string literals before processing.

## External Dependencies
- **lua.h**: `lua_State`, `lua_Reader` callback typedef (`const char * (*lua_Reader)(lua_State *L, void *ud, size_t *sz)`)
- **lmem.h**: Memory allocation macros (`luaM_reallocvector`, `luaM_realloc_`)
