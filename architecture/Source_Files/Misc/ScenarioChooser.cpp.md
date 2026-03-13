# Source_Files/Misc/ScenarioChooser.cpp

## File Purpose
Implements a fullscreen graphical scenario selector UI. Displays available game scenarios as a grid of thumbnail images with keyboard, mouse, and controller navigation. Returns the selected scenario path to the caller.

## Core Responsibilities
- Load and parse scenario metadata (name, images) from directories
- Create SDL fullscreen window and render scenario grid
- Handle keyboard, mouse, and game controller input events
- Compute grid layout with dynamic column/row sizing and scrolling
- Keep selected item visible and navigate by arrow keys, mouse, or Tab
- Scale and optimize scenario thumbnail images for display

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ScenarioChooserScenario` | class | Represents a single scenario with path, name, image, and metadata flags |
| `TitleScreenFinder` | class | FileFinder subclass that searches for and extracts title screen images from scenario files |
| `SurfacePtr` | typedef | Smart pointer for SDL_Surface with custom deleter |
| `WindowPtr` | typedef | Smart pointer for SDL_Window with custom deleter |

## Global / File-Static State
None (all mutable state is instance-scoped within `ScenarioChooser` class).

## Key Functions / Methods

### ScenarioChooserScenario::operator<
- **Signature:** `bool operator<(const ScenarioChooserScenario& other) const`
- **Purpose:** Sort comparator; primary scenarios sort before secondary, then by name (case-insensitive)
- **Inputs:** Another scenario
- **Outputs/Return:** `true` if this scenario should sort before other
- **Side effects:** None
- **Calls:** `std::lexicographical_compare`
- **Notes:** Used to order scenario list for display

### ScenarioChooserScenario::load
- **Signature:** `bool load(const std::string& path)`
- **Purpose:** Load scenario metadata and thumbnail from a directory; attempts to read scenario name from XML metadata and loads `.png` or `.bmp` thumbnail
- **Inputs:** Filesystem path to scenario directory
- **Outputs/Return:** `true` if scenario was loaded successfully
- **Side effects:** Populates `name`, `path`, `image`; may call `find_image()` if no bundled thumbnail found
- **Calls:** `DirectorySpecifier`, `boost::property_tree::read_xml`, `IMG_Load_RW`, `SDL_LoadBMP_RW`, `find_image()`
- **Notes:** Reads XML config to extract scenario name; falls back to directory name if XML parse fails; searches for title screen if no built-in thumbnail

### ScenarioChooserScenario::find_image
- **Signature:** `void find_image()`
- **Purpose:** Recursively search scenario directory for title screen images in `.imgA` or `.appl` resources
- **Inputs:** None (uses member `path`)
- **Outputs/Return:** None (updates member `image`)
- **Side effects:** Populates `image` via `TitleScreenFinder`
- **Calls:** `TitleScreenFinder::Find()`

### ScenarioChooser::run
- **Signature:** `std::pair<std::string, bool> run()`
- **Purpose:** Main UI entry point; displays fullscreen window, processes user input, and returns selected scenario
- **Inputs:** Populated scenario list (added via `add_scenario` / `add_directory`)
- **Outputs/Return:** Pair of (scenario path, is_workshop flag)
- **Side effects:** Creates SDL window, modifies global state (display, input), blocks until user selects or exits
- **Calls:** `std::sort`, `SDL_GetDesktopDisplayMode`, `SDL_CreateWindow`, `optimize_image`, `handle_event`, `redraw`, `SDL_Delay`, `SDL_PollEvent`
- **Notes:** Event loop runs at ~33 FPS (30ms delay); calls `exit(0)` on ESC/B button to terminate application

### ScenarioChooser::handle_event
- **Signature:** `void handle_event(SDL_Event& e)`
- **Purpose:** Process keyboard, mouse, and controller events; update selection and scroll state
- **Inputs:** SDL_Event from event queue
- **Outputs/Return:** None
- **Side effects:** Updates `selection_`, `done_`, `scroll_`, internal state; may call `exit(0)`
- **Calls:** `move_selection`, `ensure_selection_visible`, `joystick_added`, `joystick_removed`
- **Notes:** Handles arrow keys, mouse clicks, Tab/Shift-Tab, Page Up/Down, Return, joystick D-pad and buttons; mouse click on scenario grid sets selection and exits; B button / ESC calls `exit(0)`

### ScenarioChooser::redraw
- **Signature:** `void redraw(SDL_Window* window)`
- **Purpose:** Clear window and render all visible scenarios with selection highlight
- **Inputs:** SDL_Window pointer
- **Outputs/Return:** None
- **Side effects:** Modifies window surface, calls `SDL_UpdateWindowSurface`
- **Calls:** `SDL_GetWindowSurface`, `SDL_FillRect`, `SDL_BlitSurface`, `SDL_UpdateWindowSurface`
- **Notes:** Dark gray background (23,23,23); selected item gets light gray border; culls offscreen items

### ScenarioChooser::move_selection
- **Signature:** `void move_selection(int col_delta, int row_delta)`
- **Purpose:** Move selection by grid offset; handles wrapping and edge clamping
- **Inputs:** Column and row deltas
- **Outputs/Return:** None
- **Side effects:** Updates `selection_`, calls `ensure_selection_visible`
- **Calls:** `ensure_selection_visible`
- **Notes:** If no selection, arrow down/right selects first; arrow up/left from no-selection wraps to last; clamps to grid bounds

### ScenarioChooser::determine_cols_rows
- **Signature:** `void determine_cols_rows()`
- **Purpose:** Calculate grid layout: number of columns, rows, offsets, and max scroll distance
- **Inputs:** `window_width_`, `window_height_`, `scenarios_.size()`
- **Outputs/Return:** None
- **Side effects:** Sets `cols_`, `rows_`, `offsetx_`, `offsety_`, `max_scroll_`
- **Calls:** None
- **Notes:** Centers grid horizontally; allows vertical scrolling if content exceeds window height

### ScenarioChooser::ensure_selection_visible
- **Signature:** `void ensure_selection_visible()`
- **Purpose:** Scroll viewport to keep selected scenario in center-ish view
- **Inputs:** `selection_`, current `scroll_`
- **Outputs/Return:** None
- **Side effects:** Updates `scroll_` within `[0, max_scroll_]`
- **Calls:** None
- **Notes:** Avoids over-scrolling at edges

### ScenarioChooser::optimize_image
- **Signature:** `void optimize_image(ScenarioChooserScenario& scenario, SDL_Window* window)`
- **Purpose:** Resize and format scenario thumbnail to display size (320├ù240) and native window surface format
- **Inputs:** Scenario reference, SDL_Window pointer
- **Outputs/Return:** None
- **Side effects:** Replaces scenario image with formatted/resized version
- **Calls:** `SDL_GetWindowSurface`, `SDL_ConvertSurface`, `SDL_CreateRGBSurface`, `SDL_FillRect`, `SDL_SoftStretchLinear` / `SDL_BlitScaled`
- **Notes:** Uses linear scaling if available (SDL 2.0.16+), falls back to blit-scaled; preserves aspect ratio with black letterboxing

## Control Flow Notes
**Initialization ΓåÆ Presentation Loop ΓåÆ Shutdown**

1. `run()` is invoked after scenarios are added
2. Window created fullscreen at desktop resolution
3. Images optimized to 320├ù240 target size
4. Event loop processes input, redraws at ~30 FPS until selection or ESC
5. Returns selected path and workshop flag; caller likely transitions to scenario initialization

## External Dependencies
- **SDL2** (window, event, surface rendering)
- **SDL_image** (conditional; image format loading if `HAVE_SDL_IMAGE` defined)
- **Boost** (`algorithm::string`, `property_tree` XML parsing)
- **cseries.h** ΓÇö DirectorySpecifier, FileSpecifier, dir_entry, OpenedFile
- **find_files.h** ΓÇö FileFinder base class
- **images.h** ΓÇö `find_title_screen()`, `find_m1_title_screen()` (extract images from scenario resources)
- **sdl_fonts.h** ΓÇö font_info (included but not used in this file)
- **joystick.h** ΓÇö `joystick_added()`, `joystick_removed()` (plug/unplug callbacks)
