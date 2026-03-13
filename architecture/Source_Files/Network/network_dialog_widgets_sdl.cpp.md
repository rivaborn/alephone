# Source_Files/Network/network_dialog_widgets_sdl.cpp

## File Purpose
Implements SDL-based UI widgets for network-related dialogs in Aleph One. Provides specialized widgets for discovering network players during gathering, displaying in-game player rosters, rendering postgame carnage reports with kill/death statistics, and selecting game levels by type.

## Core Responsibilities
- Manage and display lists of prospective joiners discovered during network gathering
- Render player avatars, names, and dynamic status indicators
- Calculate and visualize kill/death/score bar graphs with scaling and collision avoidance
- Support both individual and team-based player grouping layouts
- Handle level/entry-point selection filtered by game type
- Process user interactions (clicks, keyboard navigation) on network UI elements

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `w_found_players` | class | List widget displaying prospective network joiners |
| `w_players_in_game2` | class | Complex widget for in-game rosters and postgame carnage reports |
| `w_entry_point_selector` | class | Selection widget for game levels filtered by type |
| `player_entry2` | struct | Caches player display data (name, color, rendered image) |
| `bar_info` | struct | Layout coordinates and styling for kill/death/score bar labels |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kWPIG2Width`, `kWPIG2Height` | enum | static | Standard widget dimensions (600├ù142 pixels) |
| `kMaxHeadroom`, `kNameOffset`, `kBarBottomOffset` | enum | static | Vertical layout offsets for player icons, names, and bars |
| `kPostgameHeight`, `kPostgamePlayerOffset` | enum | static | Postgame layout dimensions (larger vertical space) |
| `kUseLegendThreshhold` | enum | static | Threshold (5 players) to switch from full labels to legend mode |

## Key Functions / Methods

### w_found_players::found_player
- **Signature:** `void found_player(prospective_joiner_info &player)`
- **Purpose:** Register discovery of a new network player and add to display list.
- **Inputs:** Player info struct (stream ID, name, color, team, gathering flag).
- **Outputs/Return:** None (modifies internal vectors).
- **Side effects:** Appends to `found_players`, calls `list_player()`.
- **Calls:** `list_player()`
- **Notes:** Assumes caller has validated the player.

### w_found_players::unlist_player
- **Signature:** `void unlist_player(const prospective_joiner_info &player)`
- **Purpose:** Remove player from display list; adjust viewport if needed to prevent overhang.
- **Inputs:** Player info to match and remove.
- **Outputs/Return:** None (modifies `listed_players`, adjusts `top_item`/`num_items`).
- **Side effects:** Erases element from vector; calls `new_items()` and `set_top_item()`.
- **Calls:** `std::find()`, `std::distance()`, `new_items()`, `set_top_item()`.
- **Notes:** Uses operator== on prospective_joiner_info (compares stream_id); handles scroll offset carefully if deleted item was at/before current top.

### w_found_players::draw_item
- **Signature:** `void draw_item(vector<prospective_joiner_info>::const_iterator i, SDL_Surface *s, int16 x, int16 y, uint16 width, bool selected) const`
- **Purpose:** Render a single player entry in the list (name, possibly with "(gathering)" suffix).
- **Inputs:** Iterator to player, SDL surface, position, width, selection state.
- **Outputs/Return:** Draws text centered horizontally; returns void.
- **Side effects:** Modifies SDL surface pixels.
- **Calls:** `draw_text()`, `text_width()`, `get_theme_color()`.
- **Notes:** Disables text (grayed) if player is in gathering mode.

### w_players_in_game2::update_display
- **Signature:** `void update_display(bool inFromDynamicWorld = false)`
- **Purpose:** Refresh player roster from either live game world or network topology; create player images.
- **Inputs:** Boolean flag (true = fetch from dynamic_world; false = from network topology).
- **Outputs/Return:** None (populates `player_entries`, `players_on_team` arrays).
- **Side effects:** Clears internal storage; allocates new PlayerImage objects; reads global `dynamic_world` or calls `NetGetPlayerData()`.
- **Calls:** `clear_vector()`, `get_player_data()`, `NetGetPlayerData()`, `text_width()`, `get_dialog_player_color()`, `PlayerImage::setRandomFlatteringView()`, `PlayerImage::setPlayerColor()`, `PlayerImage::setTeamColor()`.
- **Notes:** Copies player names as C-strings; stores indices in per-team vectors for grouped rendering.

### w_players_in_game2::draw
- **Signature:** `void draw(SDL_Surface* s) const`
- **Purpose:** Main rendering entry point; orchestrates all visual elements (icons, names, bars, stats, legend).
- **Inputs:** SDL surface to draw to.
- **Outputs/Return:** Modifies surface pixels; sets clip rectangle.
- **Side effects:** Global I/O (drawing); clip rectangle set and reset.
- **Calls:** `set_drawing_clip_rectangle()`, `draw_player_icons_*()`, `draw_bars_*()`, `draw_player_names_*()`, `draw_carnage_totals()`, `draw_carnage_legend()`, `draw_bar_labels()`.
- **Notes:** Render order is back-to-front (icons ΓåÆ bars ΓåÆ names ΓåÆ labels); uses TextLayoutHelper to avoid overlapping text; respects `clump_players_by_team`, `draw_carnage_graph`, and `draw_scores_not_carnage` flags.

### w_players_in_game2::draw_player_icons_clumped
- **Signature:** `void draw_player_icons_clumped(SDL_Surface* s) const`
- **Purpose:** Draw avatars grouped and spaced by team.
- **Inputs:** SDL surface.
- **Outputs/Return:** Modifies surface (draws images).
- **Side effects:** Adjusts brightness on PlayerImage objects based on `selected_player`; may cache in image object.
- **Calls:** `get_wide_spaced_left_offset()`, `get_close_spaced_center_offset()`, `PlayerImage::setBrightness()`, `PlayerImage::drawAt()`.
- **Notes:** Outer loop by team (via net_rankings), inner by player (via players_on_team[color]); selected player highlighted at full brightness, others dimmed to 0.4.

### w_players_in_game2::find_maximum_bar_value
- **Signature:** `int find_maximum_bar_value() const`
- **Purpose:** Compute max value for bar scaling; handles mixed positive/negative scores (e.g., "Tag" mode).
- **Inputs:** None (reads `net_rankings`, `draw_scores_not_carnage`, `selected_player`).
- **Outputs/Return:** Integer maximum value for bar scaling.
- **Side effects:** None.
- **Calls:** `calculate_max_kills()`.
- **Notes:** If all nonpositive with negatives present, returns min value; if mixed positive/negative, returns max absolute value to center bars symmetrically.

### w_players_in_game2::draw_bar_or_bars
- **Signature:** `void draw_bar_or_bars(SDL_Surface* s, size_t rank_index, int center_x, int maximum_value, vector<bar_info>& outBarInfos) const`
- **Purpose:** Render one or more bars (score, kills, deaths) for a player and queue label placement.
- **Inputs:** Surface, player rank index, horizontal center, scaling max, output vector.
- **Outputs/Return:** Calls `draw_bar()`; appends bar_info (center, top_y, color, label) to output vector.
- **Side effects:** Modifies surface and vector; may generate sprintf labels into static `temporary` buffer.
- **Calls:** `draw_bar()`, `TS_GetCString()`, `sprintf()`.
- **Notes:** For scores mode, draws single bar; for carnage, draws kills/deaths separately (or suicides if selected player). Omits zero-value bars from label queue.

### w_players_in_game2::draw_bar
- **Signature:** `void draw_bar(SDL_Surface* s, int inCenterX, int inBarColorIndex, int inBarValue, int inMaxValue, bar_info& outBarInfo) const`
- **Purpose:** Render a single vertical bar with 3D bevel effect; populate bar_info for label placement.
- **Inputs:** Surface, center X, color index, value, max value.
- **Outputs/Return:** Modifies surface and outBarInfo struct (center_x, top_y, pixel_color populated).
- **Side effects:** Calls `SDL_FillRect()` three times (main, dark edge, middle shade); uses `get_net_color()`.
- **Calls:** `get_net_color()`, `SDL_MapRGB()`, `SDL_FillRect()`.
- **Notes:** Skips rendering if inBarValue == 0; computes bar height as `(theMaximumBarHeight * inBarValue) / inMaxValue`; three color shades create beveled 3D look.

### w_entry_point_selector::validateEntryPoint
- **Signature:** `void validateEntryPoint()`
- **Purpose:** Look up entry points for current game type; adjust or reset selection if invalid.
- **Inputs:** None (reads `mGameType`, `mEntryPoint.level_number`, writes `mEntryPoints`).
- **Outputs/Return:** Populates `mEntryPoints` vector; updates `mEntryPoint` and `mCurrentIndex`; sets `dirty = true`.
- **Side effects:** Clears and rebuilds entry-point vector; modifies member state.
- **Calls:** `get_entry_point_flags_for_game_type()`, `get_entry_points()`.
- **Notes:** If no valid entry points, sets level_number to NONE and name to "(no valid options)"; otherwise finds matching level or defaults to first.

### w_entry_point_selector::event
- **Signature:** `void event(SDL_Event &e)`
- **Purpose:** Handle keyboard input (left/right arrows) to cycle through available levels.
- **Inputs:** SDL event.
- **Outputs/Return:** Modifies event type to SDL_LASTEVENT if consumed; updates `mCurrentIndex`, `mEntryPoint`, sets `dirty = true`.
- **Side effects:** Advances/cycles selection; calls `play_dialog_sound()`.
- **Calls:** `play_dialog_sound()`.
- **Notes:** Modulo wraps index within `theNumberOfEntryPoints`; consumes event by setting type to SDL_LASTEVENT.

### w_entry_point_selector::gotSelected
- **Signature:** `void gotSelected()`
- **Purpose:** Pop up a selection dialog (if multiple options) to let user manually choose a level.
- **Inputs:** None; reads `mEntryPoints`, `mCurrentIndex`, file path from `environment_preferences`.
- **Outputs/Return:** None (modal dialog); updates `mCurrentIndex` and `mEntryPoint` if user approves.
- **Side effects:** Calls `clear_screen()`, creates widgets, runs dialog; may update game state.
- **Calls:** `dialog::run()`, various widget constructors, `FileSpecifier::GetName()`, `TS_GetCString()`, `sprintf()`.
- **Notes:** Only shows dialog if more than one option; dialog is text-based (w_levels, w_button, etc.).

### Helper Functions (Inline/Static)

**Spacing Calculation Functions:**
- `get_wide_spaced_center_offset()`: Centers N items across width via `(2i+1)*W / 2N`.
- `get_wide_spaced_left_offset()`: Left edge of i-th item via `i*W / N`.
- `get_close_spaced_center_offset()`: Centers N items with padding via `(i+1)*W / (N+1)`.
- `get_close_spaced_width()`, `get_wide_spaced_width()`: Item width calculations.

**Macro:**
- `get_player_y_offset()`: Returns vertical offset based on `postgame_layout` flag.
- `get_name_y_offset()`: Name vertical offset (player offset + name offset).

## Control Flow Notes

**Initialization & Display Lifecycle:**
1. **Gathering Phase:** `w_found_players` added to dialog; `found_player()` called as SSLP reports discoveries.
2. **In-Game Display:** `w_players_in_game2::update_display()` called to populate roster from network or game world; `draw()` invoked per frame.
3. **Postgame:** `set_graph_data()` enables graph mode; carnage report renders sorted by kills/deaths/scores with bars and legend.
4. **Level Selection:** `w_entry_point_selector` listens for game type changes via `setGameType()`, validates entry point, and allows manual selection via dialog or keyboard navigation.

**Rendering Pipeline (w_players_in_game2):**
- Back-to-front: player icons ΓåÆ bars ΓåÆ player names (with TextLayoutHelper overlap avoidance) ΓåÆ carnage totals ΓåÆ legend ΓåÆ bar labels.
- If `clump_players_by_team`: wide spacing for teams, close spacing within teams.
- If separate: close spacing for all items.

**Resource Management:**
- `PlayerImage` objects allocated in `update_display()`, deallocated in `clear_vector()` destructor.
- Text layout managed via temporary TextLayoutHelper (stack-allocated, exists for draw duration).

## External Dependencies
- **sdl_widgets.h:** Base widget classes (w_list, w_select_button, widget, dialog).
- **SSLP_API.h:** Network service discovery definitions (prospective_joiner_info).
- **PlayerImage_sdl.h:** Avatar rendering (PlayerImage class).
- **network_dialogs.h:** Network structures (net_rank, entry_point).
- **screen_drawing.h:** Text and shape rendering (draw_text, text_width, set_drawing_clip_rectangle, SDL_FillRect).
- **sdl_fonts.h:** Font objects and metrics (font_info, get_ascent, get_line_height).
- **TextLayoutHelper.h:** Text collision avoidance.
- **TextStrings.h:** Localized string lookups (TS_GetCString).
- **player.h, HUDRenderer.h, shell.h, interface.h, network.h:** Game world and network APIs (dynamic_world, NetGetPlayerData, get_dialog_player_color, get_net_color).
- **std::vector, std::string, SDL2, cstdio, cstring:** Standard libraries.
