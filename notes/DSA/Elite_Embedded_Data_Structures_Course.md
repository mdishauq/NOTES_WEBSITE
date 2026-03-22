# Elite Embedded Data Structures

## Course Vision
Build production grade data structure skills for embedded systems using C and C++ with a static-first mindset.

### Core Engineering Values
- Determinism first
- Fixed memory usage
- Predictable latency
- Hardware aware design
- Test and measurement driven decisions

---

## Learning Contract
By the end of this course you should be able to:
- Implement and validate data structures without runtime heap dependence
- Explain memory and timing behavior in worst case terms
- Choose C or C++ techniques based on safety, footprint, and maintainability
- Build systems that are robust under interrupts, burst traffic, and fault conditions

---

## Module 0: Measurement and Tooling Foundations
### Why this comes first
If you cannot measure cycles, RAM, and flash, you cannot optimize with confidence.

### Topics
- Memory map literacy: text, rodata, data, bss, stack, heap
- Linker map reading and symbol size attribution
- Cycle counting techniques on microcontrollers
- Worst case latency and jitter tracking
- Build mode tradeoffs: debug, O2, Os, LTO
- Static analysis and warnings as errors

### C Track
- Toolchain flags and map driven analysis
- Macro based timing probes
- Assert policy in debug versus release

### C++ Track
- Constexpr guards where possible
- No exceptions and no RTTI build profiles
- Template instantiation cost awareness

---

## Module 1: Memory Management and Lifetime Control
### Topics
- Flash versus SRAM behavior
- Heap fragmentation risks in real time systems
- Static and pool based allocation models
- Alignment and padding effects
- DMA safe memory placement
- Startup initialization cost

### C Track
- Fixed block allocators
- Region allocators
- Linker section placement strategies

### C++ Track
- Placement new
- Polymorphic memory resources
- Allocation policy control

---

## Module 2: Circular Buffer Mastery
### Topics
- Single Producer Single Consumer model
- Head and tail invariants
- Power of two sizing and mask based wraparound
- Overflow policy design
- Zero copy access patterns
- ISR to task pipeline design

### C Track
- Encapsulated struct and API discipline
- Deterministic push and pop behavior
- Optional atomics for stronger correctness

### C++ Track
- Template fixed capacity ring buffer
- Compile time capacity checks
- No heap and exception free API design

---

## Module 3: Linked Lists for Drivers
### Topics
- Singly versus doubly linked choices
- Intrusive list pattern
- Constant time insertion and removal
- Safe iteration during mutation

### C Track
- Container ownership via embedded node
- Kernel style list discipline

### C++ Track
- Strongly typed intrusive hooks
- Static polymorphism patterns

---

## Module 4: Stacks, Queues, and Scheduling Structures
### Topics
- Stack behavior in interrupt scenarios
- Queue latency and backpressure
- Fixed capacity priority structures
- Ready list bitmaps and timing wheels

### C Track
- Array based heap scheduler
- Manual sift operations

### C++ Track
- Priority queue with footprint aware configuration
- Deterministic container policies

---

## Module 5: Bit Level Structures
### Topics
- Register field mapping
- Bit packed state machines
- Endian safe packing and unpacking
- Atomic bit updates in shared state

### C Track
- Mask and shift mastery
- Union based register views with caution

### C++ Track
- Bitset and array based compact state
- Constexpr mask and field helpers

---

## Module 6: Static Search and Lookup
### Topics
- Sorted flash lookup and binary search
- Static tree layouts on arrays
- Perfect hashing for command parsing
- Collision handling under strict constraints

### C Track
- ROM lookup tables
- Binary search utilities

### C++ Track
- Constexpr lookup maps
- Fixed container search patterns

---

## Module 7: Concurrency and Memory Model
### Topics
- Why volatile is not synchronization
- Acquire release semantics in practice
- ISR and task handoff patterns
- Lock free boundaries and pitfalls

### C Track
- C11 atomics and fences

### C++ Track
- Atomic type safe synchronization wrappers

---

## Module 8: Safety and Robustness
### Topics
- Undefined behavior elimination
- Defensive API design
- Error propagation without exceptions
- Fault injection and recovery testing

### C Track
- Status code based flows
- Explicit state machine error paths

### C++ Track
- Strong enum and span based safer APIs
- Lightweight result types

---

## Module 9: Integration and Architecture
### Topics
- Combining structures in one embedded pipeline
- End to end timing budget enforcement
- Memory budget and overflow strategy alignment
- Production instrumentation hooks

### C Track
- Explicit ownership and module boundaries

### C++ Track
- Zero overhead abstraction boundaries

---

## Final Capstone Project
## Real Time Telemetry and Command Engine

### Build Objective
Create a full subsystem that:
- Captures UART command data
- Parses commands quickly from static tables
- Schedules work by priority
- Packs telemetry into compact bit fields
- Guarantees deterministic runtime behavior

### Required Structure Usage
- Ring buffers for UART RX and TX
- Static command lookup for parser dispatch
- Priority queue for scheduled jobs
- Intrusive list for active work tracking
- Bit packed telemetry status

### Acceptance Criteria
- No runtime allocation after initialization
- Bounded worst case command to action latency
- Proven overflow behavior under stress
- Verified RAM and flash budgets
- Clean recovery from malformed frames and overload

---

## Suggested Weekly Rhythm
- 2 days implementation
- 1 day measurement and profiling
- 1 day hardening and review
- 1 day challenge or mini project

## Assessment Style
- Practical first
- Invariant based reasoning
- Performance evidence required
- Clear failure mode analysis

---

## Final Note
Elite embedded engineering is the ability to make systems that are not just fast, but predictable under pressure.