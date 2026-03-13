# Source_Files/Misc/binders.h

## File Purpose
Defines a binding/synchronization system for mirroring state between pairs of objects. Allows decoupled objects that implement the `Bindable` interface to synchronize their state bidirectionally through `Binder` connectors managed by a `BinderSet` container.

## Core Responsibilities
- Define interface for objects that can export/import state (`Bindable<T>`)
- Implement abstract binding protocol (`ABinder`) for type-agnostic operations
- Provide concrete typed binder to synchronize two `Bindable<T>` objects
- Manage collections of bindings and batch synchronization operations
- Handle lifetime management of all bound pairs

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Bindable<T>` | Template interface (abstract class) | Mixin interface allowing objects to export and import typed state |
| `ABinder` | Abstract class | Type-erased base for all binders; enables polymorphic storage |
| `Binder<T>` | Template class (concrete) | Concrete bidirectional synchronizer between two `Bindable<T>` objects |
| `BinderSet` | Class | Container managing multiple `ABinder` pointers; orchestrates batch syncs |

## Global / File-Static State
None.

## Key Functions / Methods

### Bindable<T>::bind_export
- **Signature:** `virtual T bind_export () = 0`
- **Purpose:** Pure virtual; subclass exports its current state as value of type `T`
- **Inputs:** None
- **Outputs/Return:** Value of type `T` (the exported state)
- **Side effects:** None (should be const in practice)
- **Calls:** None
- **Notes:** Caller responsible for passing result to `bind_import` on target

### Bindable<T>::bind_import
- **Signature:** `virtual void bind_import (T) = 0`
- **Purpose:** Pure virtual; subclass imports/applies received state
- **Inputs:** Typed state value
- **Outputs/Return:** None
- **Side effects:** Updates internal state
- **Calls:** None
- **Notes:** Implementations should apply/apply state synchronously

### Binder<T>::migrate_first_to_second
- **Signature:** `void migrate_first_to_second ()`
- **Purpose:** Synchronize state from `thing1` to `thing2`
- **Inputs:** None (uses stored pointers)
- **Outputs/Return:** None
- **Side effects:** Calls `bind_export()` on `thing1`, then `bind_import()` on `thing2`
- **Calls:** `Bindable<T>::bind_export()`, `Bindable<T>::bind_import()`
- **Notes:** One-direction sync; both objects must be valid (checked at `insert()` time)

### Binder<T>::migrate_second_to_first
- **Signature:** `void migrate_second_to_first ()`
- **Purpose:** Synchronize state from `thing2` to `thing1`
- **Inputs:** None (uses stored pointers)
- **Outputs/Return:** None
- **Side effects:** Calls `bind_export()` on `thing2`, then `bind_import()` on `thing1`
- **Calls:** `Bindable<T>::bind_export()`, `Bindable<T>::bind_import()`
- **Notes:** Mirrors `migrate_first_to_second()` direction

### BinderSet::insert
- **Signature:** `template<typename T> void insert (Bindable<T>* first, Bindable<T>* second)`
- **Purpose:** Register a new binding pair
- **Inputs:** Two non-null `Bindable<T>` pointers
- **Outputs/Return:** None
- **Side effects:** Allocates `Binder<T>`, stores in `m_list`; does nothing if either pointer is null
- **Calls:** `Binder<T>` constructor
- **Notes:** Null-check guards against accidental dangling pointers; ownership transferred to `BinderSet`

### BinderSet::migrate_all_first_to_second
- **Signature:** `void migrate_all_first_to_second ()`
- **Purpose:** Trigger firstΓåÆsecond sync on all registered bindings
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `ABinder::migrate_first_to_second()` on each binder via `for_each`
- **Calls:** `std::for_each`, `call_first_second` (static callback)
- **Notes:** Uses function pointer callback; executes in insertion order

### BinderSet::migrate_all_second_to_first
- **Signature:** `void migrate_all_second_to_first ()`
- **Purpose:** Trigger secondΓåÆfirst sync on all registered bindings
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `ABinder::migrate_second_to_first()` on each binder via `for_each`
- **Calls:** `std::for_each`, `call_second_first` (static callback)
- **Notes:** Mirrors `migrate_all_first_to_second()`

## Control Flow Notes
Not inferable from this file. This is a utility header; orchestration logic would reside in the game loop or state management system that calls `migrate_all_*` methods at appropriate update points.

## External Dependencies
- `#include <list>` ΓÇö std::list container
- `#include <algorithm>` ΓÇö std::for_each algorithm
- No engine-specific dependencies
