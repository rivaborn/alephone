# Source_Files/Misc/thread_priority_sdl_win32.cpp

## File Purpose
Windows-specific implementation for adjusting thread priorities in an SDL-based game engine. Boosts network/worker thread priority to improve performance, with fallback strategies for older Windows versions and a main-thread priority reduction as a last resort.

## Core Responsibilities
- Boost SDL worker thread priority using Windows API (with version-specific fallbacks)
- Reduce main thread priority as compensation when worker boost is unavailable
- Prevent repeated main-thread priority reductions
- Handle dynamic loading of kernel32.dll and OpenThread function lookup
- Provide warnings on priority adjustment failures

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OpenThreadPtrT` | typedef | Function pointer for `OpenThread()` API, enabling dynamic lookup for version compatibility |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `isMainThreadPriorityReduced` | bool | static (function-local) | Guard flag in `TryToReduceMainThreadPriority()` to ensure main thread priority is reduced at most once per session |

## Key Functions / Methods

### TryToReduceMainThreadPriority
- Signature: `static bool TryToReduceMainThreadPriority()`
- Purpose: Reduce the current (main) thread's priority as a fallback when worker thread boosting fails
- Inputs: None
- Outputs/Return: `true` if reduction succeeded or was already done; `false` if SetThreadPriority failed
- Side effects: Modifies Windows thread priority for the calling thread; sets static guard flag
- Calls: `GetCurrentThread()`, `SetThreadPriority()`
- Notes: Guard flag prevents repeated calls from re-reducing priority; called only after worker boost fails

### BoostThreadPriority
- Signature: `bool BoostThreadPriority(SDL_Thread* inThread)`
- Purpose: Attempt to boost the specified SDL thread's priority for improved performance; compensate by reducing main thread priority if necessary
- Inputs: `inThread` (SDL_Thread pointer to the target worker thread)
- Outputs/Return: `true` if any priority adjustment succeeded; `false` only if all strategies failed
- Side effects: Modifies Windows thread priority (target thread and/or main thread); prints warnings to stdout on failures; dynamically loads kernel32.dll
- Calls: `GetModuleHandle()`, `GetProcAddress()`, `SDL_GetThreadID()`, `OpenThread()` (via function pointer), `SetThreadPriority()` (multiple times), `CloseHandle()`, `FreeLibrary()`, `TryToReduceMainThreadPriority()`
- Notes: 
  - Attempts modern Windows API (OpenThread) first for Win2000/WinXP+
  - Falls back to legacy undocumented Win98 approach if modern API unavailable
  - Tries multiple priority levels (TIME_CRITICAL ΓåÆ HIGHEST ΓåÆ ABOVE_NORMAL) with OR logic (any success counts)
  - Main thread reduction is only attempted if all thread boost attempts fail

## Control Flow Notes
- Initialization phase: Called (likely) once per worker thread that needs performance priority (e.g., network/rendering thread)
- Does not fit traditional frame loop (one-time setup, not per-frame)
- Compensation strategy: If cannot boost worker, reduces main thread priority to give worker relative advantage
- Guard mechanism prevents main thread priority pollution across multiple worker boosting attempts

## External Dependencies
- **Headers/Imports:**
  - `thread_priority_sdl.h` (local header with function declaration)
  - `<SDL2/SDL_thread.h>` (SDL threading types and functions)
  - `<windows.h>` (Windows API: thread, module, memory functions)
  - `<stdio.h>` (printf for diagnostics)

- **External symbols:**
  - SDL: `SDL_GetThreadID()` (defined elsewhere)
  - Windows API: `GetModuleHandle()`, `GetProcAddress()`, `OpenThread()`, `SetThreadPriority()`, `CloseHandle()`, `FreeLibrary()`, `GetCurrentThread()` (Windows kernel)
