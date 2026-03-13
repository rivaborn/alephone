# Source_Files/Files/WadImageCache.h

## File Purpose
Provides an on-disk LRU cache for thumbnail images stored in WAD files, enabling fast retrieval of resized images without re-decoding from the WAD archive. Manages cache persistence, size limits, and access tracking.

## Core Responsibilities
- Singleton instance management for global cache access
- Loading and caching thumbnail images at arbitrary resize dimensions
- LRU eviction when cache size exceeds configured limit
- Persistent cache metadata storage (filenames, sizes) to disk
- Image resizing and format conversion (SDL_Surface)
- Cache validation and expiration based on WAD file checksums

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `WadImageDescriptor` | struct | Uniquely identifies an image in a WAD file (file path, checksum, index, data tag); defines sort/equality operators |
| `cache_key_t` | typedef | Tuple of (descriptor, width, height) for lookups |
| `cache_value_t` | typedef | Pair of (cached filename, file size) |
| `cache_pair_t` | typedef | Pair of cache_key_t and cache_value_t |
| `cache_iter_t` | typedef | Iterator into `m_used` list for O(1) LRU updates |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_used` | `std::list<cache_pair_t>` | static (class member) | LRU-ordered list of cached entries; newest at end |
| `m_cacheinfo` | `std::map<cache_key_t, cache_iter_t>` | static (class member) | Fast lookup: cache key ΓåÆ position in `m_used` list |
| `m_cachesize` | `size_t` | static (class member) | Current total cache size in bytes |
| `m_sizelimit` | `size_t` | static (class member) | Max cache size (default 300 MB); triggers eviction if exceeded |
| `m_autosave` | `bool` | static (class member) | Whether to auto-save cache metadata after modifications |
| `m_cache_dirty` | `bool` | static (class member) | Tracks if cache state needs persisting to disk |

## Key Functions / Methods

### instance
- **Signature:** `static WadImageCache* instance()`
- **Purpose:** Singleton accessor; returns the global cache instance
- **Inputs:** None
- **Outputs/Return:** Pointer to WadImageCache singleton
- **Side effects:** Creates singleton on first call
- **Calls:** Not shown in header
- **Notes:** Standard singleton pattern; lazy initialization likely in .cpp

### initialize_cache
- **Signature:** `void initialize_cache()`
- **Purpose:** Load persisted cache metadata from disk at startup
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Populates `m_used` and `m_cacheinfo` from disk; must be called before other cache methods
- **Calls:** Indirectly calls image loading and map/list manipulation
- **Notes:** Critical initialization step; likely reads cache index file

### image_from_desc (static)
- **Signature:** `static SDL_Surface *image_from_desc(WadImageDescriptor& desc)`
- **Purpose:** Load image at original size directly from WAD file without caching
- **Inputs:** `desc` ΓÇô identifier for image in WAD
- **Outputs/Return:** SDL_Surface pointer (caller must free) or NULL on error
- **Side effects:** None (read-only WAD access)
- **Calls:** WAD file I/O functions (not visible here)
- **Notes:** Utility for loading full-resolution source images; caller responsible for freeing surface

### is_cached
- **Signature:** `bool is_cached(WadImageDescriptor& desc, int width, int height) const`
- **Purpose:** Check if resized image exists in cache without updating LRU
- **Inputs:** `desc`, `width`, `height`
- **Outputs/Return:** `true` if cached at exact dimensions
- **Side effects:** None
- **Calls:** Lookup in `m_cacheinfo` map
- **Notes:** Read-only check; does not mark accessed

### cache_image
- **Signature:** `void cache_image(WadImageDescriptor& desc, int width, int height, SDL_Surface *surface = NULL)`
- **Purpose:** Add image to cache (or update LRU if already present); skips if original and cached size match
- **Inputs:** `desc`, target `width`, `height`, optional pre-rendered `surface`
- **Outputs/Return:** None
- **Side effects:** Adds/updates entry in `m_used` and `m_cacheinfo`; writes image file to disk; increments `m_cachesize`; may trigger auto-save
- **Calls:** `add_to_cache()`, possibly `apply_cache_limit()`, `autosave_cache()`
- **Notes:** If `surface` is provided, uses it instead of reading WAD; updates LRU if already cached

### retrieve_image
- **Signature:** `SDL_Surface *retrieve_image(WadImageDescriptor& desc, int width, int height)`
- **Purpose:** Load a cached image by resized dimensions
- **Inputs:** `desc`, `width`, `height`
- **Outputs/Return:** SDL_Surface pointer (caller must free) or NULL if not cached
- **Side effects:** Updates LRU access order
- **Calls:** `retrieve_name()` (internal), image loading from disk
- **Notes:** Returns NULL if cache miss; caller responsible for freeing returned surface

### get_image
- **Signature:** `SDL_Surface *get_image(WadImageDescriptor& desc, int width, int height, SDL_Surface *surface = NULL)`
- **Purpose:** Load image, caching it if necessary; primary public cache interface
- **Inputs:** `desc`, `width`, `height`, optional pre-rendered `surface`
- **Outputs/Return:** SDL_Surface pointer (caller must free)
- **Side effects:** May add to cache; updates LRU; triggers size limit checks and auto-save
- **Calls:** `retrieve_image()`, `cache_image()`, possibly `image_from_desc()`
- **Notes:** Convenience method combining retrieval and caching; always returns a surface (reads WAD if needed)

### save_cache / set_cache_autosave
- **Signature:** `void save_cache()`, `void set_cache_autosave(bool enabled)`
- **Purpose:** Persist cache metadata to disk; enable/disable auto-save on mutations
- **Inputs:** `enabled` (for autosave setter)
- **Outputs/Return:** None
- **Side effects:** Writes cache index file; resets `m_cache_dirty`
- **Calls:** Disk I/O; called by `autosave_cache()`
- **Notes:** `set_cache_autosave()` controls whether `cache_image()` / `remove_image()` auto-persist

### set_limit / size / limit
- **Signature:** `void set_limit(size_t bytes)`, `size_t size()`, `size_t limit()`
- **Purpose:** Manage cache size limits; query current state
- **Inputs:** `bytes` ΓÇô new size limit
- **Outputs/Return:** Current cache size or limit
- **Side effects:** `set_limit()` may trigger eviction and auto-save if new limit is lower
- **Calls:** `apply_cache_limit()`, `autosave_cache()`
- **Notes:** `size()` and `limit()` are simple getters; `set_limit()` enforces immediately

## Control Flow Notes
- **Initialization:** `initialize_cache()` must be called once at engine startup (not shown in frame but critical precondition)
- **Frame/Render:** `get_image()` is called when WAD thumbnails are needed; returns cached surface if available, otherwise decodes and caches
- **Cleanup/Shutdown:** `save_cache()` explicitly persists state; destructor not shown but likely calls it
- **Eviction:** LRU list order maintained per access; `apply_cache_limit()` (private) removes oldest entries when size exceeded

## External Dependencies
- **FileHandler.h:** `FileSpecifier` for WAD file paths and operations
- **wad.h:** `WadDataType` enum/typedef for image tags in WAD archives
- **SDL2 (SDL_Surface):** Image buffer format and manipulation
- **Standard Library:** `<tuple>`, `<list>`, `<map>`, `<string>` (STL containers)
- **Not directly visible:** WAD I/O functions (likely in wad.cpp or separate WAD reader); disk I/O for cache files
