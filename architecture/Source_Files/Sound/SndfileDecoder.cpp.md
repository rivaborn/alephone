# Source_Files/Sound/SndfileDecoder.cpp

## File Purpose
Implements audio file decoding using libsndfile via SDL's RWops file abstraction. Provides a decoder bridge that translates between SDL file I/O and libsndfile's virtual I/O interface, enabling playback of various audio formats (WAV, FLAC, OGG, etc.).

## Core Responsibilities
- Implement virtual I/O callbacks (`sfd_*` functions) bridging SDL_RWops Γåö libsndfile
- Open and initialize SNDFILE handles from FileSpecifier objects
- Decode audio frames into float32 PCM buffers
- Manage playback position, seeking, and rewind operations
- Clean up file resources (SDL_RWops and SNDFILE handles)
- Provide audio format metadata (channels, sample rate, frame count)

## Key Types / Data Structures
None (uses external types: `SNDFILE`, `SF_INFO`, `SDL_RWops`, `SF_VIRTUAL_IO`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sf_virtual` | `SF_VIRTUAL_IO` | static | Callback function pointers for libsndfile's virtual I/O operations |

## Key Functions / Methods

### Constructor `SndfileDecoder::SndfileDecoder()`
- **Purpose:** Initialize decoder instance to null state
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes `sndfile` and `rwops` to null pointers
- **Notes:** Simple zero-initialization; resources acquired later via `Open()`

### `bool SndfileDecoder::Open(FileSpecifier& File)`
- **Purpose:** Open and initialize a sound file for decoding
- **Inputs:** `File` ΓÇô a FileSpecifier reference to the audio file
- **Outputs/Return:** `bool` ΓÇô true if SNDFILE handle successfully created, false otherwise
- **Side effects:** Calls `Close()` first; acquires SDL_RWops and SNDFILE; populates `sfinfo` (format, channels, sample rate, frame count)
- **Calls:** `File.Open()`, `openedFile.TakeRWops()`, `sf_open_virtual()`, `Close()`
- **Notes:** Initializes `sfinfo.format` to 0 before opening; relies on virtual I/O callbacks defined in this file

### `int32 SndfileDecoder::Decode(uint8* buffer, int32 max_length)`
- **Purpose:** Decode audio frames into a float32 PCM buffer
- **Inputs:** `buffer` ΓÇô destination uint8 buffer; `max_length` ΓÇô max bytes to write
- **Outputs/Return:** Bytes written (always a multiple of 4, representing float32 samples)
- **Side effects:** Advances internal libsndfile read position
- **Calls:** `sf_read_float()`
- **Notes:** Interprets buffer as float32; returns 0 if `sndfile` is null; conversion `max_length / 4` assumes 4-byte floats

### `void SndfileDecoder::Rewind()`
- **Purpose:** Reset playback to the start of the file
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Seeks libsndfile position to frame 0
- **Calls:** `sf_seek()`
- **Notes:** No-op if `sndfile` is null

### `uint32_t SndfileDecoder::Position()` (getter)
- **Purpose:** Query current playback byte position
- **Inputs:** None
- **Outputs/Return:** Byte offset in the decoded audio stream
- **Side effects:** Queries libsndfile position via `sf_seek(SEEK_CUR)`
- **Calls:** `sf_seek()`, `BytesPerFrame()`
- **Notes:** Multiplies frame position by bytes-per-frame to convert to byte offset; returns 0 if `sndfile` is null

### `void SndfileDecoder::Position(uint32_t position)` (setter)
- **Purpose:** Seek to a specific byte position in the decoded audio
- **Inputs:** `position` ΓÇô target byte offset
- **Outputs/Return:** None
- **Side effects:** Changes libsndfile read position
- **Calls:** `sf_seek()`, `BytesPerFrame()`
- **Notes:** Divides position by bytes-per-frame to convert from bytes to frames; no-op if `sndfile` is null

### `void SndfileDecoder::Close()`
- **Purpose:** Clean up and release all audio file resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes libsndfile handle; closes and frees SDL_RWops; zeroes both pointers
- **Calls:** `sf_close()`, `SDL_RWclose()`
- **Notes:** Safe to call multiple times (checks for null); called by destructor and at the start of `Open()`

### Static Helper Functions
- **`sfd_get_filelen`, `sfd_seek`, `sfd_read`, `sfd_write`, `sfd_tell`:** Bridging callbacks that libsndfile invokes for all file I/O; each casts the opaque `void*` context to SDL_RWops and delegates to SDL file functions. Details summarized rather than fully documented due to their trivial wrapper nature.

## Control Flow Notes
Typical lifecycle: `Open()` ΓåÆ `Decode()` in game loop (via parent Decoder interface) ΓåÆ `Rewind()`/`Position()` for seeking ΓåÆ `Close()` at shutdown. The virtual I/O callbacks are invoked internally by libsndfile during `Open()` (to read format metadata) and `Decode()` (to read audio frames). Destructor ensures cleanup via `Close()`.

## External Dependencies
- **Notable includes:** `"SndfileDecoder.h"`, `"sndfile.h"` (libsndfile)
- **External symbols/types:** `SNDFILE`, `SF_INFO`, `SF_VIRTUAL_IO` (libsndfile); `SDL_RWops`, `SDL_RWtell`, `SDL_RWseek`, `SDL_RWread`, `SDL_RWwrite`, `SDL_RWclose` (SDL); `FileSpecifier`, `OpenedFile` (defined elsewhere); `PlatformIsLittleEndian()`, `AudioFormat::_32_float` (inherited or platform utilities)
