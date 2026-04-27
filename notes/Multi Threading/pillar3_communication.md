# Pillar 3: Communication & Coordination — The Coordination Pillar

> **Goal:** Make threads work in a specific order. Mutexes protect data, but sometimes threads need to *signal* each other — "I'm done, you can start now." That's what this pillar is about.

---

## The Problem: Busy-Waiting

Imagine Thread A needs to process data that Thread B hasn't prepared yet. The naive solution is a **spin loop**:

```cpp
// BAD — Busy-waiting (polling)
while (!dataReady) {
    // Continuously checking... burns 100% CPU doing nothing useful
}
processData();
```

This wastes CPU cycles and blocks other threads from running. The correct solution is to put the waiting thread to **sleep** and have the other thread **wake it up** when the data is ready.

---

## `std::condition_variable` — The Signaling Mechanism

A condition variable allows a thread to:
1. **Wait** (sleep) until a specific condition becomes true.
2. **Notify** one or all waiting threads when the condition changes.

It must always be used together with a `std::unique_lock<std::mutex>`.

### Core API

```cpp
std::condition_variable cv;

// On the WAITING thread:
cv.wait(lock, predicate);    // Sleep until predicate() returns true
cv.wait_for(lock, duration); // Sleep for a duration (with timeout)
cv.wait_until(lock, time);   // Sleep until a time point

// On the NOTIFYING thread:
cv.notify_one(); // Wake up ONE waiting thread
cv.notify_all(); // Wake up ALL waiting threads
```

---

### Basic Producer–Consumer Example

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <string>

std::mutex mtx;
std::condition_variable cv;
std::string data;
bool dataReady = false; // The condition we're waiting on

void producer() {
    // Simulate preparation work
    std::this_thread::sleep_for(std::chrono::milliseconds(500));

    {
        std::lock_guard<std::mutex> lock(mtx);
        data      = "Sensor Reading: 42.7°C";
        dataReady = true;
        std::cout << "Producer: data is ready.\n";
    } // Lock released here

    cv.notify_one(); // Wake up the consumer
}

void consumer() {
    std::unique_lock<std::mutex> lock(mtx); // unique_lock required by cv.wait

    // Wait until dataReady is true
    // This atomically releases the lock and puts the thread to sleep
    cv.wait(lock, []{ return dataReady; });
    // When woken up, the lock is automatically re-acquired

    std::cout << "Consumer: received — " << data << "\n";
}

int main() {
    std::thread prod(producer);
    std::thread cons(consumer);
    prod.join();
    cons.join();
    return 0;
}
```

**Output:**
```
Producer: data is ready.
Consumer: received — Sensor Reading: 42.7°C
```

---

## The Spurious Wakeup — Why You Always Need a Predicate

A **spurious wakeup** is when a thread wakes up from `cv.wait()` even though `notify_one()` was never called. This is not a bug — it is permitted by the C++ standard and happens due to OS-level behavior on certain platforms.

### ❌ Dangerous — No Predicate

```cpp
cv.wait(lock); // Thread may wake up for NO reason and process garbage data!
```

### ❌ Also Dangerous — Manual While Loop (easy to get wrong)

```cpp
while (!dataReady) {
    cv.wait(lock); // Correct logic, but verbose and easy to forget
}
```

### ✅ Correct — Use the Predicate Overload

```cpp
// This internally does: while (!predicate()) { cv.wait(lock); }
cv.wait(lock, []{ return dataReady; });
```

The predicate overload is the idiomatic and safe way. **Always use it.**

---

## `notify_one()` vs `notify_all()`

```cpp
cv.notify_one(); // Wakes exactly one waiting thread (OS chooses which)
cv.notify_all(); // Wakes ALL waiting threads — they all compete for the lock
```

**When to use `notify_all()`:** When multiple consumers are waiting and any of them can handle the work (e.g., a thread pool), or when the condition change is relevant to all of them.

```cpp
// Thread pool example — wake all workers to check for a new task
{
    std::lock_guard<std::mutex> lock(mtx);
    newTaskAvailable = true;
}
cv.notify_all(); // All idle workers wake up; one grabs the task, rest go back to sleep
```

---

## `std::future` and `std::promise` — Return a Value from a Thread

A `promise`/`future` pair is a one-shot communication channel for returning a single value from one thread to another.

- **`std::promise<T>`** — The sender side. The thread with the promise sets a value.
- **`std::future<T>`** — The receiver side. The calling thread retrieves the value (blocking if not yet ready).

```cpp
#include <iostream>
#include <thread>
#include <future>

void heavyCalculation(std::promise<int> resultPromise) {
    std::this_thread::sleep_for(std::chrono::milliseconds(300));
    int result = 6 * 7; // The answer
    resultPromise.set_value(result); // Send the result
}

int main() {
    std::promise<int> prom;
    std::future<int>  fut = prom.get_future(); // Get the receiver

    std::thread t(heavyCalculation, std::move(prom)); // Move promise into thread

    std::cout << "Main: doing other work while thread calculates...\n";

    int answer = fut.get(); // Blocks until the promise is fulfilled
    std::cout << "Main: the answer is " << answer << "\n"; // 42

    t.join();
    return 0;
}
```

### Propagating Exceptions Through a Future

```cpp
void mightFail(std::promise<int> prom) {
    try {
        throw std::runtime_error("Something went wrong!");
        prom.set_value(42);
    } catch (...) {
        prom.set_exception(std::current_exception()); // Forward exception
    }
}

int main() {
    std::promise<int> prom;
    std::future<int>  fut = prom.get_future();

    std::thread t(mightFail, std::move(prom));
    t.join();

    try {
        int value = fut.get(); // Re-throws the exception here!
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }
}
```

---

## `std::async` — The Easiest Way to Run Work Asynchronously

`std::async` launches a function asynchronously and returns a `std::future` for the result. It hides thread creation and lifetime management from you entirely.

```cpp
#include <iostream>
#include <future>

int computeSquare(int n) {
    std::this_thread::sleep_for(std::chrono::milliseconds(200));
    return n * n;
}

int main() {
    // Launch asynchronously — returns immediately
    std::future<int> result = std::async(std::launch::async, computeSquare, 9);

    std::cout << "Main: doing other work...\n";

    // Retrieve result — blocks only if not ready yet
    std::cout << "Result: " << result.get() << "\n"; // 81
    return 0;
}
```

### Launch Policies

```cpp
// Force a new thread — always runs concurrently
std::async(std::launch::async,    myFunc, args...);

// Lazy — only runs when .get() or .wait() is called (on the calling thread)
std::async(std::launch::deferred, myFunc, args...);

// Let the implementation decide (default — can be either)
std::async(myFunc, args...);
```

> ⚠️ **Gotcha:** If you don't store the returned `std::future`, the destructor of the temporary future **blocks** until the task completes — defeating the purpose of async.

```cpp
// BAD — blocks immediately, defeating async
std::async(std::launch::async, computeSquare, 9); // future is discarded!

// GOOD — store the future
auto fut = std::async(std::launch::async, computeSquare, 9);
```

---

## `std::packaged_task` — Wrapping a Callable

`std::packaged_task` wraps a callable (function, lambda) so its return value is delivered via a `std::future`. It gives you more control than `std::async` (you decide when and how the task runs).

```cpp
#include <iostream>
#include <future>
#include <thread>

int add(int a, int b) { return a + b; }

int main() {
    // Wrap the function
    std::packaged_task<int(int, int)> task(add);
    std::future<int> fut = task.get_future();

    // Move the task into a thread and run it
    std::thread t(std::move(task), 10, 20);

    std::cout << "Sum: " << fut.get() << "\n"; // 30
    t.join();
    return 0;
}
```

---

## Tool Comparison

| Tool | Best For | Returns Value | Thread Management |
|---|---|---|---|
| `condition_variable` | Ongoing signaling, event loops | ❌ | Manual |
| `promise` / `future` | One-shot value transfer | ✅ | Manual |
| `std::async` | Quick one-off async tasks | ✅ | Automatic |
| `packaged_task` | Queuing tasks for a thread pool | ✅ | Manual |

---

## A Complete Pattern: Thread-Safe Queue

This combines a mutex and condition variable into a reusable pattern used in real thread pools.

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <optional>

template<typename T>
class SafeQueue {
    std::queue<T>           q;
    std::mutex              mtx;
    std::condition_variable cv;

public:
    void push(T value) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            q.push(std::move(value));
        }
        cv.notify_one(); // Tell one waiting consumer there's something new
    }

    T pop() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]{ return !q.empty(); }); // Sleep until non-empty
        T value = std::move(q.front());
        q.pop();
        return value;
    }
};
```

---

## Key Takeaways

- Never busy-wait. Use `condition_variable` to sleep a thread until work arrives.
- **Always** use the predicate overload of `cv.wait()` to guard against spurious wakeups.
- `promise`/`future` is the cleanest way to return a single value from a background thread.
- `std::async` is your first choice for simple asynchronous work — it hides boilerplate.
- Always store the `future` returned by `std::async`, otherwise it blocks unexpectedly.

---

*Previous: [Pillar 2 — Synchronization & Mutual Exclusion](pillar2_synchronization.md)*  
*Next: [Pillar 4 — Atomic Operations](pillar4_atomic_operations.md)*
