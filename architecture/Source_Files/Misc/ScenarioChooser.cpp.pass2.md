# Source_Files/Misc/ScenarioChooser.cpp - Enhanced Analysis

## Architectural Role

ScenarioChooser implements a **modal pre-game UI subsystem** that intercepts the application startup flow after configuration but before GameWorld initialization. It discovers available scenarios across primary and workshop directories, displays them as an image grid, and blocks execution until the user selects one or exits. This sits outside the main game loop (which runs at 30 FPS in GameWorld); instead, it runs its own event loop at ~33 FPS until selection completes, making it a synchronous gateway between shell-level application startup and scenario-specific world initialization.

## Key Cross-References

### Incoming (who depends on this file)
- **Source_Files/Misc/interface.cpp:begin_game()** ??? Called after preferences/map loading, returns selected scenario path to drive next initialization phase
- **Source_Files/Misc/shell.cpp** (main event loop) ??? Invokes ScenarioChooser::run() to block and solicit user selection before transitioning to GameWorld
- **Source_Files/Files/find_files.h:FileFinder** (base class) ??? Inherited by TitleScreenFinder for recursive directory traversal pattern

### Outgoing (what this file depends on)
- **Source_Files/CSeries/cseries.h** ??? DirectorySpecifier, FileSpecifier, dir_entry, OpenedFile abstractions for cross-platform file handling
- **Source_Files/Files/find_files.h** ??? FileFinder base class and recursive discovery infrastructure
- **Source_Files/RenderOther/images.h** ??? find_title_screen(), find_m1_title_screen() to extract image resources from scenario .imgA/.appl resource forks
- **Source_Files/RenderOther/sdl_fonts.h** ??? (included but unused; likely historical vestige)
- **Source_Files/Input/joystick.h** ??? joystick_added(), joystick_removed() callbacks for hot-plug device lifecycle
- **SDL2/SDL2_image** ??? IMG_Load_RW, SDL_LoadBMP_RW, SDL_CreateWindow, SDL_PollEvent, surface blitting
- **Boost.PropertyTree, Boost.Algorithm** ??? XML parsing and string predicates

## Design Patterns & Rationale

### 1. **FileFinder Callback Pattern** (TitleScreenFinder)
- Extends FileFinder abstract base; implements `found()` callback invoked during recursive directory traversal
- Allows stateful image extraction without exposing iteration details; callback moves responsibility to finder
- **Rationale:** Separates concern of "what to do when we find a .imgA file" (extract image) from "how to traverse directories" (FileFinder handles that)

### 2. **RAII Resource Management** (SurfacePtr, WindowPtr)
- Custom deleters (`SDL_FreeSurface`, `SDL_DestroyWindow`) bound to `std::unique_ptr` at compile time
- Guarantees deterministic cleanup even if exception thrown or early return taken
- **Rationale:** Pre-C++17 code avoiding `std::unique_ptr<SDL_Surface, decltype(...)>` template boilerplate by typedef

### 3. **Grid Layout Computation** (determine_cols_rows, ensure_selection_visible)
- Single centralized function computes rows, columns, offsets, and scroll bounds *once* at window creation
- Both rendering and mouse hit-testing reuse computed layout; no duplicate geometry logic
- **Rationale:** Prevents layout desync between rendering and input; single source of truth for grid geometry

### 4. **Lazy Image Loading with Fallback Chain**
- Load priority: chooser.png (if SDL_image available) ??? chooser.bmp ??? recursive search for title screens
- Defers expensive recursive search until needed, avoiding startup latency
- **Rationale:** Most scenarios have local thumbnail; expensive fallback only if local asset missing

### 5. **Primary vs. Secondary Scenario Sorting**
- Comparator prioritizes `is_primary` flag before lexicographic name comparison
- Separates built-in scenarios from user workshop additions at top of list
- **Rationale:** UX convention: stable/official content before mods

## Data Flow Through This File

```
INITIALIZATION PHASE:
  Shell calls begin_game()
      ??? Scenario paths added via add_scenario() / add_directory()
      ??? scenarios_ vector populated with metadata (path, name, is_primary, is_workshop)

PRESENTATION PHASE (run() called):
  Scenarios sorted by comparator (primary first, then name)
  SDL window created fullscreen at desktop resolution
  For each scenario:
    optimize_image() resizes thumbnail to 320x240, converts to window surface format
  Event loop (30ms per tick):
    handle_event() reads SDL events (keyboard, mouse, controller)
      ??? Updates selection_ (index), scroll_ (offset), done_ (exit flag)
    redraw() clears window, blits visible scenarios with selection highlight
    SDL_Delay(30ms)
  
COMPLETION:
  returns (scenarios_[selection_].path, scenarios_[selection_].is_workshop)
  Caller transitions to scenario initialization (load world, Lua, etc.)
```

**Key state machine:**
- `selection_ = -1` (no selection) initially; first arrow-down selects index 0
- `done_ = false` until user presses Return/Enter/Click/A-button, then exit
- `scroll_` clamped to `[0, max_scroll_]`; keyboard/mouse scroll updates; ensure_selection_visible() centers selected item

## Learning Notes

### Idiomatic Patterns in This Engine Era
1. **Conditional compilation** (e.g., `#ifdef HAVE_SDL_IMAGE`) reflects mid-2000s SDL2 adoption; graceful degradation if SDL_image not available
2. **XML configuration via Boost.PropertyTree** shows preferred config approach; MML/XML metadata replaces Marathon 1's binary resource forks
3. **FileFinder callback pattern** is engine convention for recursive file discovery; reused in shader loading, resource enumeration
4. **SDL_RWops abstraction** (via OpenedFile) isolates from filesystem; allows future support for ZIP archives, network resources

### Modern Engine Alternatives
- **State machine**: Would use explicit enum state (e.g., `enum UIState { Idle, Selecting, Selected }`) rather than trio of variables
- **Geometry**: Would parameterize 320x240 and margin/spacing as constants in header
- **Image loading**: Modern engines defer resize to texture upload, use async image loading
- **Event handling**: Would use event-driven callbacks rather than switch-case dispatching

## Potential Issues

### 1. **Hard Exit via exit(0)**
```cpp
case SDL_CONTROLLER_BUTTON_B:
    exit(0);  // ??? Terminates immediately, no cleanup
```
- **Risk:** Global state (open files, unfinished I/O, unsaved prefs) may not flush
- **Mitigation:** Should signal shutdown flag instead, allowing caller to clean up

### 2. **Silent XML Parse Failure**
```cpp
try {
    name = tree.get<std::string>("marathon.scenario.<xmlattr>.name");
} catch (const boost::property_tree::ptree_error&) {
    // ??? Silently ignores malformed XML; falls through to directory name
}
```
- **Risk:** User never knows if scenario name came from config or fallback; could mask config file corruption
- **Mitigation:** Log warning on parse failure (cslog.h)

### 3. **Unchecked Selection on Return**
```cpp
return std::make_pair(scenarios_[selection_].path, scenarios_[selection_].is_workshop);
```
- **Risk:** If run() called on empty scenarios_ vector, `selection_ = -1` causes out-of-bounds access
- **Mitigation:** Assert `scenarios_.size() > 0` before run(), or initialize selection to 0 if non-empty

### 4. **Mouse Hit-Test Coordinate Math**
```cpp
if (x >= left && x < left + scenario_width &&
    y >= top && y <= top + scenario_height)  // ??? Inconsistent: < vs <=
```
- **Risk:** Off-by-one at bottom-right corner; could select unintended cell on boundary
- **Mitigation:** Use consistent comparison operators (both `<` or both `<=`)
