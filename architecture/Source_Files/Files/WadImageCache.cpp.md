# Source_Files/Files/WadImageCache.cpp

## File Purpose
Implements an on-disk LRU cache for thumbnail images extracted from WAD files. Manages image resizing, storage with UUID-based filenames, and persistence of cache metadata to an INI file with automatic eviction when a size limit is exceeded.

## Core Responsibilities
- Extract images from WAD files and resize them to requested dimensions
- Cache resized images on disk with auto-generated UUID filenames
- Maintain LRU (Least Recently Used) ordering and evict entries when cache size exceeds a configurable limit
- Load/save cache metadata (file paths, checksums, dimensions, sizes) to `Cache.ini`
- Provide fast O(1) lookup of cached images via dual data structure (list + map)
- Support optional autosave on cache modifications
- Handle image format conversions (PNG if available, fallback to BMP)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `WadImageDescriptor` | struct | Uniquely identifies a source image in a WAD file (file path, checksum, wad index, tag type) |
| `cache_key_t` | typedef (tuple) | Composite key: (WadImageDescriptor, width, height) |
| `cache_value_t` | typedef (pair) | Cache entry value: (cached filename, file size in bytes) |
| `cache_pair_t` | typedef (pair) | Complete cache entry: (cache_key_t, cache_value_t) |
| `cache_iter_t` | typedef | Iterator into `m_used` list for fast removal |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_used` | `std::list<cache_pair_t>` | member (private) | Doubly-linked list maintaining LRU order (front = most recent, back = least recent) |
| `m_cacheinfo` | `std::map<cache_key_t, cache_iter_t>` | member (private) | Fast O(log n) lookup: maps cache key to iterator in `m_used` for O(1) updates |
| `m_cachesize` | `size_t` | member (private) | Running total of cached file sizes in bytes; compared against `m_sizelimit` |
| `m_sizelimit` | `size_t` | member (private) | Maximum cache size (default 300 MB); triggers LRU eviction when exceeded |
| `m_autosave` | `bool` | member (private) | If true, `save_cache()` is called automatically on mutations |
| `m_cache_dirty` | `bool` | member (private) | Flag indicating metadata has changed and needs writing to disk |

## Key Functions / Methods

### instance()
- **Signature:** `static WadImageCache* instance()`
- **Purpose:** Singleton accessor; creates and returns the single global instance.
- **Inputs:** None
- **Outputs/Return:** Pointer to the static `WadImageCache` instance
- **Side effects (global state, I/O, alloc):** Allocates instance on first call
- **Calls (direct calls visible in this file):** None visible
- **Notes:** Classic lazy-init singleton pattern; pointer is never freed

### image_from_desc(WadImageDescriptor&)
- **Signature:** `SDL_Surface* image_from_desc(WadImageDescriptor& desc)`
- **Purpose:** Extract and decode image data directly from WAD file without caching.
- **Inputs:** WAD image descriptor (file path, checksum, index, tag)
- **Outputs/Return:** Decoded SDL_Surface pointer or NULL on failure
- **Side effects (global state, I/O, alloc):** Reads WAD file, allocates SDL surface, calls `clear_game_error()`
- **Calls (direct calls visible in this file):** `open_wad_file_for_reading`, `read_wad_header`, `read_indexed_wad_from_file`, `extract_type_from_wad`, `SDL_RWFromConstMem`, `IMG_Load_RW` or `SDL_LoadBMP_RW`, `free_wad`, `close_wad_file`, `clear_game_error`
- **Notes:** Caller must free returned surface with `SDL_FreeSurface`; does not touch cache

### image_from_name(const std::string&)
- **Signature:** `SDL_Surface* image_from_name(std::string& name) const`
- **Purpose:** Load a cached image from disk by filename.
- **Inputs:** Cache filename (typically a UUID string)
- **Outputs/Return:** Decoded SDL_Surface pointer or NULL if file not found or decode fails
- **Side effects (global state, I/O, alloc):** Opens file in image cache directory, allocates SDL surface
- **Calls (direct calls visible in this file):** `FileSpecifier::SetToImageCacheDir`, `FileSpecifier::AddPart`, `FileSpecifier::Open`, `IMG_Load_RW` or `SDL_LoadBMP_RW`
- **Notes:** Private method; caller must free returned surface

### resize_image(SDL_Surface*, int, int)
- **Signature:** `SDL_Surface* resize_image(SDL_Surface* original, int width, int height) const`
- **Purpose:** Resize an image to exact dimensions if needed.
- **Inputs:** Original surface, target width and height
- **Outputs/Return:** Resized surface or NULL if no resize needed or input is NULL
- **Side effects (global state, I/O, alloc):** Allocates new surface if dimensions differ
- **Calls (direct calls visible in this file):** `SDL_Resize`
- **Notes:** Returns NULL if input is NULL or dimensions already match; caller must free result if not NULL

### image_to_new_name(SDL_Surface*, int32*)
- **Signature:** `std::string image_to_new_name(SDL_Surface* image, int32* filesize = NULL) const`
- **Purpose:** Save an image to cache directory with a new UUID-based filename.
- **Inputs:** Surface to encode, optional pointer to receive file size
- **Outputs/Return:** UUID string (filename) on success, empty string on failure
- **Side effects (global state, I/O, alloc):** Generates UUID, writes image file to temp location, renames to final location, may open file to get size
- **Calls (direct calls visible in this file):** Boost UUID generators, `FileSpecifier::SetToImageCacheDir`, `FileSpecifier::AddPart`, `FileSpecifier::SetTempName`, `IMG_SavePNG` or `SDL_SaveBMP`, `FileSpecifier::Rename`, `FileSpecifier::Open`, `OpenedFile::GetLength`
- **Notes:** Uses PNG if `HAVE_SDL_IMAGE` and `HAVE_PNG` defined, else BMP; atomic rename on success

### add_to_cache(cache_key_t, SDL_Surface*)
- **Signature:** `std::string add_to_cache(cache_key_t key, SDL_Surface* surface)`
- **Purpose:** Add or update an image entry in the cache with LRU tracking.
- **Inputs:** Cache key (descriptor, width, height), surface to encode and cache
- **Outputs/Return:** Cached filename (UUID string) on success, empty string on failure
- **Side effects (global state, I/O, alloc):** Calls `image_to_new_name` (I/O), inserts into `m_used` and `m_cacheinfo`, updates `m_cachesize`, calls `apply_cache_limit`, sets `m_cache_dirty`, calls `autosave_cache`
- **Calls (direct calls visible in this file):** `image_to_new_name`, `apply_cache_limit`, `autosave_cache`
- **Notes:** Entry is inserted at front of `m_used` (most recent); triggers cache eviction if needed

### apply_cache_limit()
- **Signature:** `bool apply_cache_limit()`
- **Purpose:** Enforce cache size limit by evicting least-recently-used entries.
- **Inputs:** None (uses member state)
- **Outputs/Return:** True if any entries were deleted, false otherwise
- **Side effects (global state, I/O, alloc):** Pops entries from back of `m_used`, deletes files from disk, updates `m_cacheinfo` and `m_cachesize`, sets `m_cache_dirty`
- **Calls (direct calls visible in this file):** `delete_storage_for_name`
- **Notes:** Loop continues while `m_cachesize > m_sizelimit`; deletes from tail (oldest) first

### retrieve_name(WadImageDescriptor&, int, int, bool)
- **Signature:** `std::string retrieve_name(WadImageDescriptor& desc, int width, int height, bool mark_accessed = true)`
- **Purpose:** Look up a cache entry by key and optionally update LRU order.
- **Inputs:** Descriptor, width, height, whether to mark as accessed
- **Outputs/Return:** Cached filename if found, empty string if not cached
- **Side effects (global state, I/O, alloc):** If `mark_accessed` and entry is not already at front, splices entry to front of `m_used` and sets `m_cache_dirty`
- **Calls (direct calls visible in this file):** `std::list::splice`
- **Notes:** Private method; O(log n) lookup via `m_cacheinfo`; splicing is O(1)

### is_cached(WadImageDescriptor&, int, int)
- **Signature:** `bool is_cached(WadImageDescriptor& desc, int width, int height) const`
- **Purpose:** Check if a specific image size is cached.
- **Inputs:** Descriptor, width, height
- **Outputs/Return:** True if entry exists, false otherwise
- **Side effects (global state, I/O, alloc):** None
- **Calls (direct calls visible in this file):** None
- **Notes:** O(log n) lookup; does not modify LRU order

### cache_image(WadImageDescriptor&, int, int, SDL_Surface*)
- **Signature:** `void cache_image(WadImageDescriptor& desc, int width, int height, SDL_Surface* image = NULL)`
- **Purpose:** Ensure an image is cached at a specific size, loading from WAD or using provided surface.
- **Inputs:** Descriptor, target width and height, optional pre-loaded surface
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** May read WAD file, allocates/frees surfaces, calls `add_to_cache`, `autosave_cache`
- **Calls (direct calls visible in this file):** `retrieve_name`, `autosave_cache`, `resize_image`, `image_from_desc`, `SDL_FreeSurface`, `add_to_cache`
- **Notes:** If already cached, just updates LRU and returns; resizes to exact dimensions before caching

### remove_image(WadImageDescriptor&, int, int)
- **Signature:** `void remove_image(WadImageDescriptor& desc, int width = 0, int height = 0)`
- **Purpose:** Delete cache entries for an image; if width/height are 0, delete all cached sizes.
- **Inputs:** Descriptor, optional specific width/height
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Deletes files from disk, erases from `m_cacheinfo` and `m_used`, updates `m_cachesize`, sets `m_cache_dirty`, calls `autosave_cache`
- **Calls (direct calls visible in this file):** `delete_storage_for_name`, `autosave_cache`
- **Notes:** Partial key match (width/height = 0) requires iteration; full key match is O(log n)

### retrieve_image(WadImageDescriptor&, int, int)
- **Signature:** `SDL_Surface* retrieve_image(WadImageDescriptor& desc, int width, int height)`
- **Purpose:** Load a cached image if it exists.
- **Inputs:** Descriptor, width, height
- **Outputs/Return:** Decoded SDL_Surface pointer or NULL if not cached
- **Side effects (global state, I/O, alloc):** Calls `retrieve_name` (may update LRU), loads from disk
- **Calls (direct calls visible in this file):** `retrieve_name`, `image_from_name`
- **Notes:** Marks as accessed; caller must free returned surface

### get_image(WadImageDescriptor&, int, int, SDL_Surface*)
- **Signature:** `SDL_Surface* get_image(WadImageDescriptor& desc, int width, int height, SDL_Surface* surface = NULL)`
- **Purpose:** Get a cached image or create it if not cached.
- **Inputs:** Descriptor, target width/height, optional pre-loaded surface
- **Outputs/Return:** Decoded SDL_Surface pointer (resized to target dimensions) or NULL if all operations fail
- **Side effects (global state, I/O, alloc):** May read WAD, allocate surfaces, cache image via `add_to_cache`
- **Calls (direct calls visible in this file):** `retrieve_image`, `resize_image`, `image_from_desc`, `SDL_FreeSurface`, `add_to_cache`
- **Notes:** Returns on-demand resized surface; caches result for future use

### initialize_cache()
- **Signature:** `void initialize_cache()`
- **Purpose:** Load cache metadata from disk at startup.
- **Inputs:** None (uses image cache directory and `Cache.ini`)
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Reads INI file, populates `m_used`, `m_cacheinfo`, and `m_cachesize`
- **Calls (direct calls visible in this file):** `FileSpecifier::SetToImageCacheDir`, `FileSpecifier::AddPart`, `FileSpecifier::Exists`, `InfoTree::load_ini`, `logError`, `FileSpecifier` constructor
- **Notes:** Silent return if `Cache.ini` doesn't exist (first-run); logs error on parse failure; reconstructs LRU order from disk

### save_cache()
- **Signature:** `void save_cache()`
- **Purpose:** Persist cache metadata to `Cache.ini` on disk.
- **Inputs:** None (uses member state)
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Writes INI file, clears `m_cache_dirty` flag
- **Calls (direct calls visible in this file):** `InfoTree::put`, `FileSpecifier::SetToImageCacheDir`, `FileSpecifier::AddPart`, `InfoTree::save_ini`, `logError`
- **Notes:** Only writes if `m_cache_dirty` is true; logs error on write failure; formats entries with UUID names as keys in INI

## Control Flow Notes

**Initialization:** `initialize_cache()` is called at startup to load persisted metadata from `Cache.ini`, rebuilding the LRU list and size tracking.

**Frame/Request Processing:** When an image is needed, `get_image()` or `cache_image()` is called. If cached, the entry is retrieved and marked as accessed (moved to front of LRU). If not cached, the image is loaded from the WAD, resized, and added to cache via `add_to_cache()`.

**Cache Eviction:** After each addition, `apply_cache_limit()` is triggered, which walks the LRU list from the back, deleting least-recent entries until `m_cachesize Γëñ m_sizelimit`.

**Persistence:** If `m_autosave` is true (default), `autosave_cache()` (inline) calls `save_cache()`, which writes all metadata to `Cache.ini`. Manual calls to `save_cache()` are also possible.

**Shutdown:** No explicit shutdown method; `save_cache()` can be called before exit to ensure metadata persists.

## External Dependencies

- **SDL2:** `SDL_Surface`, `SDL_RWops`, `SDL_LoadBMP_RW`, `SDL_SaveBMP`; conditionally `SDL2/SDL_image.h` for `IMG_Load_RW`, `IMG_SavePNG`
- **Boost:** UUID generation (`boost/uuid/uuid.hpp`, `boost/uuid/uuid_generators.hpp`, `boost/uuid/uuid_io.hpp`)
- **FileSpecifier** (from `FileHandler.h`): file I/O, path management, temp files
- **WadImageDescriptor, WAD functions** (from `wad.h`): `open_wad_file_for_reading`, `read_wad_header`, `read_indexed_wad_from_file`, `extract_type_from_wad`, `free_wad`, `close_wad_file`, `WadDataType`
- **InfoTree** (from `InfoTree.h`): INI file parsing and serialization
- **Game error system** (`game_errors.h`): `clear_game_error()`
- **SDL_Resize** (`sdl_resize.h`): Lanczos-filtered image resizing
- **Logging** (`Logging.h`): `logError` macro
