# Source_Files/RenderOther/TextLayoutHelper.h

## File Purpose
Utility class for managing non-overlapping rectangular regions in a 2D layout. Used by the Marathon: Aleph One engine to position UI elements (likely text boxes or other render components) without collisions.

## Core Responsibilities
- Track horizontal and vertical boundaries of placed rectangles
- Query for the lowest available vertical position at a given horizontal range
- Manage and clear all active reservations
- Calculate adjusted placement positions to avoid overlaps

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ReservationEnd` | struct | Marks a horizontal coordinate where a reservation starts or ends; points to the reservation it belongs to |
| `Reservation` | struct | Stores the vertical extent (top and bottom) of a reserved region |

## Global / File-Static State
None.

## Key Functions / Methods

### TextLayoutHelper (constructor)
- Signature: `TextLayoutHelper()`
- Purpose: Initialize an empty layout manager
- Inputs: None
- Outputs/Return: Instance ready to accept reservations
- Side effects: Allocates internal vector

### ~TextLayoutHelper (destructor)
- Signature: `~TextLayoutHelper()`
- Purpose: Clean up resources
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates reservation tracking data

### removeAllReservations
- Signature: `void removeAllReservations()`
- Purpose: Clear all existing reservations
- Inputs: None
- Outputs/Return: None
- Side effects: Empties `mReservationEnds` vector

### reserveSpaceFor
- Signature: `int reserveSpaceFor(int inLeft, unsigned int inWidth, int inLowestBottom, unsigned int inHeight)`
- Purpose: Request space for a rectangle; returns the actual bottom position that avoids overlaps
- Inputs: Horizontal left edge, width, minimum desired bottom, height
- Outputs/Return: Adjusted bottom coordinate (ΓëÑ `inLowestBottom`)
- Side effects: Adds a new reservation to `mReservationEnds`
- Notes: Core algorithmΓÇösweeps horizontal boundaries to find the first vertical gap that accommodates the requested dimensions

## Control Flow Notes
Typical usage: game/UI rendering loops call `reserveSpaceFor()` iteratively to position elements top-to-bottom or in arbitrary order. After a frame, `removeAllReservations()` resets state for the next layout pass.

## External Dependencies
- `<vector>` (STL for storing ReservationEnd list)
- `std::vector`
