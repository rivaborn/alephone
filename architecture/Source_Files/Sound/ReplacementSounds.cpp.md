# Source_Files/Sound/ReplacementSounds.cpp

## File Purpose
Implements external sound file loading and management of replacement sounds for the Aleph One engine. Provides APIs to load audio files from disk, decode them, and associate them with game sound indices via a singleton registry.

## Core Responsibilities
- Load external audio files using pluggable decoders and extract audio format metadata
- Store replacement sound options indexed by sound index and slot pair
- Unload original sounds when replacements are added
- Query replacement sound options by index
- Reset or selectively remove sound replacements with proper cleanup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ExternalSoundHeader` | class | Extends `SoundInfo`; loads external sound files and populates audio metadata |
| `SoundOptions` | struct | Pairs a `FileSpecifier` with loaded `ExternalSoundHeader` metadata |
| `SoundReplacements` | class | Singleton registry managing the mapping of sound indices to replacement options |
| `key` (typedef) | typedef | `std::pair<short, short>` for (Index, Slot) pair used as hash map key |

## Global / File-Static State
None (state is managed as instance members in the `SoundReplacements` singleton).

## Key Functions / Methods

### ExternalSoundHeader::LoadExternal
- **Signature:** `std::shared_ptr<SoundData> LoadExternal(FileSpecifier& File)`
- **Purpose:** Load and decode an external audio file, populate audio format metadata.
- **Inputs:** `File` ΓÇô file specifier pointing to the audio file.
- **Outputs/Return:** `shared_ptr<SoundData>` containing decoded PCM audio; null/empty on failure.
- **Side effects:** Mutates member variables (`length`, `audio_format`, `stereo`, `bytes_per_frame`, `little_endian`, `rate`); allocates memory for decoded audio.
- **Calls:** `Decoder::Get()`, `Decoder::Frames()`, `Decoder::BytesPerFrame()`, `Decoder::Rewind()`, `Decoder::Decode()`, `Decoder::GetAudioFormat()`, `Decoder::IsStereo()`, `Decoder::IsLittleEndian()`, `Decoder::Rate()`.
- **Notes:** Returns early if decoder is unavailable or if decoded length is zero. Clears `length` and resets `p` on decode failure.

### SoundReplacements::GetSoundOptions
- **Signature:** `SoundOptions* GetSoundOptions(short Index, short Slot)`
- **Purpose:** Retrieve replacement sound options by index and slot.
- **Inputs:** `Index` ΓÇô sound index; `Slot` ΓÇô sound slot.
- **Outputs/Return:** Pointer to `SoundOptions` in the hash map, or null pointer if not found.
- **Side effects:** None.
- **Calls:** `m_hash.find()`, `key()` constructor.
- **Notes:** Returns raw pointer into hash map; caller must handle null case.

### SoundReplacements::Add
- **Signature:** `void Add(const SoundOptions& Data, short Index, short Slot)`
- **Purpose:** Register a replacement sound, displacing any prior sound at that index.
- **Inputs:** `Data` ΓÇô replacement options; `Index`, `Slot` ΓÇô registration key.
- **Outputs/Return:** None.
- **Side effects:** Unloads original sound via `SoundManager::UnloadSound()`; inserts/overwrites hash map entry.
- **Calls:** `SoundManager::instance()`, `SoundManager::UnloadSound()`.
- **Notes:** Always unloads the original before replacing, regardless of whether a prior replacement exists.

### SoundReplacements::Reset
- **Signature:** `void Reset()`
- **Purpose:** Clear all registered replacement sounds and unload their originals.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Unloads all sounds referenced in the hash map; clears the map.
- **Calls:** `SoundManager::instance()`, `SoundManager::UnloadSound()`, `m_hash.clear()`.
- **Notes:** Iterates over all entries to unload before clearing.

### SoundReplacements::Remove
- **Signature:** `void Remove(short Index, short Slot)`
- **Purpose:** Unregister a specific replacement sound.
- **Inputs:** `Index`, `Slot` ΓÇô key to remove.
- **Outputs/Return:** None.
- **Side effects:** Unloads sound; erases hash map entry.
- **Calls:** `SoundManager::instance()`, `SoundManager::UnloadSound()`, `m_hash.erase()`.
- **Notes:** None.

## Control Flow Notes
This module is part of **initialization/configuration**, not the main frame loop. Called when parsing MML sound replacement directives (see `parse_mml_sounds` in `SoundManager.h`). Sound replacements are decoupled from playback; the `SoundReplacements` registry is queried before playback to swap in replacement audio.

## External Dependencies
- **Decoder.h** ΓÇô `Decoder::Get()`, `Decoder::Frames()`, `Decoder::BytesPerFrame()`, `Decoder::Decode()`, `Decoder::GetAudioFormat()`, `Decoder::IsStereo()`, `Decoder::IsLittleEndian()`, `Decoder::Rate()`, `Decoder::Rewind()`
- **SoundManager.h** ΓÇô `SoundManager::instance()`, `SoundManager::UnloadSound()`
- **SoundFile.h** ΓÇô `SoundData`, `SoundInfo` (base class)
- **boost/unordered_map.hpp** ΓÇô hash map container
- **FileHandler.h** (via header) ΓÇô `FileSpecifier` type
