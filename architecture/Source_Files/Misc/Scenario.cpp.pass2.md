# Source_Files/Misc/Scenario.cpp - Enhanced Analysis

## Architectural Role

This file implements the **scenario metadata and version-negotiation subsystem** for Aleph One, serving as the authoritative registry for the currently-loaded game scenario's identity and compatibility profile. It operates as a configuration integration point between the **MML/XML parser** (Source_Files/XML/) and runtime systems that need to validate scenario versions or enforce gameplay variants. The singleton pattern ensures exactly one active scenario exists throughout the engine lifetime.

## Key Cross-References

### Incoming (who depends on this file)
- **XML/MML parser** (`Source_Files/XML/XML_MakeRoot.cpp`) ΓåÆ calls `parse_mml_scenario()` to populate metadata from `.mml` config files
- **Network layer** ΓåÆ likely calls `IsCompatible()` to validate peer scenario versions during multiplayer handshakes (typical Marathon 2 protocol)
- **Gameplay systems** ΓåÆ call `AllowsClassicGameplay()` to conditionally apply Marathon 1 vs. Marathon 2 rules (e.g., collision, physics, item behavior)
- **Save/load subsystem** ΓåÆ references scenario ID for version negotiation and compatibility checking
- **Multiple subsystems** ΓåÆ access `Scenario::instance()` as a global configuration anchor

### Outgoing (what this file depends on)
- **InfoTree** (Source_Files/XML/InfoTree.h) ΓåÆ `read_attr()`, `children_named()` for attribute/child iteration
- **CSeries** (cseries.h) ΓåÆ `std::string`, `std::vector` STL containers, platform utilities
- **Scenario.h** (not shown) ΓåÆ class definition with `SetName()`, `SetID()`, `SetVersion()`, setters and getters

## Design Patterns & Rationale

**Lazy Singleton with static function-local instance:**
- Defers heap allocation to first access; common in 2000s engines where initialization order matters
- Thread-unsafe by design (single-threaded 30 FPS game loop assumption)
- Instance never freed (typical for singletons; intentional leak or implicit cleanup on exit)

**MML Parser Hook Pattern:**
- `parse_mml_scenario()` and `reset_mml_scenario()` follow engine's plugin architecture for config loading
- Allows scenario metadata to be specified declaratively in XML rather than hardcoded
- `reset_mml_scenario()` is a no-op stub, suggesting incomplete support for scenario reloading mid-session

**Fixed-size string truncation (24 chars):**
- Legacy behavior suggesting compatibility lists were originally stored in fixed WAD structures
- Silent truncation could lose data; no error reporting

## Data Flow Through This File

1. **Initialization:** MML parser encounters `<scenario>` XML element ΓåÆ invokes `parse_mml_scenario(root)`
2. **Metadata population:** Parser reads attributes (`name`, `id`, `version`, `allow_classic_gameplay`), calls setters on singleton
3. **Compatibility registration:** Parser iterates `<can_join>` child nodes, truncates strings, appends to `m_compatibleVersions` vector
4. **Runtime validation:** Gameplay and network code calls `IsCompatible(peer_scenario_id)` ΓåÆ linear search of compatibility list ΓåÆ boolean result
5. **Gameplay branching:** Code branches on `AllowsClassicGameplay()` flag to switch between Marathon 1/2 semantics

## Learning Notes

- **Idiomatic to this era/engine:** Lazy singleton with unguarded static (pre-C++11 threading), fixed-size string limits from WAD era, MML hook architecture
- **Modern engines differ:** Would use configuration registries, typed config systems, hot-reload support; thread-safe lazy singletons; no silent data loss
- **Version negotiation pattern:** Scenario compatibility lists enable networked games across patch versionsΓÇöa design decision balancing flexibility vs. protocol complexity

## Potential Issues

- **Thread-safety:** Lazy static without mutex; unsafe if `instance()` called from multiple threads during initialization
- **Data loss:** String truncation to 24 chars silently discards longer IDsΓÇöno validation or error reporting
- **Incomplete reload:** `reset_mml_scenario()` empty stub suggests scenario reload wasn't fully implemented
- **Linear search:** `IsCompatible()` O(n) iteration fine for typical 2ΓÇô5 items, but inelegant
- **Memory leak:** Singleton instance never freed (intentional design, but worth documenting)
