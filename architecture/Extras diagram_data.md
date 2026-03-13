# Extras/extract/shapeextract.cpp
## File Purpose
Utility to extract shape resources from Macintosh resource files (.256 resources) and serialize them to a binary output file. Processes multiple collections of shape data and writes collection headers with embedded offsets and lengths.

## Core Responsibilities
- Parse command-line arguments (source and destination file paths)
- Open Macintosh resource file and validate access
- Extract shape resources in the ID ranges 128ΓÇô255 and 1128ΓÇô1255
- Compute and track file offsets and data lengths for each resource
- Write collection header metadata and resource data to binary output file
- Manage resource handles and clean up file handles

## External Dependencies
- **Macintosh Toolbox:** `OpenResFile()`, `CloseResFile()`, `GetResource()`, `HLock()`, `ReleaseResource()`, `GetHandleSize()`, `ResError()`, `c2pstr()`
- **Standard C:** `<string.h>` (strcpy), `<stdio.h>` (fopen, fwrite, fseek, ftell, fprintf, fclose)
- **Project headers:** `macintosh_cseries.h`, `shape_descriptors.h`, `shape_definitions.h` (defines `collection_header`, `MAXIMUM_COLLECTIONS`, etc.)

# Extras/extract/sndextract.cpp
## File Purpose
A utility program that consolidates sound resources from one or more Macintosh resource files into a single binary sound definition file. It extracts sound data ('snd ' resources), manages permutations, and writes a structured output file with headers and offset tables for engine consumption.

## Core Responsibilities
- Parse and validate command-line arguments (source and destination files)
- Allocate and manage sound definition metadata arrays
- Open/close Macintosh resource files and iterate through them
- Extract 'snd ' resources with permutation support (multi-variant sounds)
- Parse and handle two sound header formats (standard and extended)
- Calculate byte-aligned offsets and sizes for sound data
- Write consolidated binary output with header, definition tables, and sound groups
- Implement fallback behavior: secondary sources inherit missing sounds from primary source

## External Dependencies
- **Macintosh Toolbox APIs:** `OpenResFile()`, `CloseResFile()`, `GetResource()`, `HLock()`, `ReleaseResource()`, `GetSoundHeaderOffset()`, `ResError()`, `c2pstr()` (string conversion)
- **Includes:**
  - `macintosh_cseries.h` (Mac-specific utilities, defines `Str255`, `NONE`, `assert()`, `halt()`)
  - `byte_swapping.h` (endianness handling, included but not directly used here)
  - `world.h`, `mysound.h` (sound-related types: `SoundHeader`, `ExtSoundHeader`, `Handle`, encode types)
  - `sound_definitions.h` (with `STATIC_DEFINITIONS` macro; provides `sound_definitions` array, `NUMBER_OF_SOUND_DEFINITIONS`, `NUMBER_OF_SOUND_SOURCES`, `SOUND_FILE_VERSION`, `SOUND_FILE_TAG`)
- **Standard C:** `stdio.h` (implied: `fprintf()`, `fopen()`, `fwrite()`, `fseek()`, `ftell()`, `fclose()`), `stdlib.h` (implied: `exit()`, `malloc()`, `free()`), `string.h` (`strcpy()`, `memset()`, `memcpy()`)

# Extras/physics_patches.cpp
## File Purpose
A command-line utility that compares two physics WAD files and generates a delta/patch file containing only the byte-level differences. The patch can later be applied to the original file to reconstruct the new version, preserving the original file's checksum in the patch metadata.

## Core Responsibilities
- Parse command-line arguments (original file path, new file path, output patch file path)
- Open and read physics WAD files from disk
- Extract physics definition data from WAD structures
- Perform byte-by-byte comparison across all physics definition types
- Identify changed regions (first non-matching offset, contiguous length)
- Generate a delta WAD file containing only changed data with offset metadata
- Write patch file with parent checksum linkage for validation

## External Dependencies
- **Headers:** `macintosh_cseries.h` (Mac OS compat), `map.h`, `effects.h`, `projectiles.h`, `monsters.h`, `weapons.h`, `wad.h`, `items.h`, `mysound.h`, `media.h`, `tags.h`
- **WAD Functions (defined elsewhere):**
  - `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`
  - `extract_type_from_wad()`, `create_empty_wad()`, `append_data_to_wad()`, `free_wad()`
  - `create_wadfile()`, `open_wad_file_for_writing()`, `fill_default_wad_header()`, `write_wad_header()`, `write_wad()`, `write_directorys()`, `calculate_and_store_wadfile_checksum()`, `calculate_wad_length()`
- **Mac OS Functions:** `FSMakeFSSpec()`, `c2pstr()` (C string to Pascal string).
- **Standard C:** `fprintf()`, `strcpy()`, `exit()`, `malloc()`.
- **Macros:** `definitions[]` array (populated via included headers), `NUMBER_OF_DEFINITIONS` (count constant), `PATCH_FILE_TYPE`, `PHYSICS_DATA_VERSION`, `CURRENT_WADFILE_VERSION`.


