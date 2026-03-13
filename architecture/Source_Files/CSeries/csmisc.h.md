# Source_Files/CSeries/csmisc.h

## File Purpose
Header file providing timing and input utility abstractions for the Aleph One game engine. Declares platform-independent functions for machine tick access, thread yielding, and blocking input polls.

## Core Responsibilities
- Declare machine tick timing queries and delays
- Declare cooperative multitasking yield function
- Declare blocking input detection
- Define tick-per-second constant (1000 Hz)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### machine_tick_count
- Signature: `uint64_t machine_tick_count(void)`
- Purpose: Get current machine tick counter value
- Inputs: None
- Outputs/Return: Current tick count (uint64_t)
- Side effects: None (read-only)
- Calls: Not visible in this file
- Notes: Tick frequency is 1000 ticks/second; used for absolute time queries

### sleep_for_machine_ticks
- Signature: `void sleep_for_machine_ticks(uint32 ticks)`
- Purpose: Block execution for a relative duration
- Inputs: ticks (uint32) ΓÇô sleep duration in ticks
- Outputs/Return: None
- Side effects: Suspends thread/process
- Calls: Not visible in this file
- Notes: Relative delay; useful for frame-rate control

### sleep_until_machine_tick_count
- Signature: `void sleep_until_machine_tick_count(uint64_t ticks)`
- Purpose: Block execution until absolute tick count is reached
- Inputs: ticks (uint64_t) ΓÇô target tick value
- Outputs/Return: None
- Side effects: Suspends thread/process
- Calls: Not visible in this file
- Notes: Absolute deadline synchronization

### yield
- Signature: `void yield(void)`
- Purpose: Voluntarily relinquish CPU to other threads/processes
- Inputs: None
- Outputs/Return: None
- Side effects: Context switch; caller may be descheduled
- Calls: Not visible in this file
- Notes: Cooperative multitasking hint

### wait_for_click_or_keypress
- Signature: `bool wait_for_click_or_keypress(uint32 ticks)`
- Purpose: Poll for user input (mouse or keyboard) with timeout
- Inputs: ticks (uint32) ΓÇô maximum wait duration
- Outputs/Return: bool ΓÇô true if input detected, false on timeout
- Side effects: May block execution
- Calls: Not visible in this file
- Notes: Input-driven synchronization point

## Control Flow Notes
These primitives support the engine's timing and input infrastructure. Likely used in main loop synchronization, menu/dialogue blocking, and cooperative scheduling between subsystems. All implementations are defined elsewhere (platform-specific).

## External Dependencies
- `cstypes.h` ΓÇô type definitions (uint32, uint64_t, bool)
- Implementations not visible in this file; declared externally
