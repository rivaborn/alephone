# Source_Files/Misc/steamshim_child.h - Enhanced Analysis

## Architectural Role

This header defines the **IPC contract between the main game engine and a sandboxed Steam API client process**. The shim pattern isolates the unstable Steam SDK behind a request-response boundary, protecting engine stability while maintaining full Workshop integration (uploads, queries, item discovery) and achievement/stats tracking. The async event-pump architecture allows the engine to continue rendering/simulating during slow Steam operations without blocking.

## Key Cross-References

### Incoming (who depends on this file)

- **Misc subsystem**: `preferences_widgets_sdl.cpp` calls `STEAMSHIM_queryWorkshopItemOwned()` and `STEAMSHIM_queryWorkshopItemMod()` for Workshop item UI (preferences dialogs)
- **Files subsystem**: Workshop item discovery and loading depends on `item_subscribed_query_result` to enumerate installed mods
- **Interface/Shell**: Achievement/stat functions called from main game loop and level completion handlers
- **Network subsystem**: Multiplayer scenarios may query Workshop items for map sync via subscribed results

### Outgoing (what this file depends on)

- **CSeries**: `uint8`, `uint64_t` integer type definitions; platform abstraction globals (used in serialization type casts)
- **C++ Standard Library**: `<sstream>`, `<string>`, `<vector>` for IPC data marshaling
- **Steam API shim parent** (`Source_Files/Steam/steamshim_parent.cpp`): Implements all declared functions; receives these structs and enums across process boundary

## Design Patterns & Rationale

**Shim + Sandbox Pattern**
- Isolates flaky Steam SDK from engine crash domain; child process can restart independently
- Binary-only API prevents accidental Steam header includes in main engine code
- Clear architectural boundary: what Steam concerns are versus engine concerns

**Async Event Pump**
- Non-blocking request-response design; game loop calls `STEAMSHIM_pump()` each frame, not blocking on I/O
- Decouples timing: Workshop uploads may take seconds; engine continues simulating/rendering
- Simpler than callback-based async (no threading/signal concerns in main thread)

**Manual Binary Serialization**
- Pre-JSON/protobuf era pattern; compact fixed-size records for IPC efficiency
- Endianness implicit (likely big-endian by convention, matching Marathon 1 WAD format heritage)
- Reinterpret casting (`reinterpret_cast<char*>(&id)`) assumes standard layout and no paddingΓÇöfragile across ABIs

**Union-Like Event Structure** (`STEAMSHIM_Event`)
- Single polymorphic container: `okay`, `ivalue`, `fvalue`, `name[]`, `items_owned`, `items_subscribed`, `game_info`
- Type safety weakness (caller must know which fields are valid for each event type)
- Memory waste (event always allocates vectors even if unused)
- Simplifies pumping interface (one pointer returned, not union dispatch)

## Data Flow Through This File

1. **Workshop Upload Lifecycle**
   - Engine: `STEAMSHIM_uploadWorkshopItem(item_upload_data)` ΓåÆ child serializes and uploads to Steam API
   - Child: tracks `STEAM_EItemUpdateStatus` progress (preparing config ΓåÆ uploading content ΓåÆ committing)
   - Engine: Polls via `STEAMSHIM_pump()` ΓåÆ receives `SHIMEVENT_WORKSHOP_UPLOAD_PROGRESS` and `_RESULT` events with percent/success
   - Data enters as `item_upload_data` struct, exits as event status and result code

2. **Workshop Query (Owned Items)**
   - Engine: `STEAMSHIM_queryWorkshopItemOwned(scenario_name)` ΓåÆ child queries Steam for user-owned items compatible with scenario
   - Child: Builds `item_owned_query_result` with vector of items (id, type, content_type, title, compatibility flag)
   - Child: Serializes struct to binary blob, sends back
   - Engine: `STEAMSHIM_pump()` deserializes via `item_owned_query_result::shim_deserialize()`, returns event with populated items vector
   - Used by preferences UI to populate Workshop item lists; cached by caller until next query

3. **Stats/Achievement Flow**
   - Engine: `STEAMSHIM_setStatI("key", value)` ΓåÆ queued in child, not immediately persisted
   - Engine: `STEAMSHIM_storeStats()` ΓåÆ child flushes all pending to Steam API
   - Engine: `STEAMSHIM_getStatI("key")` ΓåÆ async query; result polled as `SHIMEVENT_GETSTATI` event with `ivalue`
   - Sequential, not transactional: no rollback if store fails

## Learning Notes

**Idiomatic to this era (pre-C++17, Marathon legacy)**
- Manual binary struct serialization instead of Protobuf/JSON; reflects early-2000s simplicity
- Reinterpret casting + placement new pattern for zero-copy deserialization; common when memory was tight
- Raw pointer from pump() returning static/thread-local buffer; requires poll-every-frame pattern
- Event enum dispatch instead of virtual functions; matches functional/procedural game loop style

**What modern engines do differently**
- **Type-safe async**: Rust Result<T>, C++20 coroutines, or promise-based patterns instead of event polling
- **Structured serialization**: Protobuf, JSON, or language-native serialization with schema evolution
- **Capability-based API**: Optional<T>, Expected<T, E> for Results instead of union-like containers
- **IPC frameworks**: gRPC, D-Bus, or message queues instead of bespoke binary framing

**Engine-specific patterns**
- Steam shim sits at similar abstraction level as Lua scripting interface: thin wrapper over external API, event-driven
- Binary serialization mirrors Files subsystem (WAD format, AStream) for consistency across the codebase
- Workshop integration pattern (query ΓåÆ result callback) matches how Metaserver network queries work

## Potential Issues

1. **Buffer overflow in deserialization**: `item_owned_query_result::shim_deserialize()` reads `number_items` from buffer, then loopsΓÇöno bounds check if count claims more items than buffer contains. Could read past `buflen`.

2. **Alignment assumptions**: Binary structs (uint64_t, int, bool mixed) assume no padding between fields and platform byte order. ARM/x86 compatibility unclear; unspecified endianness in shim protocol.

3. **Type safety loss**: `STEAMSHIM_Event` fields (ivalue, fvalue, items_owned, items_subscribed) are all optional depending on event type. Caller must know which to readΓÇöcompiler won't catch mistakes.

4. **No versioning/forwards compatibility**: Binary protocol has no version field. If child process binary upgraded but engine not (or vice versa), deserialization silently corrupts (e.g., ContentType enum values assumed fixed).

5. **String handling**: `char name[256]` in event is fixed-size; no bounds checking when writing. Title strings in item structs use `std::getline(iss, title, '\0')` which trusts null-terminator in buffer.

6. **Memory leak in async queries**: Caller receives vector of items; if event handling crashes, items vector leaks (no RAII cleanup in calling code shown).
