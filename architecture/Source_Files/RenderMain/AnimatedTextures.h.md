# Source_Files/RenderMain/AnimatedTextures.h

## File Purpose
Header providing the public interface for animated texture handling in the Aleph One engine. Defines functions to update animated textures each frame, translate texture descriptors to account for animation state, and configure animated textures via XML.

## Core Responsibilities
- Update animated texture state each frame
- Translate shape descriptors to map base textures to their current animated frame
- Parse MML (Marathon Markup Language) XML configuration for animated textures
- Reset animated texture configuration to defaults

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef (uint16) | Encodes collection, shape, and CLUT indices; used to identify textures for animation translation |
| `InfoTree` | class (forward decl.) | XML/MML parse tree; configuration source for animated textures |

## Global / File-Static State
None.

## Key Functions / Methods

### AnimTxtr_Update
- Signature: `void AnimTxtr_Update()`
- Purpose: Advance animated texture state each frame
- Inputs: None (operates on global/static state)
- Outputs/Return: None (void)
- Side effects: Updates internal animation frame counters or state
- Calls: Not inferable from this file
- Notes: Likely called once per render frame to update all animated texture frames

### AnimTxtr_Translate
- Signature: `shape_descriptor AnimTxtr_Translate(shape_descriptor Texture)`
- Purpose: Map a base texture descriptor to the current animated frame variant
- Inputs: `Texture` ΓÇô original shape descriptor (collection.5 | shape.8 | clut.3 bits)
- Outputs/Return: Translated shape descriptor pointing to the current animation frame
- Side effects: None (pure lookup/translation)
- Calls: Not inferable from this file
- Notes: Returns original descriptor unchanged if texture is not animated; comment indicates shape_descriptor is a uint16

### parse_mml_animated_textures
- Signature: `void parse_mml_animated_textures(const InfoTree& root)`
- Purpose: Parse XML configuration tree to define which textures are animated and their parameters
- Inputs: `root` ΓÇô InfoTree parsed from MML/XML
- Outputs/Return: None (applies configuration globally)
- Side effects: Modifies global animated texture definitions
- Calls: Not inferable from this file

### reset_mml_animated_textures
- Signature: `void reset_mml_animated_textures()`
- Purpose: Reset animated texture configuration to defaults (clear all custom definitions)
- Inputs: None
- Outputs/Return: None
- Side effects: Clears animated texture state/definitions
- Calls: Not inferable from this file

## Control Flow Notes
Fits into the render pipeline:
- `parse_mml_animated_textures()` called during level/configuration load
- `AnimTxtr_Update()` called once per frame (likely in main render loop)
- `AnimTxtr_Translate()` called during texture lookup (likely in rendering/texture-binding code)
- `reset_mml_animated_textures()` called during shutdown or configuration reset

## External Dependencies
- **Includes:** `shape_descriptors.h` (defines shape_descriptor typedef and bit-field macros)
- **Forward declarations:** `InfoTree` (defined elsewhere; XML/MML parse tree)
- **Defined elsewhere:** Implementation of all four functions; actual animated texture state/tables
