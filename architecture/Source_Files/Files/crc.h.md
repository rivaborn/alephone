# Source_Files/Files/crc.h

## File Purpose
Header declaring CRC (Cyclic Redundancy Check) calculation functions for the Aleph One game engine. Provides 32-bit and 16-bit CRC computation over files and raw data buffers for integrity verification.

## Core Responsibilities
- Declare CRC-32 calculation for FileSpecifier-based files
- Declare CRC-32 calculation for OpenedFile objects
- Declare CRC-32 calculation for raw byte buffers
- Declare CRC-16 (CCITT variant) calculation for raw byte buffers
- Provide data integrity checking utilities

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | Forward-declared class | File abstraction (definition elsewhere) |
| OpenedFile | Forward-declared class | Open file handle abstraction (definition elsewhere) |

## Global / File-Static State
None.

## Key Functions / Methods

### calculate_crc_for_file
- Signature: `uint32 calculate_crc_for_file(FileSpecifier& File)`
- Purpose: Compute 32-bit CRC over an entire file specified by a FileSpecifier object
- Inputs: Reference to a FileSpecifier object
- Outputs/Return: uint32 CRC value
- Side effects: Reads file data (I/O)
- Calls: Not visible; implementation in .cpp file
- Notes: Takes file abstraction as parameter; likely opens and reads file internally

### calculate_crc_for_opened_file
- Signature: `uint32 calculate_crc_for_opened_file(OpenedFile& OFile)`
- Purpose: Compute 32-bit CRC over data in an already-opened file
- Inputs: Reference to an OpenedFile object
- Outputs/Return: uint32 CRC value
- Side effects: Reads from file position (I/O)
- Calls: Not visible; implementation in .cpp file
- Notes: Accepts pre-opened file; avoids redundant file operations

### calculate_data_crc
- Signature: `uint32 calculate_data_crc(unsigned char *buffer, int32 length)`
- Purpose: Compute 32-bit CRC over a raw byte buffer in memory
- Inputs: Pointer to byte buffer, length in bytes
- Outputs/Return: uint32 CRC value
- Side effects: None (read-only)
- Calls: Not visible; implementation in .cpp file
- Notes: Pure data operation; no I/O involved

### calculate_data_crc_ccitt
- Signature: `uint16 calculate_data_crc_ccitt(unsigned char *buffer, int32 length)`
- Purpose: Compute CCITT variant 16-bit CRC over a raw byte buffer
- Inputs: Pointer to byte buffer, length in bytes
- Outputs/Return: uint16 CRC value
- Side effects: None (read-only)
- Calls: Not visible; implementation in .cpp file
- Notes: Returns smaller 16-bit value; CCITT is a standard CRC polynomial variant

## Control Flow Notes
This is a utility library header for data integrity operations. CRC functions are typically invoked during save game validation, file transmission verification, or data corruption detection. Fits into a data validation/checksumming layer independent of main game loop.

## External Dependencies
- `cstypes.h` ΓÇö Standard type definitions (uint32, uint16, int32, unsigned char)
- `FileSpecifier` class ΓÇö defined elsewhere; game's file abstraction layer
- `OpenedFile` class ΓÇö defined elsewhere; game's file handle abstraction
