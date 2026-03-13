# Source_Files/Sound/Decoder.h

## File Purpose
Defines abstract interfaces for audio decoding from file streams and pre-decoded files. Provides a pluggable architecture for different audio codec implementations (WAVE, Ogg Vorbis, MP3, etc.) within the Aleph One game engine's sound system.

## Core Responsibilities
- Define abstract `StreamDecoder` interface for streaming audio decoders
- Define abstract `Decoder` interface for pre-decoded/frame-based decoders
- Provide methods for file I/O: opening, closing, reading audio data
- Provide query methods for audio properties: format, sample rate, channels, endianness, duration
- Provide seeking/positioning support for random access
- Define factory methods to instantiate appropriate concrete decoder based on file type

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `StreamDecoder` | abstract class | Base interface for audio stream decoders; defines contract for file access, decoding, and format queries |
| `Decoder` | abstract class | Extends `StreamDecoder` to add frame-based interface; used for pre-decoded or fully-loadable sounds |

## Global / File-Static State
None.

## Key Functions / Methods

### StreamDecoder::Open
- Signature: `virtual bool Open(FileSpecifier &File) = 0`
- Purpose: Open an audio file and prepare it for decoding
- Inputs: `File` ΓÇô specifies which audio file to open
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Initializes internal decoder state
- Calls: (implementation-dependent)
- Notes: Called before `Decode()` or property queries

### StreamDecoder::Decode
- Signature: `virtual int32 Decode(uint8* buffer, int32 max_length) = 0`
- Purpose: Decode audio data from the current position and write into caller's buffer
- Inputs: `buffer` ΓÇô destination memory; `max_length` ΓÇô bytes requested
- Outputs/Return: `int32` ΓÇô bytes actually decoded (Γëñ `max_length`; may be 0 at EOF)
- Side effects: Advances internal read position; may allocate internally for decoding
- Calls: (implementation-dependent, e.g., codec-specific decoders)
- Notes: Non-blocking; caller controls timing and buffer management

### StreamDecoder::Rewind
- Signature: `virtual void Rewind() = 0`
- Purpose: Reset playback to the beginning of the file
- Inputs: none
- Outputs/Return: void
- Side effects: Resets internal position to file start
- Calls: (implementation-dependent)

### StreamDecoder::Close
- Signature: `virtual void Close() = 0`
- Purpose: Close the audio file and release decoder resources
- Inputs: none
- Outputs/Return: void
- Side effects: Releases file handle and internal buffers
- Calls: (implementation-dependent)

### StreamDecoder::GetAudioFormat
- Signature: `virtual AudioFormat GetAudioFormat() = 0`
- Purpose: Query the audio sample format (8-bit, 16-bit, 32-bit float)
- Outputs/Return: `AudioFormat` enum value
- Notes: Call after `Open()`

### StreamDecoder::IsStereo / BytesPerFrame / Rate / IsLittleEndian / Duration / Position / Size
- Summary: Query methods for audio properties. `IsStereo()` checks for multi-channel, `BytesPerFrame()` is sample width ├ù channels, `Rate()` returns sample rate (Hz), `IsLittleEndian()` indicates byte order, `Duration()` returns length in seconds, `Position()` getter returns current sample/byte position, `Position(uint32_t)` setter seeks to position, `Size()` returns total sample/byte count.
- All are virtual and called after `Open()` to understand decoder state and format

### StreamDecoder::Get
- Signature: `static std::unique_ptr<StreamDecoder> Get(FileSpecifier &File)`
- Purpose: Factory method; instantiate appropriate concrete decoder for the given file
- Inputs: `File` ΓÇô audio file to examine
- Outputs/Return: `std::unique_ptr<StreamDecoder>` ΓÇô newly constructed decoder, or null if file type not supported
- Side effects: Examines file to determine codec; allocates decoder instance
- Notes: Returns unique_ptr; caller owns the decoder lifetime

### Decoder::Frames
- Signature: `virtual int32 Frames() = 0`
- Purpose: Return total number of audio frames in the file (for pre-decoded sounds)
- Outputs/Return: `int32` frame count
- Notes: Frames = total samples ├╖ channels; more convenient than raw byte count for loaded/decoded audio

### Decoder::Get
- Signature: `static Decoder* Get(FileSpecifier &File)`
- Purpose: Factory method; instantiate appropriate concrete decoder for pre-decoded/frame-based access
- Inputs: `File` ΓÇô audio file to load and decode
- Outputs/Return: `Decoder*` ΓÇô newly constructed decoder, or null if file type not supported or decode failed
- Side effects: May load entire file into memory; allocates decoder instance
- Notes: Returns raw pointer (non-owning); caller must delete. Different semantics from `StreamDecoder::Get()`

## Control Flow Notes
This file is part of the sound initialization and playback pipeline:
1. **Sound Load Time**: Engine calls `StreamDecoder::Get()` or `Decoder::Get()` with a file path to obtain a decoder instance.
2. **Format Negotiation**: Caller queries `GetAudioFormat()`, `IsStereo()`, `Rate()`, `Duration()` to configure audio playback (sample rate conversion, channel mixing, etc.).
3. **Streaming Playback**: `Decode()` is called repeatedly (e.g., in frame update or I/O thread) to fill audio buffers incrementally.
4. **Seeking**: `Position()` supports random access (e.g., restarting a loop, skipping forward).
5. **Shutdown**: `Close()` or destructor releases resources.

The `Decoder` subclass is used for fully pre-decoded sounds (e.g., in-game effects), while `StreamDecoder` handles larger files (music, ambient tracks) decoded on demand.

## External Dependencies
- `<memory>` ΓÇô `std::unique_ptr` for factory return type
- `cseries.h` ΓÇô defines `uint8`, `int32`, `uint32_t` base types
- `FileHandler.h` ΓÇô defines `FileSpecifier` class for file abstraction
- `SoundManagerEnums.h` ΓÇô defines `AudioFormat` enum (`_8_bit`, `_16_bit`, `_32_float`)
