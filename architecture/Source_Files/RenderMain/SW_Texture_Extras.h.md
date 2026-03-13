# Source_Files/RenderMain/SW_Texture_Extras.h

## File Purpose
Manages software-rendered textures with opacity and color lookup information. Provides a singleton interface for loading, unloading, and accessing textures by shape descriptor, and supports MML-based texture configuration parsing.

## Core Responsibilities
- Wrap shape descriptors with opacity type, scale, shift, and opacity lookup tables
- Maintain a per-collection registry of software textures via singleton pattern
- Load and unload texture collections, supporting dynamic resource management
- Build opacity tables for texture rendering calculations
- Parse and reset MML (Marathon Markup Language) texture configurations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SW_Texture` | class | Wraps a shape descriptor with opacity metadata and a lookup table for per-pixel opacity values |
| `SW_Texture_Extras` | class | Singleton managing all software textures, indexed by collection (32 collections max) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` (in `instance()`) | `static SW_Texture_Extras*` | static in method | Singleton pointer; lazy-initialized on first call |

## Key Functions / Methods

### SW_Texture::descriptor
- **Signature:** `void descriptor(shape_descriptor ShapeDesc)`
- **Purpose:** Set the shape descriptor for this texture.
- **Inputs:** `ShapeDesc` ΓÇô a 16-bit value encoding collection, shape, and CLUT indices.
- **Outputs/Return:** None.
- **Side effects:** Modifies `m_shape_descriptor`.
- **Calls:** None.
- **Notes:** No validation; descriptor format is caller's responsibility.

### SW_Texture::opac_type
- **Signature:** `int opac_type()` / `void opac_type(int new_opac_type)`
- **Purpose:** Get or set the opacity calculation type (e.g., method for blending).
- **Inputs:** `new_opac_type` (setter only).
- **Outputs/Return:** Current opac type (getter).
- **Side effects:** Modifies `m_opac_type`.
- **Calls:** None.

### SW_Texture::opac_table
- **Signature:** `uint8 *opac_table()`
- **Purpose:** Retrieve pointer to the opacity lookup table, or null if empty.
- **Inputs:** None.
- **Outputs/Return:** Pointer to first element of `m_opac_table`, or null.
- **Side effects:** None.
- **Calls:** `std::vector<uint8>::front()`, `std::vector<uint8>::size()`.
- **Notes:** Returns null if table has not been built; caller must check before dereferencing.

### SW_Texture::build_opac_table
- **Signature:** `void build_opac_table()`
- **Purpose:** Populate the opacity lookup table based on shape descriptor and opacity parameters.
- **Inputs:** Implicitly uses `m_shape_descriptor`, `m_opac_type`, `m_opac_scale`, `m_opac_shift`.
- **Outputs/Return:** None (populates `m_opac_table`).
- **Side effects:** Allocates memory in `m_opac_table`; may perform I/O or resource lookup.
- **Calls:** Not visible in header; defined elsewhere.
- **Notes:** Must be called explicitly; not automatic on construction or property changes.

### SW_Texture_Extras::instance
- **Signature:** `static SW_Texture_Extras *instance()`
- **Purpose:** Retrieve the singleton instance, creating it if necessary.
- **Inputs:** None.
- **Outputs/Return:** Pointer to the static `SW_Texture_Extras` instance.
- **Side effects:** Heap allocation on first call (not freed).
- **Calls:** Implicit constructor call if `m_instance` is null.
- **Notes:** Classic singleton pattern; thread safety not evident.

### SW_Texture_Extras::GetTexture
- **Signature:** `SW_Texture *GetTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Retrieve an existing texture by shape descriptor, or null if not found.
- **Inputs:** `ShapeDesc` ΓÇô 16-bit shape descriptor.
- **Outputs/Return:** Pointer to matching `SW_Texture`, or null.
- **Side effects:** None (read-only).
- **Calls:** Not visible in header.

### SW_Texture_Extras::AddTexture
- **Signature:** `SW_Texture *AddTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Create and register a new texture for the given descriptor.
- **Inputs:** `ShapeDesc` ΓÇô 16-bit shape descriptor.
- **Outputs/Return:** Pointer to the newly created `SW_Texture`.
- **Side effects:** Appends to `texture_list[collection]` vector.
- **Calls:** Not visible in header.

### SW_Texture_Extras::Load / Unload / Reset
- **Signature:** `void Load(short Collection)`, `void Unload(short Collection)`, `void Reset()`
- **Purpose:** Load/unload textures for a specific collection, or reset all textures.
- **Inputs:** `Collection` index (0ΓÇô31).
- **Outputs/Return:** None.
- **Side effects:** Populate/clear `texture_list` entries; may perform I/O (MML parsing, resource loading).
- **Calls:** Not visible in header.

### parse_mml_software / reset_mml_software
- **Signature:** `void parse_mml_software(const InfoTree& root)`, `void reset_mml_software()`
- **Purpose:** Parse MML configuration into texture properties, or reset configuration to defaults.
- **Inputs:** `root` ΓÇô an InfoTree (forward-declared; type defined elsewhere).
- **Outputs/Return:** None (side effects on global state or singleton).
- **Side effects:** Modifies textures via the singleton.
- **Calls:** Not visible in header.

## Control Flow Notes
This module is part of **resource initialization / setup**, not the per-frame rendering loop. Textures are loaded once (or on demand) via `Load()` and referenced during software rendering. The MML parsing (`parse_mml_software`) is likely called during engine or level initialization from a configuration file.

## External Dependencies
- **cseries.h** ΓÇô general engine utility macros and type definitions
- **cstypes.h** ΓÇô fixed-width integer types (`uint8`, `int`, `float`)
- **shape_descriptors.h** ΓÇô `shape_descriptor` typedef, `NUMBER_OF_COLLECTIONS` macro, collection enumeration
- **\<vector\>** ΓÇô C++ standard `std::vector` container
- **InfoTree** ΓÇô forward-declared; defined elsewhere (likely a markup/config tree class)
