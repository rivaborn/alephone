# Source_Files/RenderMain/shape_descriptors.h

## File Purpose
Defines the 16-bit packed `shape_descriptor` type and related macros for encoding/decoding graphical asset references in the Aleph One game engine. Enumerates all sprite collections (enemies, items, scenery, environment) used throughout the game.

## Core Responsibilities
- Define the `shape_descriptor` typedef as a packed 16-bit value (8-bit shape + 5-bit collection + 3-bit CLUT)
- Provide extraction and construction macros for descriptor components
- Enumerate 32 game asset collections (enemies, weapons, items, scenery, landscapes, etc.)
- Define bit widths and maximum values for each descriptor field
- Support color lookup table (CLUT) encoding alongside shape/collection data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef | 16-bit packed encoding: [CLUT:3][collection:5][shape:8] |

## Global / File-Static State
None.

## Key Functions / Methods

### GET_DESCRIPTOR_SHAPE(d)
- Signature: `#define GET_DESCRIPTOR_SHAPE(d) ((d)&(uint16)(MAXIMUM_SHAPES_PER_COLLECTION-1))`
- Purpose: Extract shape index from descriptor
- Inputs: `d` ΓÇô shape_descriptor
- Outputs/Return: Lower 8 bits (shape 0ΓÇô255)
- Side effects: None
- Calls: None
- Notes: Masks lower 8 bits only

### GET_DESCRIPTOR_COLLECTION(d)
- Signature: `#define GET_DESCRIPTOR_COLLECTION(d) (((d)>>DESCRIPTOR_SHAPE_BITS)&(uint16)((1<<(DESCRIPTOR_COLLECTION_BITS+DESCRIPTOR_CLUT_BITS))-1))`
- Purpose: Extract collection and CLUT combined
- Inputs: `d` ΓÇô shape_descriptor
- Outputs/Return: Middle 8 bits (collection + CLUT)
- Side effects: None
- Calls: None
- Notes: Shifts right 8, masks 8 bits (5 collection + 3 CLUT)

### BUILD_DESCRIPTOR(collection, shape)
- Signature: `#define BUILD_DESCRIPTOR(collection,shape) (((collection)<<DESCRIPTOR_SHAPE_BITS)|(shape))`
- Purpose: Pack collection and shape into a descriptor
- Inputs: `collection` ΓÇô 0ΓÇô31; `shape` ΓÇô 0ΓÇô255
- Outputs/Return: Packed shape_descriptor
- Side effects: None
- Calls: None
- Notes: Shifts collection left 8 bits, ORs with shape

### BUILD_COLLECTION(collection, clut)
- Signature: `#define BUILD_COLLECTION(collection,clut) ((collection)|(uint16)((clut)<<DESCRIPTOR_COLLECTION_BITS))`
- Purpose: Encode CLUT into collection value
- Inputs: `collection` ΓÇô 0ΓÇô31; `clut` ΓÇô 0ΓÇô7
- Outputs/Return: Encoded collection value
- Side effects: None
- Calls: None
- Notes: Shifts CLUT left 5 bits, ORs with collection

### GET_COLLECTION_CLUT(collection) / GET_COLLECTION(collection)
- Trivial extraction macros for CLUT (upper 3 bits) and collection (lower 5 bits) from an encoded collection value.

## Control Flow Notes
This is a pure definitions header with no runtime control flow. It is included by rendering and sprite systems to interpret packed shape descriptors when fetching graphical assets from collections.

## External Dependencies
- `#include "cstypes.h"` ΓÇô provides `uint16` type
