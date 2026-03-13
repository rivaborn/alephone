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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `collection_header` | struct | Metadata for shape collections; includes offset/length fields for 8-bit and 16-bit variants (defined in shape_definitions.h) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `collection_headers` | `struct collection_header[MAXIMUM_COLLECTIONS]` | global | Array holding collection metadata; written to output file twice (initially as placeholder, then updated with final offsets) |

## Key Functions / Methods

### main
- **Signature:** `void main(int argc, char **argv)`
- **Purpose:** Entry point; validates arguments, opens resource file, orchestrates extraction, and closes files.
- **Inputs:** `argc`, `argv[1]` (source resource file path), `argv[2]` (destination binary file path)
- **Outputs/Return:** None (exits with status 0 on success, 1 on error)
- **Side effects:** Opens resource file via `OpenResFile()`, opens/writes to destination file via `fopen()`, calls `extract_shape_resources()`, closes resource and destination files.
- **Calls:** `strcpy()`, `c2pstr()`, `OpenResFile()`, `fopen()`, `extract_shape_resources()`, `fclose()`, `CloseResFile()`, `ResError()`, `fprintf()`, `exit()`
- **Notes:** Requires Macintosh system calls; uses 1995-era Mac toolbox API. Error handling prints to stderr and exits immediately.

### extract_shape_resources
- **Signature:** `static void extract_shape_resources(FILE *stream)`
- **Purpose:** Iterates over all collections, writes placeholder headers to output, extracts resources for each collection (8-bit and 16-bit variants), then rewrites headers with computed offsets.
- **Inputs:** `stream` (open FILE* for binary output)
- **Outputs/Return:** None (modifies `collection_headers[]` in-place and writes to stream)
- **Side effects:** Writes to stream at current position, seeks to file start, rewrites headers.
- **Calls:** `fwrite()`, `add_resource()`, `fseek()`
- **Notes:** Two-pass write pattern: placeholder headers (required to know header size before resource data), then resource extraction, then final header rewrite. Loop index `i` maps to resource IDs via `128+i` and `1128+i`.

### add_resource
- **Signature:** `static void add_resource(short id, long *offset, long *length, FILE *stream)`
- **Purpose:** Loads a single shape resource by ID, records its file offset and length in the collection header, and writes it to the output stream.
- **Inputs:** `id` (resource ID), `offset`/`length` (pointers to header fields to populate), `stream` (output file)
- **Outputs/Return:** None (modifies `*offset` and `*length`, writes to stream)
- **Side effects:** Calls `GetResource()`, `HLock()`, `ftell()`, `fwrite()`, `ReleaseResource()`. On missing resource, sets `*offset = -1` and `*length = 0`.
- **Calls:** `GetResource()`, `HLock()`, `ftell()`, `GetHandleSize()`, `fwrite()`, `ReleaseResource()`
- **Notes:** Uses Mac Toolbox handle API; assumes `.256` resource type. Offset snapshot before write allows subsequent resources to be appended sequentially.

## Control Flow Notes
Classic tool initialization: parse arguments ΓåÆ open source file ΓåÆ write headers (placeholder) ΓåÆ extract and write resource data ΓåÆ rewind and update headers ΓåÆ close files ΓåÆ exit. All shape data is streamed sequentially to the binary output file.

## External Dependencies
- **Macintosh Toolbox:** `OpenResFile()`, `CloseResFile()`, `GetResource()`, `HLock()`, `ReleaseResource()`, `GetHandleSize()`, `ResError()`, `c2pstr()`
- **Standard C:** `<string.h>` (strcpy), `<stdio.h>` (fopen, fwrite, fseek, ftell, fprintf, fclose)
- **Project headers:** `macintosh_cseries.h`, `shape_descriptors.h`, `shape_definitions.h` (defines `collection_header`, `MAXIMUM_COLLECTIONS`, etc.)
