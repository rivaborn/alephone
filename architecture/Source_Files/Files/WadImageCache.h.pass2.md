# Source_Files/Files/WadImageCache.h - Enhanced Analysis

## Architectural Role

WadImageCache acts as a **performance bridge between the Files subsystem (WAD loading) and Rendering subsystem (image display)**. It shields the rest of the engine from expensive thumbnail re-decoding by maintaining an on-disk LRU cache, keyed by both source identity (checksum) and target dimensions. The cache persists across sessions, amortizing startup costs for scenario loading and preview rendering in UI contexts.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderOther/** (screen.cpp, images.cpp) ΓÇô Calls `get_image()` / `retrieve_image()` when rendering WAD thumbnails in UI overlays
- **Misc/** (Scenario.cpp, preferences_widgets_sdl.cpp) ΓÇô Uses cache for scenario/screenshot previews in selection menus
- **game_wad.cpp** ΓÇô May coordinate via checksum when saving/loading to detect stale cache entries
- **WadImageCache.cpp** ΓÇô Internal implementation accesses `image_from_desc()` (static helper) and resizes via `resize_image()`

### Outgoing (what this file depends on)
- **FileHandler.h** ΓÇô `FileSpecifier` for cross-platform path abstraction; used as part of descriptor equality
- **wad.h** ΓÇô `WadDataType` enum for image classification; used to uniquely identify assets
- **SDL2** ΓÇô `SDL_Surface` for image buffer lifecycle; caller owns memory (must free returned pointers)
- **Standard Library** ΓÇô `<tuple>`, `<list>`, `<map>` for dual-structure LRU + fast lookup

## Design Patterns & Rationale

**Dual-Structure LRU Cache:**  
`m_used` (linked list) maintains insertion/access order; `m_cacheinfo` (map from tuple key ΓåÆ iterator) enables O(1) lookups and O(1) LRU updates. This is idiomatic for efficient LRU without reshuffling.

**Composite Descriptor as Key:**  
`tuple<WadImageDescriptor, width, height>` bundles source identity with requested dimensions. The descriptor's multi-level comparison operator (file ΓåÆ checksum ΓåÆ index ΓåÆ tag) ensures deterministic STL container ordering; checksums act as simple change-detection (invalidates cache if WAD modified).

**Lazy Caching with Optional Passthrough:**  
`cache_image()` and `get_image()` accept optional `surface` parameter, allowing callers to pass pre-rendered surfaces and avoid WAD I/O. This optimizes the hot path when images are already decoded (e.g., during batch operations).

**Persistent Metadata via Flag:**  
`m_cache_dirty` + `m_autosave` enable batched disk writes; finer-grained than writing every mutation, coarser than manual `save_cache()` calls.

**Why not single map?**  
A map-only design would require either storing full surfaces in-memory (huge memory footprint) or expensive disk re-lookup on access. The list preserves insertion order for eviction; the map provides lookup speed.

## Data Flow Through This File

1. **Ingestion:** `get_image(descriptor, width, height, optional_surface)` receives asset ID + target size
2. **Lookup:** `retrieve_name()` queries `m_cacheinfo`; cache hit returns on-disk filename, miss triggers reload
3. **Transformation:** 
   - Cache miss ΓåÆ `image_from_desc()` loads from WAD at original size
   - Requested dimensions Γëá original? `resize_image()` scales and encodes to disk (JPEG/PNG)
   - Updates `m_cachesize`; `apply_cache_limit()` evicts oldest entries if exceeded
4. **Output:** Returns `SDL_Surface*` (caller responsible for `SDL_FreeSurface()`)
5. **Persistence:** `save_cache()` writes metadata (descriptor ΓåÆ filename mapping) to INI/binary index on disk

## Learning Notes

**Idioms from this era (~2015):**
- Manual memory management (caller must free `SDL_Surface*`) is pre-RAII; modern engines use smart pointers
- `tuple<>` for composite keys predates `struct Key { ... }` custom comparators; simpler but less readable
- Disk persistence of cache metadata is unusual; modern engines often use file hashing (mtime-based invalidation) or transient-only caches
- The `static` singleton pattern (vs. dependency injection) reflects pre-modern C++ practices

**Instructive aspects:**
- Two-level lookup (map ΓåÆ disk file ΓåÆ SDL_Surface) shows how to trade memory for speed
- Iterator caching in `m_cacheinfo` is a performance detail worth remembering for custom containers
- The `FileSpecifier::GetPath()` call in `WadImageDescriptor::operator<` reveals cross-subsystem naming conventions

## Potential Issues

1. **Checksums as sole invalidation:** If a WAD is modified in-place without changing its filename, stale cache entries persist. No timestamp or version field mitigates this.
2. **No thread safety:** Singleton `instance()` + mutable `m_used`/`m_cacheinfo` are unsafe under concurrent reads from rendering + UI threads. Likely relies on engine-wide frame-lock discipline.
3. **Iterator invalidation risk:** If `m_used.erase()` is called during `apply_cache_limit()`, iterators stored in `m_cacheinfo` become invalid. Suggests careful implementation required in .cpp.
4. **Disk space unbounded initially:** Default 300MB limit is hardcoded; no fallback if disk full during `cache_image()` write.
