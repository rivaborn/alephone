# Source_Files/Misc/ScenarioChooser.h - Enhanced Analysis

## Architectural Role

ScenarioChooser is a UI subsystem component (housed in Misc) that implements the modal scenario selection dialogΓÇöa critical bootstrap UI invoked during game startup before entering the GameWorld simulation. It bridges three architectural domains: **Files** (scenario discovery and metadata loading via `add_directory`), **CSeries** (cross-platform path resolution and rendering primitives), and **RenderOther** (screen drawing and font rendering). The chooser is the user-facing entry point for selecting which campaign, custom map, or Steam Workshop scenario to play; returning control to the caller with the selected scenario path and workshop flag.

## Key Cross-References

### Incoming (who depends on this)
- **Shell/Interface** (`interface.cpp`, likely `begin_game()` flow): Instantiates `ScenarioChooser`, calls `run()`, and routes the returned scenario path to game initialization (`game_wad`, `XML_MakeRoot` for MML loading)
- **Preferences/Misc subsystem**: May call scenario discovery methods during startup to populate scenario lists or validate installed content

### Outgoing (what this file depends on)
- **Files subsystem**: `add_directory()` internally discovers scenario files via FileHandler path traversal and resource discovery (implicit, not declared here; likely iterates filesystem)
- **CSeries**: Font resource loading (`font_info` forward decl.), platform-specific path utilities for scenario location resolution
- **RenderOther**: Screen drawing functions (`_draw_screen_shape`, `_draw_screen_text`, font rendering) invoked from `redraw()` for grid composition
- **Input subsystem**: `handle_event()` processes SDL keyboard input (arrow keys for navigation, Return/Enter for selection)
- **SDL2**: Window creation (`SDL_Window*` in `redraw()`, `optimize_image()`), event polling, surface/texture management

## Design Patterns & Rationale

**Modal Dialog Pattern**: Construction initializes empty state; `run()` blocks in an event loop until user selection or cancellation, then returnsΓÇöclassic synchronous UI modal. This design simplifies caller logic (no asynchronous callbacks) and ensures scenario is locked before game initialization.

**Lazy Scenario Metadata Loading**: `ScenarioChooserScenario` encapsulates preview image and metadata. The `optimize_image()` call suggests preview images are loaded on-demand and resized/cached to avoid VRAM bloatΓÇöa key tradeoff between responsiveness and memory for scenarios with many large preview images.

**Grid-Based Layout with Dynamic Sizing**: Rather than fixed page sizes, `determine_cols_rows()` calculates grid dimensions based on window size and preview aspect ratio (320├ù240 constant), allowing fluid resizing. Scrolling offset (`scroll_`, `max_scroll_`) is managed per row, enabling keyboard pagination.

**Separation of Concerns**: Scenario aggregation (`add_*` methods) decoupled from display logic (`run()`, `redraw()`), allowing callers to populate from heterogeneous sources (primary game files, workshop, user directories) without display coupling.

## Data Flow Through This File

1. **Initialization**: Caller invokes `add_primary_scenario()`, `add_workshop_scenario()`, `add_directory()` ΓåÆ appends to `scenarios_` vector; internal state reset (cols/rows/scroll to defaults).
2. **Display Loop** (`run()`): Creates SDL window, enters event loop; calls `redraw()` each frame to composite grid of scenario previews onto screen buffer.
3. **Input Handling** (`handle_event()`): Keyboard navigation (arrow keys) updates `selection_` and `scroll_` via `move_selection()`; redraws affected grid region.
4. **Return Path**: User presses Enter ΓåÆ `done_ = true` ΓåÆ `run()` exits loop, returns `std::pair<path, is_workshop>` to caller for game initialization.

Scenario metadata flows inward (file I/O discovers paths), while preview images flow outward (cached to disk via `WadImageCache` implicit dependency).

## Learning Notes

- **Idiomatic Modal UI in SDL2**: This is a textbook example of synchronous blocking UIΓÇöpre-SDL3's asynchronous paradigm. Modern engines (Godot, Unreal) use event-driven or coroutine-based UI, but Aleph One's approach is simpler for a single-threaded startup dialog.
- **Grid Layout Responsiveness**: `determine_cols_rows()` recalculation on window resize (likely triggered by SDL_VIDEORESIZE event) ensures the chooser reflows gracefullyΓÇöa pattern learnable for implementing responsive grid UIs in SDL.
- **Scenario Polymorphism**: By accepting path-based inputs from multiple sources (primary, workshop, custom directories), the chooser is agnostic to scenario originΓÇöa clean abstraction enabling modding/Steam integration without UI changes.

## Potential Issues

1. **Preview Image Caching Coherency**: If `optimize_image()` writes to `WadImageCache`, concurrent scenario mutations (e.g., workshop updates mid-session) could cause stale or corrupted previews. No lock guards are visible here.
2. **Window Resize During Event Loop**: If `handle_event()` receives SDL_VIDEORESIZE but `redraw()` is not defensive about stale col/row state, grid calculations could desync briefly.
3. **Large Scenario Lists**: With 100+ scenarios, scrolling linearity may feel sluggish; no batching or virtual list rendering is evidentΓÇöeach redraw may recalculate all grid cells.
