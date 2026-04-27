# Pillar 2: Synchronization & Mutual Exclusion — The Safety Pillar

> **Goal:** Prevent data corruption by ensuring only one thread accesses shared data at a time. This is where you defeat **Race Conditions**.

---

## What is a Race Condition?

A **race condition** occurs when two or more threads read and write the same shared data simultaneously, and the final result depends on the unpredictable scheduling order of those threads.

### The Classic Example — A Broken Counter

```cpp
#include <iostream>
#include <thread>

int counter = 0; // Shared data — DANGER ZONE

void increment() {
    for (int i = 0; i < 100000; ++i) {
        counter++; // This is NOT one instruction — it's READ, ADD, WRITE
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();

    // Expected: 200000
    // Actual:   some random number like 147832 — different every run!
    std::cout << "Counter: " << counter << "\n";
    return 0;
}
```

The line `counter++` compiles to three CPU steps: **Read** → **Increment** → **Write**. When two threads interleave those steps, increments get silently lost.

---

## `std::mutex` — The Lock

A **mutex** (Mutual Exclusion) is a lock that guarantees only one thread can be inside a protected section of code at a time. All other threads that try to lock it will **block** (wait) until the lock is released.

### The Mutex Rules

1. Only one thread holds the lock at any time.
2. A thread that can't acquire the lock is put to sleep by the OS.
3. When the lock is released, the OS wakes one of the waiting threads.

### Raw Usage (for understanding only — don't do this in real code)

```cpp
#include <iostream>
#include <thread>
#include <mutex>

int counter = 0;
std::mutex mtx; // The lock object — one per shared resource

void increment() {
    for (int i = 0; i < 100000; ++i) {
        mtx.lock();   // Acquire the lock
        counter++;    // Only one thread here at a time
        mtx.unlock(); // Release the lock
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "Counter: " << counter << "\n"; // Always 200000 now
    return 0;
}
```

> ⚠️ **Never use raw `.lock()` / `.unlock()` in real code.** If an exception is thrown between lock and unlock, the mutex is **never released** — every other thread waits forever. This is called a **Deadlock**.

---

## RAII Locking — The Correct Way

**RAII** (Resource Acquisition Is Initialization) means tying the lifetime of a lock to the lifetime of an object. When the object is destroyed (goes out of scope), the lock is automatically released — even if an exception is thrown.

---

### `std::lock_guard` — Simple, Scoped Lock

`lock_guard` locks the mutex when created and unlocks it when destroyed. It cannot be manually unlocked early and cannot be moved.

```cpp
#include <iostream>
#include <thread>
#include <mutex>

int counter = 0;
std::mutex mtx;

void safeIncrement() {
    for (int i = 0; i < 100000; ++i) {
        std::lock_guard<std::mutex> lock(mtx); // Locks here
        counter++;
    } // lock is destroyed here — mutex automatically unlocked
}

int main() {
    std::thread t1(safeIncrement);
    std::thread t2(safeIncrement);
    t1.join();
    t2.join();
    std::cout << "Counter: " << counter << "\n"; // Always 200000
    return 0;
}
```

**Use `lock_guard` when:** You want to lock for the entire scope of a block and need nothing more.

---

### `std::unique_lock` — Flexible Lock

`unique_lock` does everything `lock_guard` does, plus:
- Can be **manually unlocked and re-locked**.
- Can be **moved** (transferred between scopes).
- Required by `std::condition_variable` (Pillar 3).

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;
int sharedData = 0;

void flexibleWorker() {
    std::unique_lock<std::mutex> lock(mtx); // Locks here

    sharedData = 42;

    lock.unlock(); // Manually release early — other threads can run now

    // Do expensive non-shared work here (no lock needed)
    std::this_thread::sleep_for(std::chrono::milliseconds(10));

    lock.lock(); // Re-acquire when needed again
    sharedData += 1;
} // Unlocks automatically on destruction

int main() {
    std::thread t(flexibleWorker);
    t.join();
    std::cout << "Data: " << sharedData << "\n"; // 43
    return 0;
}
```

**Use `unique_lock` when:** You need to unlock early, transfer the lock, or use condition variables.

---

### `std::scoped_lock` (C++17) — The Gold Standard for Multiple Mutexes

When you need to lock **more than one mutex at the same time**, locking them one-by-one risks a **Deadlock** (Thread A locks mutex 1 and waits for mutex 2; Thread B locks mutex 2 and waits for mutex 1 — both wait forever).

`scoped_lock` uses a deadlock-avoidance algorithm to lock all given mutexes safely in one atomic step.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <string>

struct BankAccount {
    std::mutex mtx;
    double balance;
    std::string name;
};

void transfer(BankAccount& from, BankAccount& to, double amount) {
    // Lock BOTH mutexes at once — no deadlock possible
    std::scoped_lock lock(from.mtx, to.mtx);

    if (from.balance >= amount) {
        from.balance -= amount;
        to.balance   += amount;
        std::cout << "Transferred " << amount
                  << " from " << from.name
                  << " to "   << to.name << "\n";
    }
} // Both mutexes unlocked automatically here

int main() {
    BankAccount alice{.balance = 1000.0, .name = "Alice"};
    BankAccount bob  {.balance =  500.0, .name = "Bob"};

    // These two threads transfer in opposite directions simultaneously
    // With scoped_lock, this is safe. Without it, it would deadlock.
    std::thread t1(transfer, std::ref(alice), std::ref(bob),   200.0);
    std::thread t2(transfer, std::ref(bob),   std::ref(alice), 100.0);

    t1.join();
    t2.join();

    std::cout << "Alice: " << alice.balance << "\n"; // 900
    std::cout << "Bob:   " << bob.balance   << "\n"; // 600
    return 0;
}
```

---

## `std::shared_mutex` (C++17) — Reader/Writer Lock

Sometimes many threads only **read** data and occasional threads **write** it. Using a regular mutex makes all readers wait for each other unnecessarily. `shared_mutex` solves this:

- **Multiple readers** can hold the lock simultaneously.
- **Only one writer** can hold the lock, and only when no readers are active.

```cpp
#include <iostream>
#include <thread>
#include <shared_mutex>
#include <vector>
#include <string>

std::shared_mutex rwMutex;
std::string sharedConfig = "initial_value";

// Multiple reader threads can run this concurrently
void readConfig(int id) {
    std::shared_lock<std::shared_mutex> lock(rwMutex); // Read lock
    std::cout << "Reader " << id << " sees: " << sharedConfig << "\n";
}

// Only one writer thread at a time
void writeConfig(const std::string& newValue) {
    std::unique_lock<std::shared_mutex> lock(rwMutex); // Exclusive write lock
    sharedConfig = newValue;
    std::cout << "Writer updated config to: " << newValue << "\n";
}

int main() {
    std::vector<std::thread> threads;

    // Launch 5 readers and 1 writer
    for (int i = 0; i < 5; ++i) threads.emplace_back(readConfig, i);
    threads.emplace_back(writeConfig, "new_value");
    for (int i = 5; i < 8; ++i) threads.emplace_back(readConfig, i);

    for (auto& t : threads) t.join();
    return 0;
}
```

---

## Lock Comparison Table

| Type | Header | Can Unlock Early | Multi-Mutex | Best For |
|---|---|---|---|---|
| `std::lock_guard` | `<mutex>` | ❌ | ❌ | Simple single-scope locking |
| `std::unique_lock` | `<mutex>` | ✅ | ❌ | Condition variables, flexible control |
| `std::scoped_lock` | `<mutex>` | ❌ | ✅ | Locking multiple mutexes safely |
| `std::shared_lock` | `<shared_mutex>` | ✅ | ❌ | Read-heavy scenarios |

---

## Understanding Deadlock

A **deadlock** is when two or more threads permanently block each other, each waiting for a resource the other holds.

```
Thread A:  holds Mutex 1, waiting for Mutex 2 ──┐
Thread B:  holds Mutex 2, waiting for Mutex 1 ──┘
                    → Neither can proceed. Ever.
```

### ❌ Code that deadlocks

```cpp
std::mutex m1, m2;

void threadA() {
    std::lock_guard lk1(m1); // locks m1
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard lk2(m2); // waits for m2 — DEADLOCK if B has it
}

void threadB() {
    std::lock_guard lk1(m2); // locks m2
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard lk2(m1); // waits for m1 — DEADLOCK if A has it
}
```

### ✅ Fixed with `scoped_lock`

```cpp
void threadA() {
    std::scoped_lock lock(m1, m2); // Both at once, order doesn't matter
}

void threadB() {
    std::scoped_lock lock(m2, m1); // Same result — deadlock-safe
}
```

---

## The 4 Rules of Deadlock Prevention

1. **Lock multiple mutexes with `scoped_lock`**, never one by one.
2. **Don't hold a lock while calling unknown code** (callbacks, virtual functions).
3. **Keep critical sections short** — unlock as soon as you're done.
4. **Never wait for another thread while holding a lock** (that's what condition variables are for).

---

## Key Takeaways

- A race condition corrupts data silently — always protect shared data.
- Use `std::mutex` with **RAII wrappers**, never raw `.lock()` / `.unlock()`.
- `lock_guard` for simple scopes, `unique_lock` for flexibility, `scoped_lock` for multiple mutexes.
- Deadlocks are preventable — `scoped_lock` is your primary weapon against them.
- Read-heavy workloads benefit from `shared_mutex`.

---

*Previous: [Pillar 1 — Thread Management](pillar1_thread_management.md)*  
*Next: [Pillar 3 — Communication & Coordination](pillar3_communication.md)*
