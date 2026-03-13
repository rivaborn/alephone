# Source_Files/XML/QuickSave.cpp - Enhanced Analysis

## Architectural Role

QuickSave.cpp is the **save game manager subsystem**, bridging the GameWorld (for reading current state), the Files subsystem (for WAD persistence), the Rendering pipeline (for preview images), and the SDL UI framework (for dialogs and widgets). It lives at the application layer where save/load is explicitly user-driven or automatic, not in the core game loopΓÇöenabling graceful enumeration and metadata manipulation without holding locks on live world state. The module creates a time-based naming convention and LRU image cache, trading per-save file I/O simplicity for memory-bounded preview management.

## Key Cross-References

### Incoming (who depends on this file)

- **Shell/Interface layer** (interface.cpp, shell.cpp): Calls `load_quick_save_dialog()` and `create_quick_save()` during game lifecycle transitions (menu ΓåÆ load, gameplay ΓåÆ auto-save)
- **saved_game_was_networked()**: Queried after loading to flag multiplayer games for proper network setup
- **Dialog framework** (sdl_dialogs.h, sdl_widgets.h): `w_saves` inherits from `w_list_base`; dialog callbacks (`dialog_rename`, `dialog_delete`, `dialog_export`) are invoked by button clicks
- **Tests/replay validation**: May check save file integrity across version updates

### Outgoing (what this file depends on)

- **Files subsystem**: FileSpecifier (path management, dialogs), game_wad.h (`build_save_metadata()`, `build_meta_game_wad()`), WAD I/O functions, wad.h for reading/writing save containers
- **GameWorld**: `static_world`, `dynamic_world`, `local_player` for reading current state (position, level name, playtime ticks)
- **Screen/Rendering**: `_set_port_to_custom()`, `_restore_port()`, `_render_overhead_map()` for preview generation; `draw_surface` global
- **WadImageCache**: Retrieves scaled preview images from `.sgaA` WAD files via `WadImageCache::instance()->get_image()`
- **SDL2/SDL_image**: `IMG_SavePNG_RW()` or `SDL_SaveBMP_RW()` for preview serialization; `SDL_FreeSurface()` for cache cleanup
- **CSeries**: Resource strings (`getcstr()`), string encoding (`utf8_to_mac_roman()`, `mac_roman_to_utf8()`), theme/color queries
- **InfoTree**: Metadata parsing/generation via `InfoTree::load_ini()` for save attributes

## Design Patterns & Rationale

| Pattern | Where | Why |
|---------|-------|-----|
| **Singleton** | `QuickSaveImageCache::instance()`, `QuickSaves::instance()` | Single global cache per process; prevents duplicate preview loading; simplifies dialog state |
| **LRU Cache** | `QuickSaveImageCache` with `m_used` (list) + `m_images` (map) | Limits memory to 100 preview surfaces; maintains recency for viewport performance; splice-based eviction is O(1) |
| **Modal Dialog** | `load_quick_save_dialog()`, `dialog_rename()`, `dialog_delete()`, `dialog_export()` | Blocks game loop until user commits/cancels; preserves undo-safe state; nested dialogs for confirmation |
| **Callback Delegation** | `static void dialog_*` with `void *arg` cast | Widget framework convention; avoids template bloat; tight binding to parent dialog lifetime |
| **Timestamp-based Naming** | `save_time` as filename | Auto-avoids collisions; sortable lexicographically; enables "last played" ranking; no manual naming collision logic |
| **Metadata Sidecar** | INI embedded in WAD at `SAVE_GAME_METADATA_INDEX` | Preserves all metadata with save file; avoids separate `.ini` files; WAD atomic write guarantees consistency |

**Rationale for architecture choices:**
- **Preview is static, not dynamic**: Captured at save time to avoid stalling on load; previews cache as `.sgaA` in save file, not regenerated per render
- **Image cache is separate from file cache**: Decouples preview rendering (expensive) from enumerate (frequent); LRU limits VRAM, not disk
- **Dialogs don't modify `QuickSaves` instance directly**: Callbacks reconstruct WAD and call `update_selected()`; avoids holding locks during I/O
- **No threading**: File I/O and rendering happen on main thread during dialog modal loop; assumes save directory is on fast local disk

## Data Flow Through This File

```
CREATE PATH:
  create_quick_save()
    Γö£ΓöÇ read current state: local_player->location, dynamic_world, level_name
    Γö£ΓöÇ call build_map_preview() ΓåÆ render overhead map ΓåÆ save as PNG/BMP
    Γö£ΓöÇ call build_save_metadata() ΓåÆ format timestamps, player count ΓåÆ INI bytes
    Γö£ΓöÇ call save_game_file() [Files subsystem] ΓåÆ WAD containing game state + metadata
    ΓööΓöÇ if count > max_quick_saves: delete_surplus_saves() [QuickSaves::instance()]

LOAD PATH:
  load_quick_save_dialog()
    Γö£ΓöÇ QuickSaves::instance()->enumerate()
    Γöé  ΓööΓöÇ QuickSaveLoader::ParseDirectory() ΓåÆ scan .sgaA files
    Γöé     ΓööΓöÇ ParseQuickSave() for each ΓåÆ read WAD header, extract metadata (time, level, players)
    Γö£ΓöÇ display w_saves widget with 128├ù72 preview + name/time/level/duration text
    Γö£ΓöÇ user selects item
    Γö£ΓöÇ w_saves::draw_item()
    Γöé  ΓööΓöÇ QuickSaveImageCache::instance()->get(save_time)
    Γöé     Γö£ΓöÇ LRU cache hit? return from m_used list
    Γöé     ΓööΓöÇ miss? WadImageCache::get_image() ΓåÆ load .sgaA ΓåÆ cache with LRU eviction
    Γö£ΓöÇ on user confirm: return selected save_file FileSpecifier + set last_saved_networked
    ΓööΓöÇ on exit: QuickSaveImageCache::clear() + QuickSaves::clear()

UPDATE PATH (rename/delete/export):
  dialog_rename() / dialog_delete() / dialog_export()
    Γö£ΓöÇ show confirmation dialog with save preview
    Γö£ΓöÇ on confirm:
    Γöé  Γö£ΓöÇ rename: create_updated_save(sel) ΓåÆ read WAD, extract game data + image, rebuild with new INI
    Γöé  Γö£ΓöÇ delete: delete_quick_save(sel) ΓåÆ unlink file from disk
    Γöé  ΓööΓöÇ export: FileSpecifier::CopyContents() ΓåÆ user-chosen path
    Γö£ΓöÇ update parent dialog state: disable buttons if no saves left
    ΓööΓöÇ QuickSaves::instance() is NOT modified (next enumerate() reflects disk state)

IMAGE CACHE LIFECYCLE:
  First render of save item ΓåÆ cache miss ΓåÆ WadImageCache loads from .sgaA ΓåÆ LRU splice to front
  ΓåÆ subsequent renders hit m_images map + list head ΓåÆ no disk I/O
  ΓåÆ if cache > 100: evict list tail, erase from m_images map, SDL_FreeSurface()
  ΓåÆ dialog exit: clear() ΓåÆ free all surfaces + both containers
```

## Learning Notes

**For developers studying this engine:**

1. **Time-based save naming is idiomatic**: Saves are ordered by filesystem modification time, not an explicit version sequence; enables "load last save" without metadata scanning.
2. **Metadata is composable**: `QuickSave` struct (name, level_name, save_time, players, formatted_time, formatted_ticks) is designed for UI display without re-parsing WAD.
3. **Overhead map preview is platform abstraction**: `_render_overhead_map()` abstracts rendering backend (software/OpenGL); produces PNG or BMP depending on `HAVE_SDL_IMAGE`; decouples save system from renderer.
4. **LRU is manual, not templated**: Using explicit `std::list + std::map` for LRU teaches intent clearly; a modern C++ engine might use `std::unordered_map` with timestamp-based eviction or a ring buffer.
5. **Dialog widget hierarchy mimics macOS/Win32 SDK**: `w_saves : w_list_base` mirrors native platform widget trees; callbacks are function pointers (`dialog_ok`, `dialog_cancel`) matching legacy C-style UI frameworks.
6. **Metadata is WAD-embedded, not sidecar files**: No `.sgaA.ini` files on disk; metadata is part of WAD, ensuring atomic save/load. This era predates JSON/YAML metadata.
7. **Enumeration is lazy**: `QuickSaves::instance()->enumerate()` is called on demand (load dialog open), not at startup; saves are not held in memory during gameplay.

## Potential Issues

- **LRU cache 100-item limit is arbitrary**: If a user has 101+ saves, every load dialog renders item 101 with cache eviction on item 2. For large collections, should tie to VRAM budget, not item count.
- **Preview generation blocks main thread**: `build_map_preview()` calls `_render_overhead_map()` during `create_quick_save()`; if rendering is GPU-heavy, auto-save stalls gameplay. Modern engines offload this to worker thread.
- **Dialog callback void-pointer casts are fragile**: `dialog_rename(void *arg)` casts to `dialog *d`; no type checking. A callback signature change breaks all sites. Modern C++ would use `std::function<>` or lambdas with captures.
- **Limited error recovery in image cache**: `QuickSaveImageCache::get()` returns `nullptr` on load failure; calling code (`w_saves::draw_item`) blits `nullptr` surface (undefined behavior if not checked). Should return placeholder image or handle gracefully.
- **No thread-safety on cache**: `QuickSaveImageCache::instance()` is not `thread_safe`; if rendering happens off-thread, concurrent access to `m_used` / `m_images` causes data race. Unlikely in this architecture but a latent issue.
- **Delete does not invalidate UI iterator**: If `dialog_delete` removes save from disk, but enumeration happened once at dialog open, UI may have stale iterator. Mitigated by re-enumerate on next load dialog, but confusing state interim.
