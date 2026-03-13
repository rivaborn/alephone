# Source_Files/Sound/SoundsPatch.h

## File Purpose
Defines the interface for patching and overriding game sound definitions at runtime. The `SoundsPatches` class allows sounds from external sources (files or binary streams) to override base game sound definitions, enabling modding/customization without modifying core assets.

## Core Responsibilities
- Manage a collection of sound definition patches indexed by source and sound index
- Load patch data from file specifiers or binary streams (big-endian format)
- Retrieve patched `SoundDefinition` objects for a given source and sound index
- Retrieve audio sample data (`SoundData`) for a given sound definition and permutation
- Provide global patch data accessors (set/get/load functions)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SoundsPatches` | class | Container managing all active sound patches |
| `SoundDefinitionPatch` | class (fwd-decl) | Individual patch record (definition and metadata) |
| `SoundDefinition` | class (external) | Patched sound definition containing headers and metadata |
| `SoundData` | typedef (external) | Raw audio sample data (`std::vector<uint8>`) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sounds_patches` | `SoundsPatches` | extern global | Singleton instance managing all active sound patches |

## Key Functions / Methods

### add (FileSpecifier overload)
- Signature: `bool add(FileSpecifier& file_specifier)`
- Purpose: Load a sound patch from a file on disk
- Inputs: File specifier pointing to patch source
- Outputs/Return: `bool` ΓÇö success/failure
- Side effects: Updates `definition_patches` map; may allocate/load audio data
- Calls: (implementation not visible)
- Notes: File format expected to match internal patch structure

### add (BIStreamBE overload)
- Signature: `bool add(BIStreamBE& stream)`
- Purpose: Load a sound patch from a big-endian binary stream
- Inputs: Open binary input stream (big-endian byte order)
- Outputs/Return: `bool` ΓÇö success/failure
- Side effects: Updates `definition_patches` map; advances stream position
- Calls: (implementation not visible)
- Notes: Allows streaming patches from memory buffers or network

### get_definition
- Signature: `SoundDefinition* get_definition(int source, int sound_index)`
- Purpose: Retrieve a patched sound definition by source and sound index
- Inputs: `source` (sound bank/source ID), `sound_index` (index within source)
- Outputs/Return: Pointer to patched `SoundDefinition` (or nullptr if not patched)
- Side effects: None (read-only query)
- Calls: (implementation not visible)
- Notes: Returns null if no patch exists for this (source, index) pair

### get_sound_data
- Signature: `std::shared_ptr<SoundData> get_sound_data(SoundDefinition* definition, int permutation)`
- Purpose: Retrieve audio sample data for a given sound definition and permutation variant
- Inputs: `definition` (sound definition), `permutation` (variant index, 0ΓÇô4 per `MAXIMUM_PERMUTATIONS_PER_SOUND`)
- Outputs/Return: Shared pointer to audio sample vector (or nullptr if unavailable)
- Side effects: May load data from disk on first access
- Calls: (implementation not visible)
- Notes: Permutations allow multiple audio variants per logical sound

### clear
- Signature: `void clear()`
- Purpose: Remove all active patches and free resources
- Inputs: None
- Outputs/Return: None
- Side effects: Clears `definition_patches` map; deallocates all patched definitions and audio data
- Calls: (implementation not visible)
- Notes: Used during shutdown or reload cycles

### Module-level functions
- `set_sounds_patch_data()`, `get_sounds_patch_data()`, `load_sounds_patch_data()` ΓÇö manage global patch data buffer and initialization (implementation details not inferable)

## Control Flow Notes
- **Initialization**: `load_sounds_patch_data()` is likely called during engine startup to populate the global patch data
- **Patch loading**: Calls to `add()` occur after initialization to register specific patches
- **Runtime**: `get_definition()` and `get_sound_data()` are called during sound playback to retrieve patched assets
- **Shutdown**: `clear()` may be called to unload patches between level changes or during cleanup

## External Dependencies
- `#include <map>` ΓÇö standard library container (indexed by `(source, sound_index)` pair)
- `#include <memory>` ΓÇö for `std::shared_ptr<SoundData>`
- `"BStream.h"` ΓÇö provides `BIStreamBE` (big-endian binary input stream)
- `"SoundFile.h"` ΓÇö provides `SoundDefinition`, `SoundData`, `FileSpecifier` (defined elsewhere)
- Forward declarations: `FileHandler`, `SoundDefinitionPatch` (implementations in other translation units)
