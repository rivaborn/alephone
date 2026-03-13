# Source_Files/RenderMain/SW_Texture_Extras.cpp

## File Purpose
Implements software renderer texture management, specifically handling opacity table construction and configuration parsing for textures. Manages collections of textures indexed by collection and shape descriptors, with support for loading/unloading and MML-based configuration.

## Core Responsibilities
- Build opacity tables for textures from shading table data (supporting 16- and 32-bit color modes)
- Manage texture collections storage and lookup via shape descriptors
- Support three opacity calculation modes (full opacity, average RGB, or maximum channel)
- Parse texture configuration from MML (Marathon Markup Language) XML
- Load and unload opacity tables for entire collections on demand
- Singleton pattern for centralized texture extras management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SW_Texture` | class | Individual texture with opacity properties and opacity table storage |
| `SW_Texture_Extras` | class | Singleton managing a 2D array of texture collections (indexed by collection + bitmap index) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `bit_depth` | `short` (extern) | global | Current color depth (16 or 32 bits); determines opacity table construction path |

## Key Functions / Methods

### SW_Texture::build_opac_table()
- **Signature:** `void build_opac_table()`
- **Purpose:** Constructs opacity table for this texture from its shading tables. Opacity values are derived based on `m_opac_type` (full opacity, luminance average, or max channel). Table indices are clamped and scaled via `m_opac_scale` and `m_opac_shift`.
- **Inputs:** None (uses member state: `m_shape_descriptor`, `m_opac_type`, `m_opac_scale`, `m_opac_shift`)
- **Outputs/Return:** Populates `m_opac_table` vector with uint8 opacity values
- **Side effects:** Resizes and modifies `m_opac_table`; acquires bitmap/shading data from global state
- **Calls:** `get_shape_bitmap_and_shading_table()` (defined elsewhere), `MainScreenSurface()`, `PIN()` macro (clamping)
- **Notes:** Handles both 16-bit and 32-bit SDL pixel formats independently; extracts color channels using SDL format masks; applies clamping via `PIN` macro to ensure [0,255] range

### SW_Texture_Extras::AddTexture(shape_descriptor ShapeDesc)
- **Signature:** `SW_Texture *AddTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Retrieves or creates a texture entry at the specified collection/shape index, resizing the collection vector if needed.
- **Inputs:** `ShapeDesc` ΓÇö shape descriptor encoding collection and shape indices
- **Outputs/Return:** Pointer to `SW_Texture` at that index
- **Side effects:** May resize `texture_list[Collection]` vector
- **Calls:** `GET_COLLECTION()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_DESCRIPTOR_SHAPE()` macros
- **Notes:** Returns reference to newly created or existing texture; does not initialize texture properties

### SW_Texture_Extras::GetTexture(shape_descriptor ShapeDesc)
- **Signature:** `SW_Texture *GetTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Safe lookup of texture by shape descriptor; returns null if index out of bounds.
- **Inputs:** `ShapeDesc` ΓÇö shape descriptor
- **Outputs/Return:** Pointer to texture, or null if bitmap index >= collection size
- **Side effects:** None
- **Calls:** Descriptor extraction macros
- **Notes:** Bounds-checking; read-only

### SW_Texture_Extras::Load(short collection_index)
- **Signature:** `void Load(short collection_index)`
- **Purpose:** Builds opacity tables for all textures in a collection (typically called when loading textures into memory).
- **Inputs:** `collection_index` ΓÇö collection ID
- **Outputs/Return:** None
- **Side effects:** Calls `build_opac_table()` on each texture in the collection
- **Calls:** `SW_Texture::build_opac_table()`
- **Notes:** Iterates via vector iterator; no-op if collection empty

### SW_Texture_Extras::Unload(short collection_index)
- **Signature:** `void Unload(short collection_index)`
- **Purpose:** Clears opacity tables for all textures in a collection (freeing memory).
- **Inputs:** `collection_index` ΓÇö collection ID
- **Outputs/Return:** None
- **Side effects:** Clears `m_opac_table` on each texture via `clear_opac_table()`
- **Calls:** `SW_Texture::clear_opac_table()`
- **Notes:** Inverse of `Load()`

### SW_Texture_Extras::Reset()
- **Signature:** `void Reset()`
- **Purpose:** Clears all texture collections (typically on engine reset or shutdown).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears all `texture_list[i]` vectors
- **Calls:** None
- **Notes:** Iterates over `NUMBER_OF_COLLECTIONS` constant

### parse_mml_software(const InfoTree& root)
- **Signature:** `void parse_mml_software(const InfoTree& root)`
- **Purpose:** Parses XML configuration from MML root, creating and configuring textures. Reads `<texture>` child elements with `coll`, `bitmap`, `opac_type`, `opac_scale`, `opac_shift` attributes.
- **Inputs:** `root` ΓÇö InfoTree XML node
- **Outputs/Return:** None
- **Side effects:** Creates/modifies texture entries in singleton instance; modifies `SW_Texture_Extras` state
- **Calls:** `root.children_named()`, `InfoTree::read_indexed()`, `InfoTree::read_attr()`, `SW_Texture_Extras::instance()->AddTexture()`, texture setter methods
- **Notes:** Skips entries with invalid collection or bitmap indices; uses `BUILD_DESCRIPTOR()` macro to encode collection+shape

### reset_mml_software()
- **Signature:** `void reset_mml_software()`
- **Purpose:** Resets all software renderer textures to initial state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `SW_Texture_Extras::instance()->Reset()`
- **Calls:** Singleton accessor and `Reset()`
- **Notes:** Trivial wrapper function

## Control Flow Notes
**Initialization/Configuration:** MML parsing (`parse_mml_software()`) occurs at startup to populate textures from external configuration. **Per-Level Load:** `Load()` is called when a collection is brought into memory to build opacity tables. **Frame Update:** During rendering, textures are accessed read-only via `GetTexture()`. **Cleanup:** `Unload()` frees opacity memory; `reset_mml_software()` clears all state on engine shutdown or level change.

## External Dependencies
- **`interface.h`:** `get_shape_bitmap_and_shading_table()` macro ΓÇö retrieves bitmap and shading data by shape descriptor
- **`screen.h`:** `MainScreenSurface()` ΓÇö returns SDL surface with pixel format info
- **`scottish_textures.h`:** `MAXIMUM_SHADING_TABLE_INDEXES`, `bitmap_definition`, color/pixel constants
- **`collection_definition.h`:** Collection structure and constants (`NUMBER_OF_COLLECTIONS`)
- **`InfoTree.h`:** XML parsing (`InfoTree`, `children_named()`, `read_indexed()`, `read_attr()`)
- **SDL2:** `SDL_PixelFormat` for color format masks and bit shifts
- **Global:** `bit_depth` (extern), `number_of_shading_tables` (extern, used in `build_opac_table()`)
