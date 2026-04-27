# Pillar 4: Atomic Operations — The Performance Pillar

> **Goal:** Perform thread-safe operations on simple data types without the overhead of a mutex. When a mutex is too heavy, atomics are your scalpel.

---

## Why Mutexes Can Be "Heavy"

Every time a thread locks a mutex, several things happen under the hood:

1. A **system call** is made to the OS kernel.
2. If the lock is taken, the thread is put to **sleep** (context switch).
3. When unlocked, the OS **wakes** the waiting thread (another context switch).

For something as simple as incrementing a counter thousands of times per second, this overhead is enormous. **Atomic operations solve this entirely in hardware** — no OS involvement, no sleeping, no context switching.

---

## What is an Atomic Operation?

An **atomic operation** is one that is guaranteed to complete as a single, indivisible step from the perspective of all other threads. No thread can ever observe it in a half-finished state.

The non-atomic `counter++` from Pillar 2 was three steps (Read → Add → Write). An atomic `counter++` is exactly one hardware-level operation — it cannot be interrupted or interleaved.

---

## `std::atomic<T>` — The Core Tool

```cpp
#include <atomic>

std::atomic<int>      counter{0};
std::atomic<bool>     flag{false};
std::atomic<uint64_t> bytes_processed{0};
```

`std::atomic<T>` works best with:
- Integral types (`int`, `long`, `uint64_t`, etc.)
- Pointer types
- `bool`
- Any **trivially copyable** type small enough for hardware support (typically ≤ 8 bytes)

---

## The Broken Counter — Fixed With Atomics

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<int> counter{0}; // Thread-safe, no mutex needed

void increment() {
    for (int i = 0; i < 100000; ++i) {
        counter++; // This is now a single atomic hardware instruction
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();

    std::cout << "Counter: " << counter.load() << "\n"; // Always 200000
    return 0;
}
```

### Performance Comparison

| Approach | Operations/sec (approx.) | OS Involvement |
|---|---|---|
| `std::mutex` + `int` | ~10–50 million | Yes — system calls |
| `std::atomic<int>` | ~200–500 million | No — hardware only |

---

## Core Atomic Operations

### Reading and Writing

```cpp
std::atomic<int> value{10};

// Read — always use .load(), not direct access
int x = value.load();                              // Default memory order
int y = value.load(std::memory_order_relaxed);     // Explicit order

// Write — always use .store()
value.store(42);
value.store(42, std::memory_order_release);
```

> ⚠️ Reading or writing an atomic via direct assignment (`int x = value` or `value = 42`) calls `.load()` / `.store()` implicitly, but being explicit makes your intent clear and helps code reviewers.

---

### Arithmetic Operations (Integers Only)

```cpp
std::atomic<int> n{5};

n++;         // Atomic increment (returns old value: 5, n is now 6)
n--;         // Atomic decrement
n += 10;     // Atomic add (n is now 15)
n -= 3;      // Atomic subtract (n is now 12)

// Explicit forms — return the old value
int old = n.fetch_add(1); // old = 12, n = 13
int old2 = n.fetch_sub(5); // old2 = 13, n = 8
```

---

### `compare_exchange` — The Foundation of Lock-Free Programming

`compare_exchange_strong` (CAS — Compare-And-Swap) is the most powerful atomic operation. It atomically:
1. Compares the current value to an **expected** value.
2. If they match, replaces it with the **desired** value and returns `true`.
3. If they don't match, loads the current value into **expected** and returns `false`.

```cpp
#include <iostream>
#include <atomic>

std::atomic<int> value{10};

int main() {
    int expected = 10;
    int desired  = 99;

    // "If value is 10, set it to 99"
    bool success = value.compare_exchange_strong(expected, desired);

    if (success) {
        std::cout << "Swapped! value is now: " << value.load() << "\n"; // 99
    } else {
        // expected is now updated to the actual current value
        std::cout << "Failed. Actual value was: " << expected << "\n";
    }
    return 0;
}
```

#### Real Use: Lock-Free Stack Push

```cpp
template<typename T>
struct Node {
    T data;
    Node* next;
};

std::atomic<Node<int>*> head{nullptr};

void push(int value) {
    Node<int>* newNode = new Node<int>{value, nullptr};
    // Retry loop — keep trying until our CAS succeeds
    do {
        newNode->next = head.load(); // Read current head
    } while (!head.compare_exchange_weak(newNode->next, newNode));
    // If head changed between our read and CAS, we retry automatically
}
```

> `compare_exchange_weak` vs `compare_exchange_strong`: `weak` can spuriously fail (returns `false` even on match) but may be faster on some architectures. Always use it inside a retry loop. Use `strong` when you only want to try once.

---

## `std::atomic<bool>` — The Flag Pattern

A common use case is a simple flag to signal between threads.

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<bool> stopSignal{false};

void worker() {
    while (!stopSignal.load()) {
        // Keep doing work...
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        std::cout << "Working...\n";
    }
    std::cout << "Worker: stop signal received, exiting.\n";
}

int main() {
    std::thread t(worker);

    std::this_thread::sleep_for(std::chrono::milliseconds(350));
    stopSignal.store(true); // Signal the worker to stop

    t.join();
    return 0;
}
```

---

## Memory Order — The Advanced Layer

This is the part most beginners skip — and then write subtly broken lock-free code.

Modern CPUs and compilers **reorder instructions** for performance. This is invisible when you're single-threaded, but between threads it can cause another thread to see your operations happening in a different order than you wrote them.

`std::atomic` operations take a `std::memory_order` argument that tells the CPU/compiler what reordering fences to apply.

### The Six Memory Orders (from weakest to strongest)

```
relaxed  →  consume  →  acquire  →  release  →  acq_rel  →  seq_cst
 (fast)                                                       (slow/safe)
```

| Memory Order | What It Does | When to Use |
|---|---|---|
| `memory_order_relaxed` | No ordering guarantees. Just atomicity. | Independent counters, stats |
| `memory_order_acquire` | No reads/writes after this can be reordered before it | Reading a flag that guards data |
| `memory_order_release` | No reads/writes before this can be reordered after it | Writing data, then setting a flag |
| `memory_order_acq_rel` | Both acquire and release | Read-Modify-Write (fetch_add, CAS) |
| `memory_order_seq_cst` | Total global ordering for all threads | Default — safest, and the slowest |

---

### The Canonical Acquire/Release Pattern

This is the most important pattern to learn. A `release` store *synchronizes with* an `acquire` load — this guarantees the data written before the `release` is visible to the thread that does the `acquire`.

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<bool> dataReady{false};
int               sensorValue = 0; // NOT atomic — protected by the flag's ordering

void producer() {
    sensorValue = 42;                                       // (1) Write data first
    dataReady.store(true, std::memory_order_release);       // (2) THEN set the flag
    // release guarantees (1) cannot be reordered after (2)
}

void consumer() {
    while (!dataReady.load(std::memory_order_acquire));     // (3) Spin until flag is true
    // acquire guarantees (4) cannot be reordered before (3)
    std::cout << "Value: " << sensorValue << "\n";          // (4) Safe to read — always 42
}

int main() {
    std::thread prod(producer);
    std::thread cons(consumer);
    prod.join();
    cons.join();
    return 0;
}
```

Without the acquire/release pairing, the consumer could see `dataReady == true` but still read `sensorValue == 0` because the CPU reordered the writes.

---

### Relaxed — For Pure Counters

When you only care that an operation is atomic (not about ordering relative to other variables), `relaxed` is fastest:

```cpp
std::atomic<uint64_t> totalBytesProcessed{0};

void networkWorker(uint64_t bytesThisChunk) {
    // We just want an accurate total — we don't care about ordering
    totalBytesProcessed.fetch_add(bytesThisChunk, std::memory_order_relaxed);
}
```

---

## `std::atomic_flag` — The Lightest Atomic

`std::atomic_flag` is the only type **guaranteed** to be lock-free on every platform. It has exactly two operations: `test_and_set()` and `clear()`.

It's used to build **spinlocks** — a busy-waiting mutex that avoids OS calls entirely (useful when you expect very short wait times).

```cpp
#include <atomic>
#include <thread>

class Spinlock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
public:
    void lock() {
        // Spin until we successfully set the flag (acquire it)
        while (flag.test_and_set(std::memory_order_acquire)) {
            // Hint to the CPU that we're in a spin loop — improves performance
            // on x86: __builtin_ia32_pause() or std::this_thread::yield()
        }
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
};

Spinlock sl;
int counter = 0;

void work() {
    for (int i = 0; i < 10000; ++i) {
        sl.lock();
        counter++;
        sl.unlock();
    }
}

int main() {
    std::thread t1(work), t2(work);
    t1.join(); t2.join();
    // counter is always 20000
}
```

> Use spinlocks only when the critical section is **extremely short** (nanoseconds). For longer sections, a regular mutex performs better because it lets waiting threads sleep.

---

## `std::atomic` vs `std::mutex` — Decision Guide

```
Is the operation on a simple type (int, bool, pointer)?
│
├─ YES ─► Is it just a read, write, or arithmetic?
│         ├─ YES ─► Use std::atomic<T>. Done.
│         └─ NO  ─► Is it a CAS / compare-then-act?
│                   ├─ YES ─► Use compare_exchange in a retry loop.
│                   └─ NO  ─► Use std::mutex.
│
└─ NO  ─► Use std::mutex + a proper data structure.
```

| Scenario | Best Tool |
|---|---|
| Shared counter / stats | `std::atomic<int>` with `fetch_add` |
| Stop signal flag | `std::atomic<bool>` |
| Protecting a struct / multiple variables together | `std::mutex` + `lock_guard` |
| Building a lock-free data structure | `compare_exchange_weak` |
| Building your own fast mutex | `std::atomic_flag` + spinlock |

---

## Key Takeaways

- `std::atomic<T>` makes read-modify-write operations on simple types thread-safe without any OS overhead.
- Always use `.load()` and `.store()` explicitly for clarity.
- `compare_exchange` is the primitive that powers all lock-free data structures.
- Memory order matters — `acquire`/`release` is the standard pairing; `relaxed` for isolated counters; `seq_cst` is the safe default.
- Atomics don't replace mutexes — they complement them. Use atomics for simple flags and counters; mutexes for guarding compound operations and data structures.

---

*Previous: [Pillar 3 — Communication & Coordination](pillar3_communication.md)*  
*Back to start: [Pillar 1 — Thread Management](pillar1_thread_management.md)*
