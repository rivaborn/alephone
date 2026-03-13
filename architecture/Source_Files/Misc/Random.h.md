# Source_Files/Misc/Random.h

## File Purpose
Provides a C++ struct encapsulating pseudorandom number generators (PRNGs) based on algorithms by George Marsaglia. Enables independent PRNG instances with separate state for game subsystems requiring reproducible or varied random sequences.

## Core Responsibilities
- Implement multiple pseudorandom number generation algorithms (KISS, MWC, LFIB4, SWB, SHR3, CONG)
- Maintain internal algorithmic state (seeds, lookup table, counters)
- Generate 32-bit unsigned integer random values
- Convert raw integers to normalized floats in (0,1) and (-1,1) ranges
- Support independent PRNG instances for different gameplay systems

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GM_Random` | struct | Container for PRNG state and algorithm implementations |

## Global / File-Static State
None.

## Key Functions / Methods

### KISS
- Signature: `uint32 KISS()`
- Purpose: High-quality combined generator ("Keep It Simple Stupid")
- Inputs: None
- Outputs/Return: 32-bit random value
- Side effects: Updates `z`, `w`, `jsr`, `jcong` via component generators
- Calls: `MWC()`, `CONG()`, `SHR3()`
- Notes: Primary recommended generator; combines three independent algorithms for strong statistical properties

### UNI
- Signature: `float UNI()`
- Purpose: Generate uniform random float in (0, 1)
- Inputs: None
- Outputs/Return: Float in (0, 1)
- Side effects: Calls `KISS()` (updates internal state)
- Calls: `KISS()`
- Notes: Scales 32-bit output by 2.328306e-10

### VNI
- Signature: `float VNI()`
- Purpose: Generate signed uniform random float in (-1, 1)
- Inputs: None
- Outputs/Return: Float in (-1, 1)
- Side effects: Calls `KISS()` (updates internal state)
- Calls: `KISS()` cast to `int32`
- Notes: Scales signed 32-bit output by 4.656613e-10

### SetTable
- Signature: `void SetTable()`
- Purpose: Initialize 256-entry lookup table with KISS-generated values
- Inputs: None
- Outputs/Return: None
- Side effects: Fills `t[256]`; calls `KISS()` 256 times
- Calls: `KISS()` (in loop)
- Notes: Called from constructor; required for LFIB4/SWB initialization

### GM_Random (constructor)
- Signature: `GM_Random()`
- Purpose: Initialize independent PRNG instance with default seed values
- Inputs: None
- Outputs/Return: Initialized struct
- Side effects: Sets state fields to hardcoded defaults; populates lookup table
- Calls: `SetTable()`
- Notes: Hard-coded seeds ensure reproducible sequences; no parameter-based seeding in header

## Notes
Component generators (`znew()`, `wnew()`, `MWC()`, `SHR3()`, `CONG()`, `LFIB4()`, `SWB()`) are internal helpers. Users choose one of KISS, MWC, LFIB4, SWB, SHR3, or CONG for raw 32-bit output, or UNI/VNI for normalized floats. Table-based generators (LFIB4, SWB) use circular counter `c` (unsigned char, auto-wraps at 256).

## Control Flow Notes
Utility component used on-demand by game subsystems. Not part of frame/update/render loop. Typical usage: instantiate once per consumer, call generator methods as needed. Multiple independent instances support separate PRNGs for enemies, effects, loot, etc.

## External Dependencies
- `cstypes.h`: Provides fixed-size integer typedefs (`uint32`, `int32`, `uint8`)
