# Source_Files/Sound/SndfileDecoder.h

## File Purpose
Defines `SndfileDecoder`, a concrete decoder implementation that loads and decodes audio files using libsndfile. Inherits from the `Decoder` interface to support the engine's sound subsystem with format-agnostic file playback.

## Core Responsibilities
- Open and manage audio files via libsndfile library
- Decode audio frames into 32-bit float PCM buffers
- Track and report playback position and seek within files
- Provide audio metadata (sample rate, channel count, duration, frame count)
- Handle file I/O through SDL_RWops abstraction layer

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SndfileDecoder | class | Concrete decoder wrapping libsndfile; inherits `Decoder` |
| SNDFILE | typedef (opaque) | libsndfile file handle |
| SF_INFO | struct (external) | libsndfile metadata container (channels, frames, samplerate) |
| SDL_RWops | struct (external) | SDL file I/O operations handle |

## Global / File-Static State
None.

## Key Functions / Methods

### Open
- Signature: `bool Open(FileSpecifier& File)`
- Purpose: Open and initialize an audio file for decoding
- Inputs: File specifier (path/reference)
- Outputs/Return: `true` if successful, `false` on failure
- Side effects: Allocates SNDFILE handle, populates sfinfo, creates SDL_RWops
- Calls: Not visible in header

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Decode audio frames into a buffer
- Inputs: Output buffer, max bytes to decode
- Outputs/Return: Bytes written to buffer (or 0 at EOF)
- Side effects: Advances internal playback position
- Calls: Not visible in header

### Rewind
- Signature: `void Rewind()`
- Purpose: Reset playback to file start
- Side effects: Resets internal file pointer

### Close
- Signature: `void Close()`
- Purpose: Release file handle and resources
- Side effects: Closes SNDFILE and SDL_RWops

### Position (overloaded)
- Signature: `uint32_t Position()` / `void Position(uint32_t position)`
- Purpose: Query current frame position or seek to a frame
- Outputs/Return: Current frame position (getter); void (setter)
- Side effects: Seeks file pointer (setter)

**Trivial accessors** (summarized): `IsStereo()`, `BytesPerFrame()`, `Rate()`, `Duration()`, `Size()`, `Frames()` ΓÇö all query `sfinfo` and perform arithmetic; `GetAudioFormat()` always returns `AudioFormat::_32_float`; `IsLittleEndian()` delegates to platform utility.

## Control Flow Notes
Typical usage: `Open()` ΓåÆ repeated `Decode()` calls (until returns 0) ΓåÆ `Close()`. Optional `Rewind()` or `Position()` to replay or jump. Inherits from `Decoder` (which itself inherits `StreamDecoder`), integrating into the engine's sound loading/streaming pipeline.

## External Dependencies
- **libsndfile** (`sndfile.h`) ΓÇö decoding library; defines SNDFILE, SF_INFO
- **SDL** (implicit via `SDL_RWops`) ΓÇö cross-platform I/O abstraction
- **Decoder.h** ΓÇö parent class `Decoder` and `StreamDecoder` interface
- **FileHandler.h** (via Decoder.h) ΓÇö defines `FileSpecifier`
- **SoundManagerEnums.h** (via Decoder.h) ΓÇö defines `AudioFormat` enum
- Platform utilities (`PlatformIsLittleEndian()` ΓÇö defined elsewhere)
