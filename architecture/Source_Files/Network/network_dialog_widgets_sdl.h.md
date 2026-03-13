# Source_Files/Network/network_dialog_widgets_sdl.h

## File Purpose
Declares custom SDL widget classes for network game dialogs, including player discovery display, game roster visualization, and level selection for networked multiplayer games.

## Core Responsibilities
- **Player discovery & network list**: `w_found_players` manages SSLP callback integration to display available joinable players
- **Game roster & postgame reporting**: `w_players_in_game2` renders current game participants or postgame carnage statistics with bars, icons, and rankings
- **Level selection**: `w_entry_point_selector` filters available map levels by game type compatibility

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `player_entry2` | struct | Cached display data per player (name, color, image surface) |
| `w_found_players` | class | List widget displaying discovered network players |
| `w_players_in_game2` | class | Roster/stats widget with dual layout modes (gather vs. postgame) |
| `w_entry_point_selector` | class | Dropdown-style selector for filtered map entry points |
| `player_selected_callback_t` | typedef | `void (*)(w_found_players*, prospective_joiner_info)` callback |
| `element_clicked_callback_t` | typedef | Complex callback for postgame graph clicks (player/team, team/player toggle, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### w_found_players

**Constructor**
- Signature: `w_found_players(int width, int numRows)`
- Purpose: Initialize list widget for discovered players; inherits from `w_list<prospective_joiner_info>`
- Initializes `num_items = 0` to sync with empty `listed_players` vector
- Side effects: Sets up empty `found_players`, `hidden_players`, `listed_players` storage

**found_player**
- Signature: `void found_player(prospective_joiner_info &player)`
- Purpose: Add newly discovered network player to display list
- Calls private `list_player()` to render update

**update_player**
- Signature: `void update_player(prospective_joiner_info &player)`
- Purpose: Update existing player's information (e.g., name/status changed)

**hide_player**
- Signature: `void hide_player(const prospective_joiner_info &player)`
- Purpose: Remove player from visible list (e.g., after joining)
- Calls private `unlist_player()`

**set_player_selected_callback**
- Signature: `void set_player_selected_callback(player_selected_callback_t callback)`
- Purpose: Register callback invoked when user clicks a player entry
- Side effects: Stores callback function pointer for `item_selected()` dispatch

**callback_on_all_items**
- Signature: `void callback_on_all_items()`
- Purpose: Invoke registered callback for each listed player (used for joining)
- Notes: Comment warns callback must remove item from listΓÇöfragile design but acceptable for this use

---

### w_players_in_game2

**Constructor**
- Signature: `w_players_in_game2(bool inPostgameLayout)`
- Purpose: Create roster widget; supports both gather-dialog and postgame-report layouts
- Inputs: `inPostgameLayout` ΓÇô true for tall postgame carnage display
- Side effects: Initializes `displaying_actual_information = false`, empty player storage

**update_display**
- Signature: `void update_display(bool inFromDynamicWorld = false)`
- Purpose: Refresh player list and stats from world or static topology
- Inputs: `inFromDynamicWorld` ΓÇô if true, read from postgame dynamic state; else from topology
- Side effects: May rebuild `player_entries`, `net_rankings`, team vectors

**start_displaying_actual_information**
- Signature: `void start_displaying_actual_information()`
- Purpose: Enable real data rendering (must be called once topology data is valid)
- Sets `displaying_actual_information = true`

**draw**
- Signature: `virtual void draw(SDL_Surface *s) const`
- Purpose: Render player icons/names and bars/scores based on layout mode
- Calls various private drawing helpers: `draw_player_icons_*()`, `draw_bars_*()`, `draw_carnage_totals()`, etc.

**click**
- Signature: `virtual void click(int x, int y)`
- Purpose: Handle user click; invoke `element_clicked_callback` if user clicked near a player icon
- Notes: Currently callback only fires in postgame layout despite appearances

**set_graph_data**
- Signature: `void set_graph_data(const net_rank* inRankings, int inNumRankings, int inSelectedPlayer, bool inClumpPlayersByTeam, bool inDrawScoresNotCarnage)`
- Purpose: Populate postgame stats and layout parameters
- Inputs: Rankings array, clumping mode, score-vs-carnage flag
- Side effects: Stores rankings, selected player index, team clumping preference

**set_element_clicked_callback**
- Signature: `void set_element_clicked_callback(element_clicked_callback_t inCallback)`
- Purpose: Register callback for postgame graph interaction

**draw_bar** (private helper)
- Signature: `void draw_bar(SDL_Surface *s, int inCenterX, int inBarColorIndex, int inBarValue, int inMaxValue, bar_info& outBarInfo) const`
- Purpose: Render single kill/score bar; return bar bounds via `outBarInfo` for click detection
- Notes: Key for element click handling

---

### w_entry_point_selector

**Constructor**
- Signature: `w_entry_point_selector(size_t inGameType, int16 inLevelNumber)`
- Purpose: Create level selector tied to game type; inherits from `w_select_button`
- Inputs: Game type, initial level number
- Side effects: Calls `setGameType()` to load compatible entry points for display

**setGameType**
- Signature: `void setGameType(size_t inGameType)`
- Purpose: Change game type and revalidate current level selection
- Calls `validateEntryPoint()` if game type actually changed
- Side effects: May adjust level to first available if current incompatible

**reset**
- Signature: `void reset()`
- Purpose: Revert to first available entry point for current game type (e.g., after map file change)
- Side effects: Sets `mEntryPoint.level_number = NONE`, invokes validation

**setLevelNumber**
- Signature: `void setLevelNumber(int16 inLevelNumber)`
- Purpose: Attempt to select specific level number
- Calls `validateEntryPoint()` to confirm compatibility

**getEntryPoint**
- Signature: `const entry_point& getEntryPoint()`
- Purpose: Return currently-selected entry point data
- Outputs: Const reference to `mEntryPoint`

**event**
- Signature: `virtual void event(SDL_Event& e)`
- Purpose: Handle left/right cursor navigation to cycle through compatible levels

**validateEntryPoint** (private)
- Signature: `void validateEntryPoint()`
- Purpose: Revalidate entry point against current game type; select matching level or first available
- Side effects: Updates `mEntryPoint`, `mEntryPoints` vector, refreshes button display label
- Notes: If no entry points available, sets `level_number = NONE`

## Control Flow Notes
These widgets are embedded in SDL dialog flows. `w_found_players` responds to SSLP network discovery callbacks. `w_players_in_game2` updates from game state during gather/postgame phases. `w_entry_point_selector` intercepts game-type changes and revalidates/updates level availability in real time.

## External Dependencies
- **sdl_widgets.h**: Base `widget`, `w_list<T>`, `w_select_button` classes
- **SSLP_API.h**: `SSLP_ServiceInstance` (unused here; SSLP integration in implementation)
- **player.h**: `MAXIMUM_PLAYER_NAME_LENGTH` constant; player team/color enums
- **PlayerImage_sdl.h**: `PlayerImage` class for rendering player sprites
- **network_dialogs.h**: `net_rank` struct (postgame rankings), `entry_point` (defined in map.h)
- Standard: `<vector>` (STL containers for player/entry-point lists)
