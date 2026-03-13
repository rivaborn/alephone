# Source_Files/Files/WadImageCache.cpp - Enhanced Analysis

## Architectural Role

WadImageCache serves as the **UI-layer image cache** bridging WAD file asset extraction and rendering system needs. Unlike the GPU texture cache in RenderMain (which holds decoded surfaces in VRAM), this cache persists resized thumbnail images to disk, enabling fast scenario selection UI and mod browser interactions without repeatedly extracting images from large WAD archives. It's positioned between the Files subsystem (WAD I/O) and UI/rendering systems that display map previews, ensuring UI remains responsive during scenario enumeration.

## Key Cross-References

### Incoming (Dependents)
- **ScenarioChooser** (Misc/ScenarioChooser.h): Enumerates scenario metadata and calls `cache_image()` / `retrieve_image()` to populate preview UI without blocking
- **preferences_widgets_sdl.cpp**: Workshop mod browsing UI fetches cached thumbnails via `get_image()`
- **RenderOther**: Screen/HUD rendering may use cached images for overlay displays
- **Network**: Multiplayer game lobbies may cache team color variations

### Outgoing (Dependencies)
- **WAD reader functions** (wad.h): `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `extract_type_from_wad()` ΓÇö lazy extraction on cache miss
- **FileSpecifier** (FileHandler.h): Image cache directory path resolution, atomic temp-file rename pattern
- **InfoTree** (XML/InfoTree.h): INI metadata serialization (load/save Cache.ini)
- **SDL2**: Surface decoding (SDL_LoadBMP_RW, IMG_Load_RW), encoding (IMG_SavePNG), resizing
- **Boost UUID**: Collision-free cache filenames (`boost::uuids::random_generator`)

## Design Patterns & Rationale

**Dual-Structure LRU Cache** (list + map):
- `m_used` (std::list) maintains insertion order at O(1) splice cost for LRU updates
- `m_cacheinfo` (std::map) provides O(log n) lookup by composite key, storing iterator into list
- Balances fast access (lookup) with fast eviction (tail removal) ΓÇö O(log n) lookup is acceptable for UI-layer cache, faster than O(n) map iteration
- Classic pattern since pre-C++11 when hashtables weren't standard

**Lazy Initialization**: `initialize_cache()` reconstructs LRU state from disk on first access. Silent failure (no exception on missing Cache.ini) allows fresh cache on first run.

**Atomic Rename Pattern**: Write to temp file, rename to final UUID filename only on success. Prevents partial writes or corruption if process crashes mid-save.

**Stateful Dirty Flag** (`m_cache_dirty`): Batches metadata updates to avoid frequent INI writes. Only persists when flag is set ΓÇö performance optimization for high-throughput caching.

**Optional Pre-Loaded Surface**: `cache_image()` and `get_image()` accept optional pre-loaded surfaces to avoid re-extracting if caller already has a copy. Reduces duplicate WAD I/O.

## Data Flow Through This File

**Initialization**: App startup ΓåÆ `initialize_cache()` reads Cache.ini ΓåÆ reconstructs `m_used` list and `m_cacheinfo` map ΓåÆ pre-populates `m_cachesize` for limit enforcement.

**Cache Miss / Get**:
1. `get_image(desc, w, h)` calls `retrieve_image()` ΓåÆ cache miss
2. Load from WAD: `image_from_desc(desc)` ΓåÆ `open_wad_file_for_reading()` ΓåÆ read and decompress image data
3. Resize to exact dimensions: `resize_image()` ΓåÆ `SDL_Resize()`
4. Encode and write: `add_to_cache()` ΓåÆ `image_to_new_name()` ΓåÆ generate UUID ΓåÆ write temp file ΓåÆ atomic rename
5. Return resized surface to caller; caller must `SDL_FreeSurface()`

**Cache Hit**:
1. `retrieve_image()` finds entry in `m_cacheinfo` map
2. Update LRU: splice entry to front of `m_used` list
3. Load from cache directory: `image_from_name()` ΓåÆ open cached file ΓåÆ decode
4. Return surface

**Eviction** (after each `add_to_cache()`):
1. While `m_cachesize > m_sizelimit`, pop back of `m_used` (least recent)
2. Delete on-disk file: `delete_storage_for_name()`
3. Erase from `m_cacheinfo` map
4. Update running size total
5. Set `m_cache_dirty = true`

**Persistence**: If `m_autosave` enabled, `autosave_cache()` (inline, not shown) calls `save_cache()` after mutations, writing INI with all metadata keyed by UUID filename.

## Learning Notes

- **WAD extraction is lazy**: Images are only extracted when first requested, not enumerated upfront. This is essential for responsive scenario selection with hundreds of mods.
- **Filesystem cache over VRAM**: Unlike GPU texture caches, this persists across process exit, reducing startup latency when reparsing the same scenarios.
- **INI as cache manifest**: Human-readable metadata enables manual cache debugging and corruption inspection without binary parsing.
- **Pre-PNG fallback**: Conditional compilation (`HAVE_SDL_IMAGE && HAVE_PNG`) reflects an older codebase where PNG support was optional; modern engines would assume PNG always.
- **No cache coherency mechanism**: If a WAD file is modified on disk, the cached images become stale. Relies on checksum field for manual invalidation (e.g., modder rebuilds WAD, changes checksum).

## Potential Issues

1. **Memory Leak (Singleton)**: Line 38ΓÇô44 creates static pointer but never deletes. No destructor shown, so `m_used`, `m_cacheinfo` leak on process exit. Minor (reclaimed by OS) but sloppy by modern standards.

2. **Thread Safety**: No mutex protecting `m_cacheinfo` and `m_used`. If two threads call `retrieve_image()` + `add_to_cache()` concurrently, race conditions can corrupt the LRU list or cause iterator invalidation.

3. **Orphaned Cache Files**: If Cache.ini is lost but cached PNG/BMP files remain, they become unreferenced waste. No cleanup mechanism. Filling disk over time if cache dir isn't manually purged.

4. **Checksum as Weak Invalidation**: Relies on `WadImageDescriptor::checksum` field to detect stale cache. If WAD is modified but checksum isn't updated, cached thumbnail won't refresh. No automatic invalidation on WAD file mtime.

5. **No Cache Size Limit at Startup**: `m_sizelimit` (and its value) not shown in this file; if set very high, first app run could exhaust disk before eviction kicks in.
