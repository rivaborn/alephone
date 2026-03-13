# Source_Files/Sound/SoundFile.cpp

## File Purpose
Implements sound file loading for the Aleph One game engine, supporting both classic Mac System 7 sound resources (M1) and the modern M2 sound format. Handles parsing audio headers, loading raw audio data, managing endianness/format conversions, and lazy-loading permutations (variants) of sounds.

## Core Responsibilities
- Parse System 7 sound headers in three variants (standard, extended, compressed)
- Load raw PCM audio data with format conversions (signedΓåÆunsigned, endianness)
- Implement M1 sound file interface (Mac resource-based audio)
- Implement M2 sound file interface (modern format with multiple sources)
- Manage sound definitions and permutations (up to 5 variants per sound)
- Cache sound headers and data to minimize re-parsing
- Handle I/O errors and stream failures gracefully

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SoundInfo | class | Base metadata: audio format, sample rate, loop points, stereo flag |
| SoundHeader | class | Parses/holds a single sound header; loads audio data |
| SoundData | typedef | `std::vector<uint8>` for raw audio samples |
| SoundDefinition | class | Metadata + array of SoundHeader objects; one per permutation |
| SoundFile | class | Abstract interface; polymorphic base for M1/M2 implementations |
| M1SoundFile | class | Concrete implementation for Mac System 7 resources |
| M2SoundFile | class | Concrete implementation for M2 format files |

## Global / File-Static State
None.

## Key Functions / Methods

### SoundHeader::UnpackStandardSystem7Header
- **Signature:** `bool UnpackStandardSystem7Header(BIStreamBE &header)`
- **Purpose:** Parse a 22-byte Mac System 7 standard sound header.
- **Inputs:** Big-endian binary stream positioned at header start.
- **Outputs/Return:** `true` if parse succeeds; sets `length`, `rate`, `loop_start`, `loop_end`.
- **Side effects:** Modifies member variables; assumes 8-bit mono; throws `basic_bstream::failure` on read error.
- **Notes:** Hardcoded for 8-bit mono, little_endian=false.

### SoundHeader::UnpackExtendedSystem7Header
- **Signature:** `bool UnpackExtendedSystem7Header(BIStreamBE &header)`
- **Purpose:** Parse a 64-byte extended System 7 header; supports 8/16-bit, mono/stereo, and compression info.
- **Inputs:** Big-endian stream.
- **Outputs/Return:** `true` on success; sets audio format, channels, sample size, signed flag.
- **Side effects:** Reads ~64 bytes from stream; rejects non-AIFF or non-signed compression.
- **Notes:** Validates `format == FOUR_CHARS_TO_INT('t','w','o','s')` and `comp_id == -1` for AIFF-only compressed audio.

### SoundHeader::Load (overloaded for BIStreamBE)
- **Signature:** `bool Load(BIStreamBE& s)`
- **Purpose:** Detect header type (standard/extended/compressed) via byte at offset 20, then dispatch.
- **Inputs:** Stream with 64+ bytes available.
- **Outputs/Return:** `true` if header parsed; sets `data_offset` (22 or 64).
- **Side effects:** Seeks within stream to peek at encoding byte without consuming it.
- **Notes:** Returns false if encoding type not recognized.

### SoundHeader::LoadData (overloaded for BIStreamBE)
- **Signature:** `std::shared_ptr<SoundData> LoadData(BIStreamBE& s)`
- **Purpose:** Read raw audio samples from stream after header has been loaded.
- **Inputs:** Stream positioned after header; `data_offset` and `length` must be set.
- **Outputs/Return:** Shared pointer to audio buffer; null on failure.
- **Side effects:** Converts signedΓåÆunsigned for 8-bit audio; byte-swaps for little-endian 16-bit if platform differs from file.
- **Calls:** `ConvertSignedToUnsignedByte`, `byte_swap_memory`, `PlatformIsLittleEndian()`.
- **Notes:** Silently returns null if `data_offset==0` or `length<=0`.

### SoundDefinition::Unpack
- **Signature:** `bool Unpack(BIStreamBE& header)` (second overload)
- **Purpose:** Parse a 64-byte sound definition record (metadata, not audio data).
- **Inputs:** Stream with definition struct.
- **Outputs/Return:** `true` if valid; populates `sound_code`, `behavior_index`, `permutations`, offsets, etc.
- **Side effects:** Resizes `sound_offsets` to `MAXIMUM_PERMUTATIONS_PER_SOUND` (5).
- **Notes:** Ignores final two uint32 fields (last_played, reserved).

### SoundDefinition::Load
- **Signature:** `bool Load(OpenedFile &SoundFile, bool LoadPermutations)`
- **Purpose:** Load all (or just one) SoundHeader for each permutation.
- **Inputs:** Opened file; flag to load all permutations or only first.
- **Outputs/Return:** `true` if all requested headers loaded; clears `sounds` vector on failure.
- **Side effects:** Seeks to `group_offset + sound_offsets[i]` for each permutation.
- **Notes:** Lazy-loads headers; audio data not read until `LoadData()` is called.

### M2SoundFile::Open
- **Signature:** `bool Open(FileSpecifier &SoundFileSpec)`
- **Purpose:** Open and parse M2 format sound file (modern format with multiple sources).
- **Inputs:** FileSpecifier pointing to .snd2 file.
- **Outputs/Return:** `true` if valid M2 file and all definitions loaded.
- **Side effects:** Reads header (260 bytes), allocates 2D vector of sound definitions, loads all headers, keeps file open in `opened_sound_file`.
- **Calls:** Parses version/tag/counts; validates version Γêê {0,1} and tag == 'snd2'.
- **Notes:** If `sound_count==0`, sets it equal to `source_count` and `source_count=1` for backward compatibility.

### M1SoundFile::GetSoundDefinition
- **Signature:** `SoundDefinition* GetSoundDefinition(int, int sound_index)`
- **Purpose:** Retrieve or lazily create a sound definition from Mac resource file.
- **Inputs:** Ignored source (always 1 for M1); sound index.
- **Outputs/Return:** Pointer to cached definition, or null if resource not found.
- **Side effects:** Creates definition on demand; counts permutations by probing resource file; caches in `definitions` map.
- **Notes:** Permutation count capped at `MAXIMUM_PERMUTATIONS_PER_SOUND` (5); all get `behavior_index=2` (sound_is_loud).

### M1SoundFile::GetSoundData
- **Signature:** `std::shared_ptr<SoundData> M1SoundFile::GetSoundData(SoundDefinition* definition, int permutation)`
- **Purpose:** Load audio data for a single permutation from Mac resource.
- **Inputs:** Definition pointer; permutation index.
- **Outputs/Return:** Shared pointer to audio buffer.
- **Side effects:** Fetches resource, caches it in `cached_rsrc`, calls `GetSoundHeader()` and `LoadData()`.
- **Notes:** Single-level resource cache (only one resource resident at a time).

## Control Flow Notes
**Initialization:** Game loads sound file via `M1SoundFile::Open` or `M2SoundFile::Open`.  
**Query:** Engine calls `GetSoundDefinition(source, index)` to retrieve metadata.  
**Playback prep:** Engine calls `GetSoundData(definition, permutation)` to fetch audio samples on demand.  
**Lazy-loading strategy:** Headers loaded eagerly; audio data loaded lazily to reduce memory footprint.

## External Dependencies
- **BStream.h**: Big-endian binary stream readers (`BIStreamBE`, `AIStreamBE`).
- **FileHandler.h**: File abstractions (`OpenedFile`, `LoadedResource`, `FileSpecifier`, `opened_file_device`).
- **SoundManagerEnums.h**: `AudioFormat` enum.
- **Logging.h**: `logWarning()` macro.
- **byte_swapping.h**: `byte_swap_memory()`.
- **boost/iostreams**: Stream buffer wrapper for memory/file sources.
- **cseries.h / cstypes.h**: Integer typedefs (`uint8`, `int16`, `_fixed`), `FOUR_CHARS_TO_INT` macro, `PlatformIsLittleEndian()`.
- **Standard library:** `<memory>`, `<vector>`, `<map>`, `<assert.h>`, `<utility>`.
