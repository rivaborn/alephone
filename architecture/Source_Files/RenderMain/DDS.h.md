# Source_Files/RenderMain/DDS.h

## File Purpose
Defines DirectDraw Surface (DDS) file format structures and flag constants for texture loading. Implements the DDS file format specification from DirectX 9, allowing the engine to parse and read DDS-formatted texture files. Includes conditional guards to avoid conflicts with system DDRAW headers.

## Core Responsibilities
- Define DDS surface descriptor structure (DDSURFACEDESC2) for file header parsing
- Define DDS surface capability flags (DDSCAPS, DDSCAPS2)
- Define pixel format flags (DDPF_*) for texture color/alpha interpretation
- Define surface description flags (DDSD_*) to indicate which fields are valid
- Provide header-only DDS format support without system DDRAW dependency

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| DDSURFACEDESC2 | struct | Main DDS file surface descriptor; contains image dimensions, pitch/linear size, mipmap count, pixel format info, and capability flags. Used to parse DDS file headers. |

## Global / File-Static State
None.

## Key Functions / Methods
None ΓÇö this is a data definition header.

## Control Flow Notes
Not applicable. This is a pure data definition file included by texture loaders and DDS parsers during render initialization or resource loading phases.

## External Dependencies
- `cstypes.h` ΓÇö provides `uint32` typedef via SDL2 types
- Conditional: `#ifndef __DDRAW_INCLUDED__` ΓÇö guards against redefinition if system DirectDraw headers are already loaded
