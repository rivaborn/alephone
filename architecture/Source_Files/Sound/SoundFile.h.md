# Source_Files/Sound/SoundFile.h

## File Purpose
Defines abstract and concrete interfaces for loading sound definitions and audio data from Marathon 1 and 2 sound files. Handles deserialization of sound metadata (headers, pitch, looping) and raw audio samples from both binary streams and resource forks.

## Core Responsibilities
- Define containers for sound metadata (`SoundInfo`, `SoundHeader`, `SoundDefinition`)
- Provide abstract sound file loading interface (`SoundFile`)
- Implement format-specific loaders for M1 (resource-based) and M2 (binary-based) sound files
- Parse and unpack System 7 sound headers (standard, extended, compressed)
- Load audio sample data with support for multiple permutations per sound
- Track looping parameters, pitch ranges, and playback flags

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SoundData` | typedef | `std::vector<uint8>` alias for raw audio sample bytes |
| `SoundInfo` | struct | Base sound metadata: format (8/16-bit), stereo/mono, endianness, rate, loop points, length |
| `SoundHeader` | class | Extends `SoundInfo`; unpacks System 7 sound headers; loads audio data from streams/resources |
| `SoundDefinition` | class | Sound metadata container: code, behavior, flags, pitch range, permutation offsets, total length |
| `SoundFile` | abstract class | Base interface for format-specific sound file loaders |
| `M1SoundFile` | class | Concrete loader for Marathon 1; uses resource forks via `OpenedResourceFile` |
| `M2SoundFile` | class | Concrete loader for Marathon 2; uses sequential binary format via `OpenedFile` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `stdSH` | uint8 (0x00) | SoundHeader private constant | Standard System 7 sound header type marker |
| `extSH` | uint8 (0xFF) | SoundHeader private constant | Extended System 7 sound header type marker |
| `cmpSH` | uint8 (0xFE) | SoundHeader private constant | Compressed System 7 sound header type marker |
| `bufferCmd` | uint16 (0x8051) | SoundHeader private constant | System 7 buffer command code |
| `MAXIMUM_PERMUTATIONS_PER_SOUND` | int (5) | SoundDefinition/M1SoundFile | Max audio variants per sound |

## Key Functions / Methods

### SoundInfo constructor
- **Signature:** `SoundInfo()`
- **Purpose:** Initialize sound metadata with safe defaults
- **Inputs:** None
- **Outputs/Return:** Initialized instance
- **Side effects:** None
- **Calls:** None
- **Notes:** All fields default to no-sound state: 8-bit mono, little-endian, zero rate/length

### SoundHeader::Load (BIStreamBE variant)
- **Signature:** `bool Load(BIStreamBE& stream)`
- **Purpose:** Deserialize a System 7 sound header from big-endian binary stream
- **Inputs:** Binary input stream positioned at header
- **Outputs/Return:** `true` if header unpacked successfully
- **Side effects:** Updates this object's SoundInfo fields; reads stream position
- **Calls:** `UnpackStandardSystem7Header`, `UnpackExtendedSystem7Header`
- **Notes:** Dispatches based on header type byte; handles standard and extended formats

### SoundHeader::LoadData (multiple overloads)
- **Signature:** `std::shared_ptr<SoundData> LoadData(BIStreamBE& stream)`, `LoadData(OpenedFile&)`, `LoadData(LoadedResource&)`
- **Purpose:** Deserialize raw audio samples following a sound header
- **Inputs:** Stream/file/resource positioned after header
- **Outputs/Return:** Shared pointer to vector of audio bytes; nullptr if load fails
- **Side effects:** Reads audio data; may call `ConvertSignedToUnsignedByte` for 8-bit signedΓåÆunsigned conversion
- **Calls:** Stream read operators, `ConvertSignedToUnsignedByte`
- **Notes:** Returns dynamically allocated data via smart pointer; respects `length` field from header

### SoundDefinition::Unpack
- **Signature:** `bool Unpack(OpenedFile&)`, `bool Unpack(BIStreamBE&)`
- **Purpose:** Deserialize sound definition metadata (flags, pitch, permutation offsets) from file or stream
- **Inputs:** Opened file or binary stream
- **Outputs/Return:** `true` if unpacked successfully; populates `sound_code`, `flags`, `chance`, pitch fields, offsets
- **Side effects:** Reads binary data; updates all public members
- **Calls:** Stream deserialization operators
- **Notes:** Must read exactly 64 bytes (HeaderSize); reads permutation offsets based on `permutations` count

### SoundDefinition::Load
- **Signature:** `bool Load(OpenedFile&, bool LoadPermutations)`
- **Purpose:** Load sound definition and optionally all permutation headers
- **Inputs:** Opened file, flag to load header data
- **Outputs/Return:** `true` if successful; populates `sounds` vector
- **Side effects:** Reads and caches `SoundHeader` objects; populates internal state
- **Calls:** Unpack, file seeking/reading via OpenedFile
- **Notes:** If `LoadPermutations` false, `sounds` remains empty; headers loaded on demand

### SoundFile::GetSoundDefinition
- **Signature:** `virtual SoundDefinition* GetSoundDefinition(int source, int sound_index) = 0`
- **Purpose:** Abstract method; retrieve metadata for a specific sound by source and index
- **Inputs:** Source (0 for M1; 0ΓÇôN for M2), sound index
- **Outputs/Return:** Pointer to SoundDefinition or nullptr if not found
- **Side effects:** None (or caching per implementation)
- **Calls:** None (implementation-defined)
- **Notes:** M1 ignores source parameter; M2 indexes into `sound_definitions` by source

### M1SoundFile::GetSoundDefinition
- **Signature:** `SoundDefinition* GetSoundDefinition(int source, int sound_index)`
- **Purpose:** Load or retrieve cached M1 sound definition from resource fork
- **Inputs:** `source` (ignored), `sound_index` (sound code to look up)
- **Outputs/Return:** Pointer to cached SoundDefinition; nullptr if not found
- **Side effects:** May trigger resource load; caches definitions in `definitions` map
- **Calls:** `OpenedResourceFile::Get`, `SoundDefinition::Unpack`
- **Notes:** Uses `sound_index` as resource ID; caches result to avoid re-parsing

### M2SoundFile::GetSoundDefinition
- **Signature:** `SoundDefinition* GetSoundDefinition(int source, int sound_index)`
- **Purpose:** Return M2 sound definition from pre-loaded sound_definitions table
- **Inputs:** `source` (selects row), `sound_index` (selects column)
- **Outputs/Return:** Pointer to SoundDefinition or nullptr
- **Side effects:** None
- **Calls:** None
- **Notes:** Direct lookup in 2D vector; M2 format pre-loads all definitions at Open()

## Control Flow Notes

**Initialization:** `Open()` is called before any sound access. M1SoundFile defers resource loading; M2SoundFile reads the entire header (260 bytes) and populates `sound_definitions` upfront.

**Sound Retrieval:** Caller invokes `GetSoundDefinition()`, then `GetSoundHeader()` and `GetSoundData()` for a specific permutation. M1 lazily loads data on demand; M2 pre-loaded headers are returned directly.

**Deserialization:** `Load()` and `Unpack()` are called during construction or lazy-loading; they read from binary streams or resource forks using `BIStreamBE`, `OpenedFile`, or `LoadedResource` abstractions.

**Cleanup:** `Close()` releases open file handles; `Unload()` clears cached sound data.

## External Dependencies

- **AStream.h**: `AIStreamBE` for deserializing big-endian data (not directly used here; BStream is preferred)
- **BStream.h**: `BIStreamBE` for big-endian binary deserialization
- **FileHandler.h**: `OpenedFile`, `OpenedResourceFile`, `LoadedResource`, `FileSpecifier` for file and resource access
- **SoundManagerEnums.h**: `AudioFormat` enum, sound codes, behavior flags
- **&lt;memory&gt;**: `std::shared_ptr`, `std::unique_ptr`
- **&lt;vector&gt;**, **&lt;map&gt;**: Container types for sound data and metadata caching
