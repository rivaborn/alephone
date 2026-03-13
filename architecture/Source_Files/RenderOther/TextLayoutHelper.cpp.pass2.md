# Source_Files/RenderOther/TextLayoutHelper.cpp - Enhanced Analysis

## Architectural Role

TextLayoutHelper implements a **sweep-line algorithm** for conflict-free 2D UI element placement, serving the screen composition and HUD rendering subsystems. When rendering overlapping text labels or help messages on the HUD (e.g., coordinate displays, objective text, terminal output), this helper prevents visual occlusion by computing safe vertical placement given horizontal bounds. It's called during HUD composition phases before rasterization, not per-frame but per layout event (terminal opens, help triggered, objectives change).

## Key Cross-References

### Incoming (who depends on this file)
- **RenderOther subsystem** (`screen_drawing.cpp`, `computer_interface.cpp`, overhead_map, motion_sensor) ΓÇö likely calls `reserveSpaceFor()` to position UI text labels without overlap
- **HUD/Terminal rendering** ΓÇö computer interface draws multi-line text; TextLayoutHelper prevents label stack-up

### Outgoing (what this file depends on)
- **CSeries platform layer** ΓÇö `<assert.h>` for runtime safety checks
- **C++ STL** ΓÇö `std::vector` (inherited via header), `std::multiset` for active-reservation tracking
- No game world, rendering backend, or input subsystem dependencies; purely geometric computation

## Design Patterns & Rationale

**Sweep-line algorithm** with event-driven endpoint tracking:
- Horizontal coordinates (`mHorizontalCoordinate`) mark rectangle left/right boundaries
- `mReservationEnds` vector is implicitly sorted by x-coordinate, allowing linear scan
- `mStartOfReservation` flag distinguishes opening vs. closing events without separate data structures
- `CollectionOfReservationPointers` (multiset) maintains active reservations overlapping the x-range in O(log n) insertion/erasure

**Why this structure?** 
- Avoids quadratic pre-computation; scales with number of active overlaps per query, not total count
- Reuse-friendly: can reserve many rectangles without clearing state (for streaming HUD updates)
- Simple and self-contained with no external dependency on RenderMain or GameWorld

**Tradeoff:** O(n┬▓) worst-case vertical conflict resolution (re-walk the overlap set after each y-adjustment) is intentional ΓÇö n is UI element count (typically 2ΓÇô10), so the cost is negligible. More sophisticated algorithms (segment trees, interval trees) would add complexity not justified by typical UI density.

## Data Flow Through This File

1. **Input:** Rectangle dimensions (left, width, min_bottom, height) from caller (HUD compositor)
2. **Scan Phase 1:** Walk `mReservationEnds` left-to-right; collect all active (open) reservations at/before the new rectangle's left edge
3. **Insert left endpoint:** Place new reservation's start marker in sorted order
4. **Scan Phase 2:** Continue walk; collect all reservation opens before the new rectangle's right edge  
5. **Insert right endpoint:** Place new reservation's end marker
6. **Conflict loop:** Iterate collected overlapping reservations; if any vertically interfere with current y-position, bump y up and re-check entire list (ensuring cascade-free final position)
7. **Output:** Store final vertical bounds in Reservation struct; return bottom coordinate to caller
8. **Lifetime:** Reservation pointers persist in `mReservationEnds` until `removeAllReservations()` is called (e.g., scene transition, HUD clear)

## Learning Notes

- **Sweep-line idiom:** This file is a textbook example of the sweep-line computational geometry pattern, teaching how to avoid O(n┬▓) rectangle overlap queries by maintaining invariants (sorted events, active set)
- **Pre-STL era conventions:** No smart pointers (`std::unique_ptr`); manual `new`/`delete` with explicit lifecycle (destructor cleanup). This reflects early 2000s C++ practices
- **Assertion patterns:** Type-check assertions on `inHeight` (ensuring `unsigned int` is losslessly cast to/from `int`) ΓÇö defensive against integer overflow bugs in caller
- **Screen coordinate system:** Y-axis points downward (`inLowestBottom` is a y-coordinate; decreasing y is "up" visually); inHeight subtracted from bottom yields top
- **Idiomatic difference from modern engines:** Most modern engines (Unity, Unreal, Godot) use UI layout systems with constraint solvers (flexbox-like) or hierarchical relative positioning; this flat reservation system is designed for ad-hoc HUD labels (in-world distances, kill counters) that don't form a coherent hierarchy

## Potential Issues

1. **Unbounded reservation accumulation:** If `removeAllReservations()` is never called (e.g., long-running session with many ephemeral UI elements), `mReservationEnds` vector grows without bound, and O(n) scans become expensive. No maximum size check or LRU eviction.

2. **Integer underflow risk:** `theCurrentBottom - static_cast<int>(inHeight)` in the conflict check assumes caller ensures `inHeight <= theCurrentBottom`; no guard against negative y or out-of-bounds placement. If inHeight > inLowestBottom, result can be far offscreen.

3. **Assertion on every call:** The repeated `assert(inHeight == ...)` inside the conflict loop (executed up to n times per call) may impact performance in debug builds; could be moved outside the loop or use a one-time validation at function entry.

4. **Memory leak path:** If `reserveSpaceFor()` throws (e.g., during multiset insertion), the allocated Reservation is leaked. No RAII or exception safety guarantees in pre-C++11 style.
