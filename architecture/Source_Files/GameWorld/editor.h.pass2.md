# Source_Files/GameWorld/editor.h - Enhanced Analysis

## Architectural Role

This file establishes the **version contract for serialized map data** across the three Marathon engine generations, enabling the Files and GameWorld subsystems to negotiate compatibility when loading/saving WAD archives and synchronized multiplayer maps. By defining version constants (0ΓåÆ1ΓåÆ2), it creates a compact versioning scheme that `game_wad.cpp` references during serialization/deserialization to handle format evolution gracefully. The current `EDITOR_MAP_VERSION` pinning to `MARATHON_INFINITY_DATA_VERSION` signals that Aleph One treats Infinity format as the canonical target for new content, though it can still read legacy formats.

## Key Cross-References

### Incoming (who depends on this file)

- **Files/game_wad.cpp** ΓÇö Map loading/saving; version checking during WAD deserialization to select correct parsing path for player/monster/projectile/platform state
- **Network/network_messages.cpp** ΓÇö Map synchronization; version constants ensure client and server agree on serialized entity format before applying network deltas
- **GameWorld/placement.cpp** ΓÇö Object instantiation from versioned map data; may branch on `EDITOR_MAP_VERSION` to handle format-specific placement quirks
- **Shell/Interface startup** ΓÇö Level loading pipeline; version check gates compatibility warnings ("This map requires Infinity engine")

### Outgoing (what this file depends on)

- None ΓÇö pure constant definitions; no includes or external dependencies

## Design Patterns & Rationale

**Minimal Versioning Scheme**: Using small integers (0, 1, 2) rather than epoch timestamps or semantic version tuples is characteristic of 1990s engine design. Pros: compact binary serialization (1 byte vs. 3ΓÇô4), simple comparison (`if (version >= MARATHON_TWO_DATA_VERSION)`). Cons: no granularity between patch releases; once Infinity format becomes obsolete, adding a v3 requires code churn.

**Linear Monotonic Versioning**: Each version strictly supersedes the previous; no branching or parallel formats. This simplifies loading logic ("if v1, apply v1 rules; else if v2, ...") but inflexible if Marathon 2 maps have non-overlapping features vs. Infinity.

**Alias to Latest**: `EDITOR_MAP_VERSION` aliasing to `MARATHON_INFINITY_DATA_VERSION` encodes the **design intent** that new maps default to Infinity format, while legacy loaders can still opt-in via explicit constant reference. Found comment (Feb 2000) confirms this was a deliberate modernization step during Aleph One's early development.

## Data Flow Through This File

No dynamic data flowΓÇöthese are compile-time constants. Usage pattern:

1. **Map Load Path**: WAD reader peeks at version field ΓåÆ matches against `MARATHON_ONE/TWO/INFINITY_DATA_VERSION` ΓåÆ selects parser
2. **Map Save Path**: Editor/network code writes version field = `EDITOR_MAP_VERSION` ΓåÆ downstream readers interpret entities accordingly
3. **Network Sync**: Client compares local map version to server's `EDITOR_MAP_VERSION` embedded in join acceptance; version mismatch ΓåÆ reject join or emit compatibility warning

## Learning Notes

**Versioning in Legacy Game Engines**: This file exemplifies how pre-"semantic versioning" engines achieved forward compatibility. The scheme is deterministic and self-documenting in saved filesΓÇöyou always know a map's origin (v0ΓåÆMarathon 1, v1ΓåÆMarathon 2, v2ΓåÆInfinity). Modern engines (Unreal, Unity) use more granular versioning with per-field migration routines; here, the entire entity format shifts per version.

**Historical Context**: The Feb 2000 comment shows Aleph One (an open-source Marathon port) retrofitting Infinity support ~6 years after Infinity's 1996 release. The constants preserve that lineageΓÇönew contributors reading the code immediately learn Infinity is the "latest."

**Multiplayer Compatibility**: In networked games, version constants become **network protocol anchors**ΓÇömismatched versions break sync because entity state encoding differs. This explains why `game_wad.cpp` and `network_messages.cpp` both reference these constants; they're part of the serialization contract, not just editor metadata.

## Potential Issues

- **No Sub-Version Granularity**: Bug fixes or feature additions to Infinity format cannot be versioned sub-v2 without bumping to v3 and maintaining dual parsers. Any hidden assumptions in parsing (e.g., "Infinity assumes platform state bit layout X") are unversioned.
- **Hardcoded Current Version**: If format evolution continues, `EDITOR_MAP_VERSION = MARATHON_INFINITY_DATA_VERSION` locks new maps to v2 permanently. A future v3 requires grep-and-replace across save/load paths.
- **No Version Discovery**: The constants exist but there's no reverse mapping (e.g., function to convert version int to human-readable string for error messages). Users see "version 2" without contextΓÇöis that old or new?
