# Source_Files/RenderOther/TextStrings.cpp - Enhanced Analysis

## Architectural Role

This file implements the runtime string repository subsystem that replaces Marathon's hardcoded MacOS STR# resources. It serves as a localization bridge: MML configuration (XML parsing via `InfoTree`) provides UTF-8 encoded strings, which are decoded to Mac Roman during engine initialization and cached globally for retrieval by UI/rendering code. The decoupling of string loading (startup) from string retrieval (runtime) is intentionalΓÇöUTF-8ΓåÆMac Roman conversion cost is paid once at init, not per lookup.

## Key Cross-References

### Incoming (who depends on this file)
- **RenderOther subsystem** (screen_drawing.cpp, computer_interface.cpp, overhead_map.cpp, etc.): Call `TS_GetCString()` to fetch localized UI strings for HUD, terminal text, menu labels
- **XML/MML system** (XML_MakeRoot.cpp): Calls `parse_mml_stringset()` during config loading phase to populate repository
- **Preferences/Shell**: Uses strings for interface labels, dialog text, status messages
- Callers assume NULL pointer on miss and handle gracefully (defensive null-check pattern)

### Outgoing (what this file depends on)
- **InfoTree** (XML/MML): `parse_mml_stringset()` depends on `InfoTree::read_attr()`, `children_named()`, `read_indexed()` for tree traversal
- **unicode_to_mac_roman()** (defined elsewhere in CSeries): UTF-8 decoded codepoints converted to Mac Roman bytes
- **CSeries types** (cseries.h): uint8, uint16, uint32, int16, size_t foundational types

## Design Patterns & Rationale

**Repository Pattern with Global State**
- Centralizes all string lookups into single `StringSetRoot` map
- Era-appropriate for 1990s codebase (predates dependency injection)
- Trade-off: simplicity + fast lookups vs. thread-safety and testability concerns

**Two-Level Indexing (ID ΓåÆ Index ΓåÆ String)**
- ID = string set (logically grouping related strings, e.g., all terminal messages = ID 5)
- Index = position within set (enables sparse arrays and semantic grouping)
- Mirrors Marathon's original STR# "resource ID ΓåÆ index" scheme, easing asset conversion

**Lazy Null Return vs. Exceptions**
- `TS_GetCString()` returns NULL on missing entries (no exception thrown)
- Reflects pre-exception era C++ practices; callers must guard with `if (str != NULL)`
- Combined with silent-ignore on invalid indices in parsing, creates graceful degradation for malformed MML

**UTF-8 Decoding at Init, Not Retrieval**
- `DeUTF8()` called only during `parse_mml_stringset()`, storing pre-converted Mac Roman strings
- Avoids repeated decode cost at runtime (30 FPS frame budget sensitive)
- Assumes immutable MML config loaded once at startup (valid for this engine architecture)

## Data Flow Through This File

```
MML File (UTF-8 strings)
    Γåô
XML Parsing ΓåÆ InfoTree tree structure
    Γåô
parse_mml_stringset(root)
  Γö£ΓöÇ read_attr("index", ID)      [extract set ID]
  ΓööΓöÇ for each child "string":
     Γö£ΓöÇ read_indexed("index", cindex)  [extract index within set]
     Γö£ΓöÇ get_value<string>()  [extract UTF-8 payload]
     Γö£ΓöÇ DeUTF8_C()  [convert UTF-8 ΓåÆ Mac Roman, null-terminate]
     ΓööΓöÇ StringSetRoot[ID][cindex] = converted_string  [store in global map]

Runtime (per-frame HUD rendering):
    render_code calls TS_GetCString(ID, Index)
    Γåô
    StringSetRoot.find(ID) ΓåÆ if not found, return NULL
    Γåô
    StringSetRoot[ID].find(Index) ΓåÆ if not found, return NULL
    Γåô
    return string.c_str()  [pointer into std::string buffer]
```

**State Lifetime**: Strings loaded once during engine init via `_ParseAllMML()` (called by XML_MakeRoot.cpp), persist for engine lifetime, cleared on shutdown via `TS_DeleteAllStrings()` (called by engine cleanup).

## Learning Notes

**Marathon Era Localization Approach**
- String resources are **loaded from config files**, not compiled into binary (unlike modern engines which embed .pot files)
- Enables at-runtime localization without recompile; MML files can be patched post-release
- Mac Roman encoding reflects Mac OS era (pre-Unicode engine origin); modern engines assume UTF-8 throughout

**UTF-8 Decode Complexity**
- `DeUTF8()` implements RFC 3629 UTF-8 parsing: recognizes 1ΓÇô6 byte sequences, reassembles code points bit-by-bit
- Missing overlong detection (mentioned in comment line 223), which modern UTF-8 validators enforceΓÇöacceptable for trusted MML input
- Lossy conversion: codepoints outside U+0000ΓÇôU+FFFF or invalid sequences ΓåÆ '$' placeholder

**Contrast with Modern Approaches**
- Modern engines: centralized string manager with key-value (string ID Γåö localized text), plural forms, gender-aware substitution
- This engine: flat two-level indexing, no templating or pluralization (hardcoded strings per scenario)
- No thread-safety primitives; relies on single-threaded init phase and read-only runtime access

## Potential Issues

1. **Buffer Overflow in DeUTF8_C()** (line 133ΓÇô135)
   - Fixed `cbuf[256]` allocation; if a single MML string's UTF-8 expansion exceeds 256 bytes after Mac Roman conversion, it silently truncates
   - Mitigated by assumption that MML strings are curated, but could cause silent data loss with untrusted input

2. **Thread Safety Violation**
   - `StringSetRoot` global is accessed without synchronization; concurrent calls from background threads (e.g., rendering thread in multithreaded render pipeline) could race
   - Assumption: all `TS_GetCString()` calls happen on main thread after init phase completes

3. **Pointer Lifetime Fragility**
   - `TS_GetCString()` returns `c_str()` pointer into `std::string` stored in map; if `TS_DeleteString()` or `TS_DeleteStringSet()` is called while pointer is in use, dangling pointer results
   - No safeguard against accidental double-delete or mid-frame deletion

4. **Contiguity Claim Not Enforced** (line 80 comment)
   - Comment says "Count the strings (contiguous from index zero)" but `TS_CountStrings()` counts all entries (sparse indices allowed)
   - Could be misleading if code assumes contiguous indexing; no validation that indices are 0, 1, 2, ΓÇª NΓÇô1
