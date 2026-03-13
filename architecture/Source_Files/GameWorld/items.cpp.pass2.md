# Source_Files/GameWorld/items.cpp - Enhanced Analysis

## Architectural Role

Items.cpp acts as a **configuration-driven runtime interpreter** bridging static MML-defined item properties (`item_definitions[]`) with dynamic world state (objects, player inventory). It's not merely a containerΓÇöit actively orchestrates three key flows: **item lifecycle** (creation with game-mode filtering), **player acquisition** (multi-stage pickup validation via spatial + line-of-sight checks), and **animation coordination** (per-frame shape updates with lazy randomization). The file also serves as a **compatibility shim**, exposing both Marathon 1 and Marathon 2 swiping behaviors via a dispatcher pattern controlled by `film_profile.swipe_nearby_items_fix`.

## Key Cross-References

### Incoming (who depends on this file)
- **marathon2.cpp** calls `animate_items()` every game tick (30 FPS frame update loop)
- **marathon2.cpp** calls `swipe_nearby_items()` when player enters a polygon (collision callback)
- **placement.cpp** calls `new_item()` during level load to instantiate items from map data
- **Lua VM** (lua_script.h) receives item creation/pickup events via `L_Call_Item_Created()` and `L_Call_Got_Item()` hooks
- **map.h** object lists are scanned by `trigger_nearby_items()` during flood-fill activation

### Outgoing (what this file depends on)
- **map.h**: `new_map_object()`, `get_object_data()`, `remove_map_object()`, `get_polygon_data()`, `get_line_data()` for object lifecycle and spatial queries
- **player.h**: `get_player_data()`, inventory array updates, `mark_player_inventory_as_dirty()` for collection
- **weapons.h**: `process_new_item_for_reloading()` integrates ammo pickups with active weapon state
- **SoundManager.h**: `PlaySound()` for pickup audio, ball acquisition sound trigger
- **fades.h**: `start_fade()` initiates screen flash when local player collects item
- **flood_map.h**: `flood_map()` used by `trigger_nearby_items()` to discover reachable polygons
- **FilmProfile.h**: `film_profile.swipe_nearby_items_fix` flag selects compatibility implementation

## Design Patterns & Rationale

**Configuration-Driven Entity Behavior**: Item types are immutable post-load; `get_item_definition()` accessors enforce bounds checking, preventing scripting exploits. MML parsing backs up originals in `original_item_definitions` before overwriting, allowing resetΓÇöthough the design choice to leak memory if `reset_mml_items()` is called twice (missing null check on original copy) suggests this was a "load once, run forever" pattern.

**Two-Stage Pickup Validation**: `swipe_nearby_items()` ΓåÆ `get_item()` ΓåÆ `try_and_add_player_item()` enforces separation of concerns: first stage checks 2D proximity + polygon traversal, second stage validates line-of-sight via `test_item_retrieval()` (raycast-like polygon/line crossing), third stage checks inventory limits and triggers side effects (weapon reload, powerup processing, Lua callbacks). This layering prevents expensive raycasts on distant items.

**Dispatcher via Feature Flag**: `swipe_nearby_items()` delegates to either `a1_swipe_nearby_items()` (Marathon 1 fix: searches current + neighbors with kludgy -1 loop start) or `m2_swipe_nearby_items()` (Marathon 2 original: direct neighbor iteration). This pattern mirrors how Aleph One handles engine variant incompatibilitiesΓÇöconfiguration decouples runtime behavior without recompilation.

**Lazy Randomization Flag Reuse**: `object->facing >= 0` serves as a "not yet randomized" sentinel for unanimated items; set to `NONE` after first `randomize_object_sequence()` call. This repurposing of a geometric field (facing direction) saves memory but is fragileΓÇöassumes facing is unused for items, and the comment indicates this caused confusion during development ("polarity reversal").

## Data Flow Through This File

**Item Creation Path:**
```
new_item(location, type) 
  ΓåÆ get_item_definition(type) [bounds check]
  ΓåÆ Check environment flags vs. dynamic_world->player_count
  ΓåÆ new_map_object() ΓåÆ object_data [owner=_object_is_item, permutation=type]
  ΓåÆ object_was_just_added() [notify placement.cpp]
  ΓåÆ L_Call_Item_Created() [Lua hook]
  ΓåÆ SET_OBJECT_INVISIBILITY() if network-only in wrong mode
```

**Pickup Path:**
```
swipe_nearby_items(player_idx)
  ΓåÆ Iterate neighbor polygons (flood-fill or simple loop)
  ΓåÆ For each visible item within MAXIMUM_ARM_REACH:
    ΓåÆ test_item_retrieval() [raycast polygon/line checks + platform obstruction]
    ΓåÆ get_item(player_idx, object_idx)
      ΓåÆ try_and_add_player_item()
        ΓåÆ legal_player_powerup() / process_player_powerup() [special handling]
        ΓåÆ Increment player->items[type] (respecting max_count + difficulty-specific caps)
        ΓåÆ process_new_item_for_reloading() [weapon integration]
        ΓåÆ Sound_GotItem() / start_fade() [audio/visual feedback]
        ΓåÆ L_Call_Got_Item() [Lua hook]
      ΓåÆ remove_map_object() [delete from world]
```

**Animation Path:**
```
animate_items()
  ΓåÆ For each object where owner == _object_is_item:
    ΓåÆ If facing >= 0 [not randomized]: randomize_object_sequence()
    ΓåÆ Set facing = NONE [mark done]
    ΓåÆ animate_object() [advance frame/sequence]
```

## Learning Notes

- **Era-specific hacks**: Using `object->facing` as a randomization flag (instead of a dedicated flag field) reflects memory-constrained design of early 1990s engines. Modern engines would use explicit `bool randomized` or a state enum.
- **Dual implementations for compatibility**: The A1 vs M2 swiping code exists because Marathon 1's neighbor flood-fill had subtle bugs (`precalculate_map_indexes()` and `intersecting_flood_proc()` cited in comments), so the A1 fix iterates manually from -1 to include current polygon. This workaround persists rather than fixing the root causeΓÇötypical of legacy codebases balancing correctness vs. risk.
- **Network-only items stay in memory**: Items marked `_environment_network` are made invisible but never removed if spawned in single-player, wasting object slots. A more efficient design would prevent creation entirely.
- **Ball special-casing**: Ball items trigger `static_world->ball_in_play` flag and play a sound on first creationΓÇömixing game logic with item data. CTF/Rugby ball pickup is handled in `try_and_add_player_item()` with explicit polygon checks for base zones, showing awareness of game mode but scattered across files.

## Potential Issues

- **Hardcoded MAXIMUM_ARM_REACH** (3├ùWORLD_ONE_FOURTH) tightly couples swiping behavior; changing arm reach requires recompilation and invalidates existing level designs relying on this constant.
- **test_item_retrieval() checks moving platforms but only for obstruction at current frame**ΓÇöif a platform is moving toward the item, it could pass the check then collide mid-frame during actual item transfer.
- **MML reset leaks memory**: `reset_mml_items()` calls `free(original_item_definitions)` but doesn't set to NULL; calling twice accesses freed memory. The backup is allocated once on first parse via `malloc()`, but the pattern assumes single parseΓåÆreset cycle.
- **Network environment filtering at creation time only**: Items are hidden, not removed, if created in wrong game mode. If game mode changes mid-level, hidden items don't become visibleΓÇöintentional but surprising.
