# Source_Files/Sound/ReplacementSounds.h

## File Purpose
Provides a singleton registry for managing MML-specified external sound replacements in the Aleph One game engine. Enables loading custom audio files to override built-in game sounds via index and slot identifiers.

## Core Responsibilities
- Define `ExternalSoundHeader` to encapsulate metadata and loading logic for external sound files
- Provide `SoundOptions` struct to pair file paths with sound headers
- Implement `SoundReplacements` singleton to store, retrieve, and manage the collection of sound replacements
- Support add/remove/reset operations on the replacement registry

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ExternalSoundHeader` | class | Extends `SoundInfo` to load audio data from external files via `LoadExternal()` |
| `SoundOptions` | struct | Bundles a `FileSpecifier` and `ExternalSoundHeader` for a single replacement entry |
| `SoundReplacements` | class | Singleton managing a hash-based registry of sound replacements keyed by `(Index, Slot)` pairs |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `SoundReplacements*` | static (within `instance()`) | Singleton instance of the sound replacement registry |

## Key Functions / Methods

### ExternalSoundHeader::LoadExternal
- **Signature:** `std::shared_ptr<SoundData> LoadExternal(FileSpecifier& File)`
- **Purpose:** Load audio sample data from an external file specified by `File`
- **Inputs:** `FileSpecifier& File` ΓÇô file path/handle for the audio resource
- **Outputs/Return:** `std::shared_ptr<SoundData>` ΓÇô shared pointer to loaded PCM data
- **Side effects:** File I/O, memory allocation for audio buffer
- **Calls:** Likely calls into SoundFile hierarchy or file reading system (not visible in this file)
- **Notes:** Inherits audio format metadata from `SoundInfo` base class

### SoundReplacements::instance
- **Signature:** `static inline SoundReplacements* instance()`
- **Purpose:** Thread-unsafe lazy singleton accessor
- **Inputs:** None
- **Outputs/Return:** Pointer to the static `SoundReplacements` instance
- **Side effects:** Allocates the singleton on first call only
- **Calls:** None
- **Notes:** Simple static initializer pattern; not thread-safe

### SoundReplacements::GetSoundOptions
- **Signature:** `SoundOptions* GetSoundOptions(short Index, short Slot)`
- **Purpose:** Retrieve replacement sound options for a given sound index and slot
- **Inputs:** `Index` (sound ID), `Slot` (permutation/variant)
- **Outputs/Return:** Pointer to `SoundOptions` or null if not found
- **Side effects:** Hash table lookup only
- **Calls:** `m_hash.find()` or equivalent unordered_map operation (hidden)
- **Notes:** Index and Slot are packed into a pair key for lookup

### SoundReplacements::Add / Remove / Reset
- **Purpose:** Modify the replacement registry (add entry, remove entry by key, clear all)
- **Inputs (Add):** `SoundOptions& Data`, `short Index`, `short Slot`
- **Inputs (Remove):** `short Index`, `short Slot`
- **Side effects:** Hash table mutation; potential memory reallocation
- **Notes:** No visible bounds checking or duplicate-key detection

## Control Flow Notes
This file is part of the sound initialization / resource loading pipeline. `SoundReplacements` is likely queried during sound playback requests to check if an external replacement exists before falling back to built-in sound data. The singleton pattern suggests global, application-lifetime scope. No frame-update or render-loop involvement is evident.

## External Dependencies
- **`SoundFile.h`** ΓÇô provides `SoundInfo` base class, `SoundData` typedef (`std::vector<uint8>`), and `FileSpecifier`
- **`boost/unordered_map.hpp`** ΓÇô hash map container for fast lookup by `(Index, Slot)` pair
- **Standard C++ library** ΓÇô `<string>`, `<memory>` (smart pointers)
