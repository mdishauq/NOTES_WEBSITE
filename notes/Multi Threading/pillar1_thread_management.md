# Pillar 1: Thread Management — The Execution Pillar

> **Goal:** Start and stop workers safely. Before you can synchronize data or coordinate tasks, you must understand how to create, control, and correctly destroy threads.

---

## What is a Thread?

A **thread** is an independent path of execution inside your program. Your `main()` function already runs on one thread — the *main thread*. When you create additional threads, your program can do multiple things simultaneously (or appear to).

Think of threads like workers on an assembly line. The factory manager (your OS) decides which worker gets to run at any moment, but all workers share the same workspace (your process memory).

---

## The Core Problem: Thread Lifecycle

Every `std::thread` object has a strict lifecycle rule:

> **A thread object must be either `.join()`-ed or `.detach()`-ed before it is destroyed.**

If you break this rule, the C++ runtime calls `std::terminate()` — your program crashes immediately, no exception, no warning.

---

## `std::thread` — The Basic Unit

### Syntax

```cpp
#include <thread>

// Creating a thread
std::thread myThread(callable, args...);
```

The callable can be a plain function, a lambda, or a functor.

---

### `.join()` — Wait for the Thread to Finish

When you call `.join()`, the calling thread **blocks** (pauses) until the target thread completes its work. Use this when you need the result of the thread's work before moving on.

```cpp
#include <iostream>
#include <thread>

void doWork(int id) {
    std::cout << "Worker " << id << " is running.\n";
    // simulate work...
}

int main() {
    std::thread worker(doWork, 42);

    std::cout << "Main thread: waiting for worker...\n";

    worker.join(); // Main BLOCKS here until worker finishes

    std::cout << "Main thread: worker is done.\n";
    return 0;
}
```

**Output (order of first two lines may vary):**
```
Main thread: waiting for worker...
Worker 42 is running.
Main thread: worker is done.
```

---

### `.detach()` — Fire and Forget

When you call `.detach()`, the thread is released from the `std::thread` object and runs completely independently. The thread object no longer manages it. Use this only when you genuinely do not care when or whether the thread finishes (e.g., a background logger).

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void backgroundLogger() {
    // Runs independently, main thread won't wait
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    std::cout << "Background log written.\n";
}

int main() {
    std::thread logger(backgroundLogger);
    logger.detach(); // Released — we no longer manage it

    std::cout << "Main continues immediately.\n";
    // WARNING: If main() exits before the detached thread finishes,
    // the thread is killed abruptly!
    std::this_thread::sleep_for(std::chrono::seconds(1)); // give it time
    return 0;
}
```

> ⚠️ **Danger:** A detached thread accesses memory (local variables, stack frames) that may have already been destroyed if the thread outlives the scope that created it. This is a common source of crashes.

---

### Passing Arguments to Threads

```cpp
#include <iostream>
#include <thread>
#include <string>

void greet(const std::string& name, int times) {
    for (int i = 0; i < times; ++i) {
        std::cout << "Hello, " << name << "!\n";
    }
}

int main() {
    // Arguments are COPIED by default
    std::thread t(greet, "Alice", 3);
    t.join();
    return 0;
}
```

#### Passing by Reference — Use `std::ref`

```cpp
#include <iostream>
#include <thread>

void increment(int& value) {
    value += 10;
}

int main() {
    int x = 5;
    // std::ref is REQUIRED — threads copy args by default
    std::thread t(increment, std::ref(x));
    t.join();
    std::cout << "x = " << x << "\n"; // x = 15
    return 0;
}
```

---

### Using Lambdas

```cpp
#include <iostream>
#include <thread>

int main() {
    int result = 0;

    std::thread t([&result]() {
        result = 100; // captures result by reference
    });

    t.join();
    std::cout << "Result: " << result << "\n"; // 100
    return 0;
}
```

---

## `std::jthread` (C++20) — The Safer Thread

`std::jthread` is a "joining thread." It automatically calls `.join()` when it goes out of scope — you can never forget to join it. It also supports **cooperative cancellation** via `std::stop_token`.

```cpp
#include <iostream>
#include <thread>   // includes jthread in C++20

void doWork(std::stop_token stoken, int id) {
    while (!stoken.stop_requested()) {
        std::cout << "Worker " << id << " is working...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(300));
    }
    std::cout << "Worker " << id << " received stop signal.\n";
}

int main() {
    // jthread automatically joins when it goes out of scope
    std::jthread worker(doWork, 1);

    std::this_thread::sleep_for(std::chrono::seconds(1));

    worker.request_stop(); // Politely ask the thread to stop
    // worker.join() is called automatically here when worker goes out of scope
    std::cout << "Main: all done.\n";
    return 0;
}
```

### `std::thread` vs `std::jthread` — Quick Comparison

| Feature | `std::thread` | `std::jthread` (C++20) |
|---|---|---|
| Auto-joins on destruction | ❌ No (crashes!) | ✅ Yes |
| Cooperative cancellation | ❌ No | ✅ Via `std::stop_token` |
| Overhead | Minimal | Minimal |
| When to use | Legacy / full control needed | Default choice in C++20 |

---

## Hardware Concurrency — How Many Threads?

```cpp
#include <iostream>
#include <thread>

int main() {
    unsigned int cores = std::thread::hardware_concurrency();
    std::cout << "Logical CPU cores available: " << cores << "\n";
    // Use this to decide thread pool size
    return 0;
}
```

---

## Thread Lifecycle — The Full Picture

```
                  ┌─────────────┐
                  │  Created    │  std::thread t(func);
                  └──────┬──────┘
                         │
              ┌──────────┴──────────┐
              │                     │
         .join()               .detach()
              │                     │
     ┌────────▼────────┐   ┌────────▼────────┐
     │ Main BLOCKS     │   │ Thread runs     │
     │ until thread    │   │ independently   │
     │ finishes        │   │ (no handle)     │
     └────────┬────────┘   └─────────────────┘
              │
     ┌────────▼────────┐
     │ Thread object   │
     │ safely destroyed│
     └─────────────────┘
```

---

## Common Beginner Mistakes

### ❌ Forgetting to join or detach

```cpp
void badExample() {
    std::thread t([]{ /* work */ });
    // t goes out of scope here — std::terminate() is called!
}
```

### ❌ Accessing destroyed stack variables from a detached thread

```cpp
void dangerous() {
    int localVar = 42;
    std::thread t([&localVar]{ // captures reference to local!
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << localVar; // localVar is GONE — undefined behavior
    });
    t.detach();
} // localVar destroyed here, but thread still running!
```

### ✅ Safe fix — capture by value

```cpp
void safe() {
    int localVar = 42;
    std::thread t([localVar]{ // copy, not reference
        std::cout << localVar; // always valid
    });
    t.detach();
}
```

---

## Key Takeaways

- Every `std::thread` **must** be joined or detached — no exceptions.
- Use `.join()` when you need the thread's work to complete before proceeding.
- Use `.detach()` sparingly and only when the thread's lifetime is clearly managed.
- Prefer `std::jthread` in C++20 for automatic, crash-free cleanup.
- Always pass by value (or use `std::ref` consciously) to avoid dangling references.

---

*Next: [Pillar 2 — Synchronization & Mutual Exclusion](pillar2_synchronization.md)*
