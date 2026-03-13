# Source_Files/Sound/SoundsPatch.cpp

## File Purpose
Implements sound patching functionality that loads replacement sound definitions and audio data from binary streams or files. Enables overriding built-in sound assets with patched versions without modifying the original sound files.

## Core Responsibilities
- Parse binary sound patch streams containing "sndc" chunks with source/index pairs
- Load sound definition headers and audio data permutations from patch data
- Manage a global collection of active sound patches indexed by (source, index)
- Provide lookup API to retrieve patched sound definitions and audio data
- Integrate with the replacement sounds system to remove old definitions during patching

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SoundDefinitionPatch` | struct | Pairs a `SoundDefinition` with its loaded audio `SoundData` permutations |
| `SoundsPatch` | class | Parses a single patch stream and applies patches to the global collection |
| `SoundsPatches` | class (external) | Singleton container holding all active patches, indexed by (source, index) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sounds_patches` | `SoundsPatches` | global | Global singleton instance managing all loaded patches |
| `sounds_patch` | `std::vector<uint8_t>` | static | Raw binary patch data buffer |

## Key Functions / Methods

### `set_sounds_patch_data`
- Signature: `void set_sounds_patch_data(const uint8_t* data, size_t length)`
- Purpose: Store raw binary patch data for later parsing
- Inputs: Binary buffer and size
- Outputs/Return: None
- Side effects: Replaces contents of global `sounds_patch` vector
- Calls: `std::vector<uint8_t>::assign`

### `get_sounds_patch_data`
- Signature: `uint8_t* get_sounds_patch_data(size_t& length)`
- Purpose: Retrieve the stored raw patch data
- Inputs: Output parameter for length
- Outputs/Return: Pointer to data (or nullptr if empty); sets length parameter
- Side effects: None
- Calls: `std::vector<uint8_t>::size`, `std::vector<uint8_t>::data`

### `load_sounds_patch_data`
- Signature: `void load_sounds_patch_data()`
- Purpose: Parse the raw patch data into the global `sounds_patches` singleton
- Inputs: None (uses global `sounds_patch` vector)
- Outputs/Return: None
- Side effects: Populates `sounds_patches` with parsed patches
- Calls: `SoundsPatches::add`

### `SoundDefinitionPatch::load`
- Signature: `bool SoundDefinitionPatch::load(BIStreamBE& stream)`
- Purpose: Load sound headers and audio data for all permutations of a sound definition
- Inputs: Big-endian stream reader positioned at first permutation
- Outputs/Return: True on success; false if any permutation fails
- Side effects: Populates `data` vector with loaded `SoundData` permutations; modifies stream position repeatedly
- Calls: `SoundHeader::Load`, `SoundHeader::LoadData`, `std::ios_base` seek operations
- Notes: Seeks back to base offset between each permutation; definition group offsets are ignored in patches

### `SoundsPatch::load`
- Signature: `bool SoundsPatch::load(BIStreamBE& stream)`
- Purpose: Parse a complete patch stream containing one or more "sndc" chunks
- Inputs: Big-endian stream reader
- Outputs/Return: True on success; false if magic number or unpacking fails
- Side effects: Populates `definition_patches` map; modifies stream position
- Calls: `SoundDefinition::Unpack`, `SoundDefinitionPatch::load`, `SoundsPatch::apply`
- Notes: Each chunk contains source (uint16), index (uint16), definition, permutation sizes, and sound data. Anvil format includes redundant permutation size list.

### `SoundsPatch::apply`
- Signature: `void SoundsPatch::apply(std::map<std::pair<int, int>, SoundDefinitionPatch>& patches)`
- Purpose: Merge loaded patches into the global patch collection
- Inputs: Patches map to merge
- Outputs/Return: None
- Side effects: Removes old sound definitions via `SoundReplacements::Remove` for each permutation; merges and swaps patch maps
- Calls: `SoundReplacements::instance`, `std::map::merge`, `std::swap`

### `SoundsPatches::add` (stream overload)
- Signature: `bool SoundsPatches::add(BIStreamBE& stream)`
- Purpose: Load and apply patches from a stream
- Inputs: Big-endian stream reader
- Outputs/Return: True on success; false if parsing fails
- Side effects: Updates global patch collection
- Calls: `SoundsPatch::load`, `SoundsPatch::apply`

### `SoundsPatches::add` (file overload)
- Signature: `bool SoundsPatches::add(FileSpecifier& file_specifier)`
- Purpose: Load and apply patches from a file
- Inputs: File specification
- Outputs/Return: True on success; false if file open fails
- Side effects: Updates global patch collection
- Calls: `FileSpecifier::Open`, `SoundsPatches::add` (stream variant)

### `SoundsPatches::get_definition`
- Signature: `SoundDefinition* SoundsPatches::get_definition(int source, int sound_index)`
- Purpose: Retrieve patched sound definition by source and index
- Inputs: Source ID and sound index
- Outputs/Return: Pointer to `SoundDefinition` if patch exists; nullptr otherwise
- Side effects: None
- Calls: `std::map::find`

### `SoundsPatches::get_sound_data`
- Signature: `std::shared_ptr<SoundData> SoundsPatches::get_sound_data(SoundDefinition* definition, int permutation)`
- Purpose: Retrieve audio data for a patched sound by matching definition and permutation
- Inputs: Sound definition pointer and permutation index
- Outputs/Return: Shared pointer to audio data; nullptr if not found or permutation out of range
- Side effects: None
- Calls: None directly (linear search through `definition_patches`)
- Notes: Matches by `sound_code` and `total_length` to find the corresponding patch data

### `SoundsPatches::clear`
- Signature: `void SoundsPatches::clear()`
- Purpose: Unload all patches
- Inputs: None
- Outputs/Return: None
- Side effects: Clears the `definition_patches` map
- Calls: `std::map::clear`

## Control Flow Notes
- **Initialization**: Raw patch data is set via `set_sounds_patch_data()`, then parsed via `load_sounds_patch_data()`, which invokes `SoundsPatches::add()`.
- **Patching**: At load time, `SoundsPatch::apply()` removes existing definitions from `SoundReplacements` and merges new patches into the global map.
- **Lookup**: Game code queries `get_definition()` and `get_sound_data()` during sound playback to retrieve patched assets.
- **Shutdown**: `clear()` discards all patches.

## External Dependencies
- **Boost.IOStreams**: `boost::iostreams::array_source`, `boost::iostreams::stream_buffer` for memory-mapped streaming
- **BStream.h**: `BIStreamBE` (big-endian binary stream reader)
- **SoundFile.h**: `SoundDefinition`, `SoundHeader`, `SoundData`, `SoundInfo`
- **ReplacementSounds.h**: `SoundReplacements` singleton for coordinating patch application
- **FileHandler.h**: `FileSpecifier`, `OpenedFile`, `opened_file_device` for file I/O
