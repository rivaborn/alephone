# Source_Files/RenderMain/textures.h

## File Purpose
Defines bitmap and texture metadata structures and utility functions for the rendering pipeline. Provides bitmap definition buffers, pixel data mapping, and row-address calculation for texture processing in the Aleph One game engine.

## Core Responsibilities
- Define bitmap metadata structure (`bitmap_definition`) with pixel dimensions, row stride, and flags
- Provide RAII bitmap buffer class for safe allocation of bitmap definition + row pointer arrays
- Declare bitmap origin and row-address calculation functions
- Declare pixel data remapping functions for color/palette transformations
- Define bitmap flags for rendering hints (column order, transparency, patching)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition` | struct | Bitmap metadata: dimensions, bytes per row, flags, bit depth, row pointers |
| `bitmap_definition_buffer` | class | RAII wrapper managing allocation of bitmap_definition + row pointer array |
| (unnamed) | enum | Bitmap flags: `_COLUMN_ORDER_BIT`, `_TRANSPARENT_BIT`, `_PATCHED_BIT` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SIZEOF_bitmap_definition` | int | file-static const | Constant for manual struct size calculation (30 bytes) |

## Key Functions / Methods

### calculate_bitmap_origin
- Signature: `pixel8 *calculate_bitmap_origin(struct bitmap_definition *bitmap)`
- Purpose: Calculate the memory address where pixel data begins, assuming it follows the bitmap_definition structure immediately in memory
- Inputs: Pointer to bitmap_definition
- Outputs/Return: Pointer to first pixel byte
- Side effects: None
- Calls: (defined elsewhere)
- Notes: Requires pixel data to be laid out contiguously after bitmap_definition

### precalculate_bitmap_row_addresses
- Signature: `void precalculate_bitmap_row_addresses(struct bitmap_definition *texture)`
- Purpose: Populate the row pointer array (`row_addresses`) based on height and bytes_per_row for fast row lookup
- Inputs: Initialized bitmap_definition with bytes_per_row and height set; row_addresses[0] pointing to pixel data
- Outputs/Return: None (modifies in-place)
- Side effects: Modifies bitmap_definition.row_addresses array
- Calls: (defined elsewhere)
- Notes: Must be called during bitmap initialization before rendering

### map_bytes
- Signature: `void map_bytes(byte *buffer, byte *table, int32 size)`
- Purpose: Apply a lookup table transformation to a block of bytes (e.g., palette remapping)
- Inputs: buffer (data to transform), table (lookup values), size (byte count)
- Outputs/Return: None (modifies buffer in-place)
- Side effects: Overwrites input buffer
- Calls: (defined elsewhere)

### remap_bitmap
- Signature: `void remap_bitmap(struct bitmap_definition *bitmap, pixel8 *table)`
- Purpose: Remap all pixels in a bitmap using a color/palette lookup table
- Inputs: bitmap (texture to modify), table (color mapping)
- Outputs/Return: None (modifies bitmap in-place)
- Side effects: Modifies bitmap pixel data
- Calls: (defined elsewhere, likely calls map_bytes)

## Control Flow Notes
This is a **texture initialization/loading** module. Functions are called during map/texture load time, not per-frame. The bitmap_definition_buffer provides safe allocation for dynamic row counts. Bitmap flags indicate rendering properties (RLE transparency, column-major layout) used by downstream renderers.

## External Dependencies
- `cseries.h` ΓÇö core platform abstractions, type definitions (pixel8, byte, int16, uint16)
- `<vector>` ΓÇö standard library for dynamic byte buffer
- Defined elsewhere: `pixel8` (typedef for uint8), bitmap processing implementations in TEXTURES.C
