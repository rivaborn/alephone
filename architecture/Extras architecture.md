# Subsystem Overview

## Purpose
The Extras subsystem contains command-line utility programs that extract and convert Marathon's legacy Macintosh resource data into consolidated binary formats usable by the engine. This includes shape geometry extraction, sound resource consolidation, and physics WAD patch generation for development and build-time data preparation.

## Key Files
| File | Role |
|------|------|
| `Extras/extract/shapeextract.cpp` | Extracts shape resources from Macintosh .256 resource files and writes consolidated binary shape collections with embedded offset tables |
| `Extras/extract/sndextract.cpp` | Consolidates sound resources ('snd ') from one or more Macintosh resource files with permutation support into a single binary sound definition file |
| `Extras/physics_patches.cpp` | Generates delta WAD patch files by byte-level comparison of original and modified physics WAD files with checksum linkage |

## Core Responsibilities
- Extract shape geometry data from Macintosh resources (IDs 128ΓÇô255, 1128ΓÇô1255) with computed offset and length tracking
- Consolidate sound resources with permutation support and parse legacy sound header formats (standard and extended)
- Calculate byte-aligned offsets and sizes for efficient engine resource loading
- Perform byte-by-byte comparison of physics WAD files to identify and isolate changed regions
- Write consolidated binary output files with collection/definition headers and embedded metadata tables
- Manage Macintosh resource file I/O and handle lifecycle (OpenResFile, HLock, ReleaseResource, CloseResFile)
- Implement fallback sound inheritance (secondary resource sources inherit missing sounds from primary source)
- Generate patches with parent file checksum linkage for validation

## Key Interfaces & Data Flow
**Exposes:**
- Consolidated binary shape files with collection headers for engine shape loading
- Consolidated binary sound files with definition tables and permutation metadata
- Physics WAD delta patch files linked to parent file checksums

**Consumes:**
- Macintosh resource files (.rsrc, resource forks) via Mac Toolbox APIs
- Physics WAD files (original and modified versions) from disk
- Command-line arguments specifying source and destination file paths

## Runtime Role
These are off-line build-time utilities that execute during development/data preparation, not at engine runtime. They bridge Marathon's legacy Macintosh resource formats to the engine's binary data formats for efficient in-engine loading.

## Notable Implementation Details
- Uses Mac Toolbox APIs (OpenResFile, GetResource, HLock, ResError, c2pstr) with fallback to standard C file I/O
- Implements collection-based resource grouping with embedded offset metadata for direct binary access without index lookups
- Physics patches preserve parent WAD checksums to enable incremental patch validation
- Sound extractors support resource permutations for multi-variant sound playback and per-definition source fallback inheritance
- Resource memory handles locked and released atomically; file offsets computed and tracked for all extracted data
