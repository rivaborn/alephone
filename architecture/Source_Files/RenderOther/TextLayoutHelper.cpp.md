# Source_Files/RenderOther/TextLayoutHelper.cpp

## File Purpose
Implements a 2D rectangle reservation/layout system that prevents overlapping UI elements. Given a rectangle's horizontal bounds and a minimum vertical position, it finds the lowest non-overlapping vertical placement by consulting existing reservations.

## Core Responsibilities
- Track active rectangle reservations across horizontal space using sorted endpoints
- Compute vertical placement for new rectangles by detecting conflicts with existing ones
- Manage memory lifecycle of reservation metadata
- Support clearing all reservations for reuse

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Reservation` | struct | Stores vertical bounds (`mTop`, `mBottom`) of a reserved rectangle |
| `ReservationEnd` | struct | Marks a horizontal coordinate where a reservation starts or ends; links to its `Reservation` |
| `CollectionOfReservationEnds` | typedef | `vector<ReservationEnd>`; tracks all horizontal boundary events sorted by x-coordinate |
| `CollectionOfReservationPointers` | local typedef | `multiset<Reservation*>`; active reservations overlapping a given x-range |

## Global / File-Static State
None.

## Key Functions / Methods

### reserveSpaceFor
- **Signature:** `int reserveSpaceFor(int inLeft, unsigned int inWidth, int inLowestBottom, unsigned int inHeight)`
- **Purpose:** Place a new rectangle in reserved space and return its final vertical position.
- **Inputs:**
  - `inLeft`: left edge (x-coordinate)
  - `inWidth`: width in pixels
  - `inLowestBottom`: minimum allowed bottom (highest y-value, since y increases downward)
  - `inHeight`: height in pixels
- **Outputs/Return:** The bottom (y-coordinate) where the rectangle was placed to avoid conflicts.
- **Side effects:** Allocates a `Reservation` struct; inserts two `ReservationEnd` entries into `mReservationEnds`; updates reservation vertical bounds.
- **Calls:** `mReservationEnds.insert()`, `std::multiset` insert/erase, `new Reservation`.
- **Notes:**
  - Algorithm walks `mReservationEnds` twice: once to find overlapping reservations by x-coordinate, once to resolve vertical conflicts by iterating until no interference remains.
  - Uses a do-while loop to re-check the entire overlap set after each vertical adjustment, which is O(n┬▓) in worst case but simple.
  - Includes assertions on inHeight type safety but relies on caller to pass valid values.

### removeAllReservations
- Iterates `mReservationEnds`, deletes each `Reservation*` exactly once (identified by `mStartOfReservation` flag), clears vector.

### Constructor / Destructor
- Constructor: no-op.
- Destructor: calls `removeAllReservations()`.

## Control Flow Notes
Not part of main game loop. Called from UI/text rendering layers to position UI elements (e.g., help text, on-screen messages) without overlap. One-shot utility for layout computation per frame or update cycle.

## External Dependencies
- `#include <set>` ΓÇô for `std::multiset`
- `#include <assert.h>` ΓÇô runtime assertions
- `#include <vector>` (via header) ΓÇô `std::vector`
