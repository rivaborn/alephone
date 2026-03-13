# Source_Files/Files/crc.h - Enhanced Analysis

## Architectural Role

CRC calculation is a data integrity utility serving the **Files** and **Network** subsystems in Aleph One. It provides three entry points to compute 32-bit and 16-bit checksums over different data sources: files (via abstraction), open file handles, and raw buffers. This positions CRC as a **validation layer** used during game state persistence (save game corruption detection), multiplayer map synchronization, and WAD archive integrity checkingΓÇöcritical for deterministic gameplay and networked consistency.

## Key Cross-References

### Incoming (who depends on this file)
- **Files subsystem** (`game_wad.cpp`, `wad.cpp`): Calls CRC functions to validate saved games and WAD archive integrity
- **Network subsystem** (`network_games.cpp`, likely `network_messages.cpp`): Computes map checksums for peer synchronization during multiplayer setup
- **Metaserver communication**: Possibly validates map packages before distribution
- **Shell/Interface** (through Files): Indirect dependency during save game load validation

### Outgoing (what this file depends on)
- **cstypes.h**: Type definitions (`uint32`, `uint16`, `int32`, `unsigned char`)
- **FileSpecifier** class (`Source_Files/Files/FileHandler.h` or similar): Abstract file referenceΓÇöCRC never touches implementation details
- **OpenedFile** class (`Source_Files/Files/FileHandler.h` or similar): Abstract file handleΓÇöallows CRC computation on partially-read files or streaming scenarios
- No direct dependency on endianness handling (CRC algorithms are byte-order agnostic, unlike binary serialization in AStream)

## Design Patterns & Rationale

**Three-Level Abstraction**
- `calculate_crc_for_file(FileSpecifier&)`: File path ΓåÆ full CRC (opens, reads, closes internally)
- `calculate_crc_for_opened_file(OpenedFile&)`: Active file handle ΓåÆ CRC (avoids redundant open/close; supports streaming)
- `calculate_data_crc(buffer, length)`: In-memory data ΓåÆ CRC (zero I/O overhead; used for network messages, save validation)

This stratification reflects the engine's separation-of-concerns philosophy: CRC logic is orthogonal to file I/O, so callers choose the most efficient path. The file-handle variant enables incremental checksumming (e.g., partial file reads during network transmission).

**Dual CRC Variants**
- **CRC-32**: Detects 99.99% of bit errors over files up to 4GB (industry standard for integrity)
- **CRC-16 (CCITT)**: Smaller footprint for network protocol fields; suggests CCITT polynomial is mandated by existing multiplayer protocol spec (Marathon or Aleph One tradition)

This dual approach implies **protocol versioning conservatism**ΓÇöCRC-16 was likely established in early Marathon networking; CRC-32 was added later for critical save game validation without breaking existing network messages.

## Data Flow Through This File

```
Game Save File ΓåÆ calculate_crc_for_file ΓåÆ CRC-32 ΓåÆ game_wad.cpp ΓåÆ validation check
                                                   Γåô corrupted? ΓåÆ alert_user("Save game invalid")

Network Map File ΓåÆ calculate_crc_for_opened_file ΓåÆ CRC-32 ΓåÆ network_games.cpp ΓåÆ peer comparison
                                                         Γåô mismatch? ΓåÆ resync or abort join

Player Action Buffer ΓåÆ calculate_data_crc ΓåÆ CRC-32 ΓåÆ network_messages ΓåÆ append to packet

Serialized Entity Data ΓåÆ calculate_data_crc_ccitt ΓåÆ CRC-16 ΓåÆ protocol header field
```

The header file serves as the **interface contract** for these callers; the implementation (`crc.cpp`) contains the polynomial tables and update loops, hidden from this header.

## Learning Notes

**Era & Style**
- This is authentic late-1990s/early-2000s systems code: raw pointers, no exceptions, no bounds checking
- Forward declarations of `FileSpecifier` and `OpenedFile` show intentional **abstraction** rather than C-style global functionsΓÇöthe engine's architecture does separate concerns

**Idiomatic Patterns**
- The three-level API mirrors the engine's **file abstraction hierarchy** (FileSpecifier ΓåÆ OpenedFile ΓåÆ raw bytes)
- Absence of status codes or error handling (return type is just `uint32`) reflects an era before RAII; callers must validate file opening separately

**Modern Contrast**
- A modern engine would likely:
  - Use generic `std::vector<uint8_t>` or `std::span<const uint8_t>` instead of `unsigned char*`
  - Return `std::optional<uint32_t>` with error codes
  - Implement CRC as a stateful incremental object (e.g., `CRCAccumulator` that can feed chunks and finalize)
  - Use `constexpr` tables for compile-time initialization

## Potential Issues

1. **Pointer Safety**: `unsigned char *buffer` lacks boundsΓÇöcaller must ensure `length` is valid. In modern C++, would use `std::span<const std::uint8_t>`.

2. **Signedness Mismatch**: `int32 length` is signed; could theoretically be passed negative (logic error). Should be `uint32 length` to match return type semantics.

3. **Silent Failure Path**: If `calculate_crc_for_file()` encounters I/O errors (disk read failure, permission denied), the header doesn't indicate how errors are communicated. Likely returns 0 or garbage; callers may not detect corruption vs. missing file.

4. **No Incremental CRC**: CCITT variant is used for streamed data (network), but there's no stateful API to accumulate CRC over multiple buffers without concatenating themΓÇöforces copies or requires calling implementation directly.
