# Source_Files/RenderMain/collection_definition.h

## File Purpose
Defines binary-compatible data structures for Marathon/Aleph One game asset collections. Collections package shapes, animations, bitmaps, and color palettes into discrete files categorized by type (wall, object, interface, scenery).

## Core Responsibilities
- Define `collection_definition` as the root container for all assets in a collection file
- Define shape animation metadata (`high_level_shape_definition`) with frame timing and sound cues
- Define individual sprite properties (`low_level_shape_definition`) including mirroring, origin/key points, and lighting
- Define color palette entry format (`rgb_color_value`) with luminescence flag
- Provide collection type enumeration and file format versioning constants
- Enforce binary layout compatibility with hardcoded struct sizes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `collection_definition` | struct | Root container; holds counts/offsets for high/low shapes, bitmaps, color tables; stores actual asset vectors |
| `high_level_shape_definition` | struct | Animation data: frames per view, ticks per frame, transfer modes, sound indices, and variable-length low-level shape index array |
| `low_level_shape_definition` | struct | Individual sprite properties: mirror flags, lighting, bitmap index, origin/key point coords, world bounds |
| `rgb_color_value` | struct | Palette entry: flags (luminescence), value, and RGB components |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `COLLECTION_VERSION` | macro (int) | global | Currently 3; version stamp for collection binary format |
| `NUMBER_OF_PRIVATE_COLORS` | macro (int) | global | 3; reserved color entries at start of palette used by asset extractor |
| Collection type enum | enum | global | `_unused_collection`, `_wall_collection` (raw), `_object_collection` (RLE), `_interface_collection` (raw), `_scenery_collection` (RLE) |
| `_X_MIRRORED_BIT`, `_Y_MIRRORED_BIT`, `_KEYPOINT_OBSCURED_BIT` | macros (uint16) | global | Flag bits in `low_level_shape_definition.flags` |
| `SELF_LUMINESCENT_COLOR_FLAG` | macro (uint8) | global | 0x80; flag in `rgb_color_value.flags` for self-lit colors |
| `SIZEOF_*` constants | int | global | Hardcoded binary struct sizes (544, 90, 36, 8 bytes); enforce format compatibility |

## Key Functions / Methods
NoneΓÇöthis is a pure data definition header.

## Control Flow Notes
Not applicable; this file defines only static data structures used during asset loading and sprite rendering. The `collection_definition` struct serves as a file format root: offsets and counts direct access to shape/bitmap/color data stored in binary collection files. The hardcoded `SIZEOF_*` constants enforce binary-compatible layouts.

## External Dependencies
- `cstypes.h`: provides fixed-width types (`int16`, `uint16`, `int32`, `uint32`, `uint8`, `_fixed`), `FIXED_ONE`, and fixed-point macros
- `<vector>`: STL containers (`std::vector`) used in `collection_definition` for runtime storage of parsed assets
- Forward declarations: `bitmap_definition` (not defined here)

## Notes
- `low_level_shape_definition` has a variable-length behavior: the single-element array `low_level_shape_indexes[1]` is actually overflowed in memory during parsing to hold multiple indices
- Transfer mode and transfer mode period allow runtime sprite blending/effect changes
- Sounds are indexed per animation phase: first frame, key frame (pivot), last frame
- `pixels_to_world` scale factor appears in both collection and high-level shape, allowing per-collection and per-shape override
- Collection types hint at compression: `_object_collection` and `_scenery_collection` use RLE; walls and interface use raw format
