# Source_Files/RenderMain/textures.cpp

## File Purpose
Low-level bitmap memory management and pixel remapping for the Aleph One game engine. Handles initialization of bitmap row address tables, pixel data origin calculation, and color table remapping for both linear and RLE-compressed bitmap formats.

## Core Responsibilities
- Calculate bitmap pixel data origin based on structure layout and row addressing mode
- Precalculate row address lookup tables for quick per-row access
- Support both row-major and column-major bitmap layouts
- Handle RLE (Run-Length Encoded) compressed bitmap formats (MARATHON1 and MARATHON2 variants)
- Apply color remapping tables to bitmap pixels using lookup-table substitution
- Manage endianness for big-endian RLE metadata

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition` | struct | Defines bitmap metadata (width, height, row addresses, compression flags) |
| `bitmap_definition_buffer` | class (C++ wrapper) | RAII container managing dynamically-allocated bitmap definition + row array |

## Global / File-Static State
None.

## Key Functions / Methods

### calculate_bitmap_origin
- **Signature:** `pixel8 *calculate_bitmap_origin(struct bitmap_definition *bitmap)`
- **Purpose:** Compute the address where pixel data begins, accounting for the bitmap_definition structure and row address pointer array.
- **Inputs:** Bitmap definition struct with width/height/flags initialized.
- **Outputs/Return:** Pointer to the first byte of pixel data.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Skips past the bitmap_definition struct and either `width` or `height` worth of row pointers depending on the `_COLUMN_ORDER_BIT` flag.

### precalculate_bitmap_row_addresses
- **Signature:** `void precalculate_bitmap_row_addresses(struct bitmap_definition *bitmap)`
- **Purpose:** Initialize the bitmap's row_addresses array so each row can be accessed directly without scanning.
- **Inputs:** Bitmap with `bytes_per_row`, `height`, and `row_addresses[0]` already set.
- **Outputs/Return:** None (modifies bitmap in-place).
- **Side effects:** Writes to all entries in `bitmap->row_addresses[]`.
- **Calls:** `map_bytes()` indirectly (in MARATHON1/MARATHON2 RLE scanning).
- **Notes:** Branches on `bytes_per_row != NONE` to handle linear vs. RLE formats. MARATHON2 (active) interprets RLE metadata as two big-endian uint16 values per row (first/last pixel offset). Row count depends on `_COLUMN_ORDER_BIT`: if set, `width` is row count; otherwise `height`.

### remap_bitmap
- **Signature:** `void remap_bitmap(struct bitmap_definition *bitmap, pixel8 *table)`
- **Purpose:** Apply a color lookup table to every pixel in the bitmap, remapping colors for palette changes or effects.
- **Inputs:** Initialized bitmap and a 256-byte remapping table.
- **Outputs/Return:** None (modifies bitmap pixels in-place).
- **Side effects:** Overwrites all pixel values via `map_bytes()`.
- **Calls:** `map_bytes()`.
- **Notes:** Requires precalculate_bitmap_row_addresses() to have been called first. Handles both linear and RLE formats; for RLE, scans and remaps run-length data blocks.

### map_bytes
- **Signature:** `void map_bytes(byte *buffer, byte *table, int32 size)`
- **Purpose:** Simple lookup-table remap: replace each byte with `table[byte]`.
- **Inputs:** Byte buffer, 256-entry lookup table, size in bytes.
- **Outputs/Return:** None (modifies buffer in-place).
- **Side effects:** Overwrites buffer contents.
- **Calls:** None.
- **Notes:** Trivial loop; no bounds checking on table indices.

## Control Flow Notes
These functions are initialization/setup routines called before rendering. Typical order:
1. Load bitmap_definition and raw pixel data.
2. Call `precalculate_bitmap_row_addresses()` to cache row pointers.
3. Optionally call `remap_bitmap()` to apply color remapping.
4. Pass bitmap to renderer.

Not part of per-frame update loop; called infrequently when bitmaps load or palettes change.

## External Dependencies
- **Includes:** `cseries.h` (base types, endianness detection), `textures.h` (bitmap_definition).
- **External symbols:** `pixel8`, `byte`, `int32`, `int16`, `uint16` (defined in cseries/cstypes.h); `NONE` (likely a -1 constant).
- **Preprocessor:** `MARATHON1`/`MARATHON2` conditionals select RLE format variant.
