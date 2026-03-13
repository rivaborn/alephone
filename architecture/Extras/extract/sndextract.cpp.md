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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sound_definition` | struct | Sound metadata: code, offsets, permutation count, total/single length (defined elsewhere) |
| `sound_file_header` | struct | Output file header: version, tag, source/sound counts (defined elsewhere) |
| `SoundHeader` / `ExtSoundHeader` | struct | Mac OS sound data headers for format parsing (defined elsewhere) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sound_definitions` | array of `sound_definition` | static (via STATIC_DEFINITIONS include) | Definition template/master copy, loaded from `sound_definitions.h` |

## Key Functions / Methods

### main
- **Signature:** `void main(int argc, char **argv)`
- **Purpose:** Entry point; validates argument count and delegates to `build_sounds_file()`
- **Inputs:** Command-line argc/argv
- **Outputs/Return:** Exit code (0 on success, 1 on error)
- **Side effects:** Calls `build_sounds_file()` or exits with error message to stderr
- **Calls:** `build_sounds_file()`, `exit()`, `fprintf()`
- **Notes:** Expects at least 3 arguments: program name, destination, and one or more source files

### build_sounds_file
- **Signature:** `void build_sounds_file(char *destination_filename, short source_count, char **source_filenames)`
- **Purpose:** Orchestrates creation of the consolidated sound file; writes header, allocates definition arrays, processes each source file
- **Inputs:** Destination filename, count of source files, array of source filenames
- **Outputs/Return:** None (writes to file)
- **Side effects:** Creates/overwrites destination file, allocates heap memory, opens/closes resource files, calls I/O (fwrite, fseek)
- **Calls:** `fopen()`, `malloc()`, `assert()`, `memset()`, `fwrite()`, `strcpy()`, `c2pstr()`, `OpenResFile()`, `extract_sound_resources()`, `CloseResFile()`, `memcpy()`, `free()`, `fclose()`, `fprintf()`, `exit()`
- **Notes:** 
  - First source (index 0) is the "primary"; subsequent sources fall back to primary for missing sounds
  - Writes placeholder sound definitions early, then `extract_sound_resources()` updates them in-place
  - Allocates two definition arrays: `original_definitions` (copy from first source) and `working_definitions` (per-source copy)

### extract_sound_resources
- **Signature:** `static void extract_sound_resources(FILE *stream, long *definition_offset, short base_resource, struct sound_definition *original_definitions, struct sound_definition *working_definitions)`
- **Purpose:** Extracts sound resources from a single Macintosh resource file and writes audio data to the output stream; updates definition metadata with offsets
- **Inputs:** 
  - `stream`: Open output file handle
  - `definition_offset`: Pointer to current file offset for definition table (updated)
  - `base_resource`: Base resource ID (0 for primary source, 10000+ for alternates)
  - `original_definitions`: Primary source definitions (NULL for primary source itself, used for fallback)
  - `working_definitions`: Current source's definitions (modified in-place)
- **Outputs/Return:** None (modifies definitions and writes to stream)
- **Side effects:** 
  - Seeks to end of file, writes sound data (permutations), seeks back to definition offset, writes definition array
  - Allocates/releases Mac OS resource handles
  - Updates `*definition_offset` pointer
- **Calls:** `fseek()`, `ftell()`, `GetResource()`, `HLock()`, `GetSoundHeaderOffset()`, `fwrite()`, `ReleaseResource()`, `memcpy()`
- **Notes:** 
  - Iterates up to `MAXIMUM_PERMUTATIONS_PER_SOUND` per sound
  - Handles two encode types: `stdSH` (standard) and `extSH` (extended)
  - Skips sounds with `sound_code == NONE`
  - Uses `ALIGN_LONG()` to pad sound data to 4-byte boundaries
  - If no permutations found: clears sound_code (primary) or inherits from original_definitions (secondary)
  - Records `single_length` (first permutation) separately from `total_length` (all permutations)

## Control Flow Notes
- **Startup:** `main()` ΓåÆ `build_sounds_file()` writes file header and placeholder definitions
- **Processing loop:** For each source file ΓåÆ open resource fork ΓåÆ `extract_sound_resources()` extracts and writes sound groups, updates definitions
- **Shutdown:** Close output file, free memory, exit

The output file structure is: `[header] [definition_array_1] [definition_array_2] ... [sound_group_1] [sound_group_2] ...`

## External Dependencies
- **Macintosh Toolbox APIs:** `OpenResFile()`, `CloseResFile()`, `GetResource()`, `HLock()`, `ReleaseResource()`, `GetSoundHeaderOffset()`, `ResError()`, `c2pstr()` (string conversion)
- **Includes:**
  - `macintosh_cseries.h` (Mac-specific utilities, defines `Str255`, `NONE`, `assert()`, `halt()`)
  - `byte_swapping.h` (endianness handling, included but not directly used here)
  - `world.h`, `mysound.h` (sound-related types: `SoundHeader`, `ExtSoundHeader`, `Handle`, encode types)
  - `sound_definitions.h` (with `STATIC_DEFINITIONS` macro; provides `sound_definitions` array, `NUMBER_OF_SOUND_DEFINITIONS`, `NUMBER_OF_SOUND_SOURCES`, `SOUND_FILE_VERSION`, `SOUND_FILE_TAG`)
- **Standard C:** `stdio.h` (implied: `fprintf()`, `fopen()`, `fwrite()`, `fseek()`, `ftell()`, `fclose()`), `stdlib.h` (implied: `exit()`, `malloc()`, `free()`), `string.h` (`strcpy()`, `memset()`, `memcpy()`)
