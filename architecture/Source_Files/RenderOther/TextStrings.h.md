# Source_Files/RenderOther/TextStrings.h

## File Purpose
Header for a text string repository system that replaces legacy MacOS STR# resources. Provides a two-level lookup scheme (resource ID + index) for managing localized and game strings. Includes UTF-8 decoding and MML markup support.

## Core Responsibilities
- Store and retrieve C-strings organized by resource ID and index
- Add/delete individual strings or entire string sets
- Query string set presence and cardinality
- Decode UTF-8 strings to C format
- Parse and manage MML-formatted string data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| InfoTree | class (external) | Hierarchical configuration/markup tree for MML parsing |

## Global / File-Static State
None.

## Key Functions / Methods

### TS_PutCString
- Signature: `void TS_PutCString(short ID, short Index, const char *String)`
- Purpose: Store or replace a string in the repository
- Inputs: Resource ID, index within set, C-string pointer
- Outputs/Return: None
- Side effects: Modifies string repository; repeated calls with same (ID, Index) replace previous value
- Calls: Not visible in header
- Notes: Replaces deprecated MacOS resource API

### TS_GetCString
- Signature: `const char *TS_GetCString(short ID, short Index)`
- Purpose: Retrieve a stored string by ID and index
- Inputs: Resource ID, index within set
- Outputs/Return: Pointer to C-string; NULL if not found
- Side effects: None (read-only)
- Calls: Not visible in header
- Notes: Returns NULL for invalid ID/Index pairs

### TS_IsPresent
- Signature: `bool TS_IsPresent(short ID)`
- Purpose: Check if a string set with given ID exists
- Inputs: Resource ID
- Outputs/Return: Boolean presence indicator
- Side effects: None
- Calls: Not visible in header

### TS_CountStrings
- Signature: `size_t TS_CountStrings(short ID)`
- Purpose: Count contiguous strings in a set starting from index 0
- Inputs: Resource ID
- Outputs/Return: Count of strings
- Side effects: None
- Calls: Not visible in header

### DeUTF8_C
- Signature: `size_t DeUTF8_C(const char *InString, size_t InLen, char *OutString, size_t OutMaxLen)`
- Purpose: Decode UTF-8 encoded string to C string format
- Inputs: Input UTF-8 string, input length, output buffer, buffer capacity
- Outputs/Return: Character count written to output (excludes null terminator)
- Side effects: Writes to OutString buffer
- Calls: Not visible in header
- Notes: Requires at least (OutMaxLen + 1) characters allocated in output buffer

### parse_mml_stringset, reset_mml_stringset
- Signature: `void parse_mml_stringset(const InfoTree& root)`; `void reset_mml_stringset()`
- Purpose: Load/initialize strings from MML markup; clear all loaded MML strings
- Inputs: InfoTree configuration hierarchy (MML)
- Outputs/Return: None
- Side effects: Populates string repository; reset clears MML-sourced strings
- Calls: Not visible in header

**Trivial helpers:** `TS_DeleteString`, `TS_DeleteStringSet`, `TS_DeleteAllStrings` ΓÇö removal operations for individual strings, sets, or entire repository.

## Control Flow Notes
This module is likely initialized during engine startup via `parse_mml_stringset()`. String lookups occur during rendering/UI phases via `TS_GetCString()`. Not tied to frame loop; purely data-storage utility.

## External Dependencies
- `#include <stddef.h>` ΓÇö for `size_t` type
- `InfoTree` class ΓÇö defined elsewhere; represents hierarchical MML configuration data
- Called by render/UI systems expecting string IDs to map to localized content
