# Use std::atomic<std::shared_ptr<T>> (C++20) for safe concurrent pointer updates

**Category:** Concurrency and Parallelism  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/memory/shared_ptr/atomic2>  

---

## Topic Overview

C++20 provides `std::atomic<std::shared_ptr<T>>` - a proper atomic specialization for shared pointers. Before C++20, concurrent access to `shared_ptr` required either a mutex or the deprecated free functions `std::atomic_load`/`std::atomic_store`. The new specialization makes the atomic nature part of the type itself, which is a much cleaner design.

### The Problem

Here is why you cannot simply share a `shared_ptr` between threads without synchronization. A `shared_ptr` is not a single value - it is two pointers bundled together, and updating them cannot be done atomically with ordinary assignment:

```cpp
Thread A: auto p = global_ptr;     // copies shared_ptr (reads ref count + ptr)
Thread B: global_ptr = new_ptr;    // modifies shared_ptr (writes ref count + ptr)

shared_ptr has TWO members: pointer + control block pointer.
Updating both is NOT atomic -> data race -> UB!

C++20 fix: std::atomic<shared_ptr<T>> makes load/store/CAS atomic.
```

The reason this trips people up is that the control block's reference count is already atomic internally - but the `shared_ptr` object itself has two separate pointers, and reading or writing those two pointers is not one atomic operation. Even just copying a `shared_ptr` while another thread replaces it is a data race.

### API Summary

| Operation | Description |
| --- | --- |
| `a.load()` | Returns a copy of the stored `shared_ptr` (atomic) |
| `a.store(p)` | Replaces stored pointer with `p` (atomic) |
| `a.exchange(p)` | Swaps and returns old pointer (atomic) |
| `a.compare_exchange_weak/strong(expected, desired)` | CAS on the shared_ptr |
| `a.is_lock_free()` | Check if implementation uses locks (usually `false`) |

---

## Self-Assessment

### Q1: Use atomic shared_ptr to implement a lock-free cache with concurrent read and update

**Answer:**

The read-copy-update pattern shown here lets readers grab a consistent snapshot while writers build a new copy and publish it - all without a mutex.

```cpp
#include <atomic>
#include <memory>
#include <map>
#include <string>
#include <thread>
#include <vector>
#include <iostream>

// Read-copy-update pattern using atomic<shared_ptr>
class Config {
    using DataMap = std::map<std::string, std::string>;
    std::atomic<std::shared_ptr<const DataMap>> data_;

public:
    Config() : data_(std::make_shared<const DataMap>()) {}

    // READ: lock-free, returns a snapshot
    std::string get(const std::string& key) const {
        auto snapshot = data_.load(); // atomic copy of shared_ptr
        auto it = snapshot->find(key);
        return (it != snapshot->end()) ? it->second : "";
        // snapshot keeps the data alive even if another thread updates data_
    }

    // WRITE: creates a new copy, atomically swaps
    void set(const std::string& key, const std::string& value) {
        auto old = data_.load();
        std::shared_ptr<const DataMap> desired;
        do {
            // Copy current map + add/update entry
            auto updated = std::make_shared<DataMap>(*old);
            (*updated)[key] = value;
            desired = std::move(updated);
            // CAS: replace only if nobody else updated since we read
        } while (!data_.compare_exchange_weak(old, desired));
    }

    size_t size() const {
        return data_.load()->size();
    }
};

int main() {
    Config config;
    constexpr int WRITERS = 4;
    constexpr int READERS = 4;
    constexpr int OPS = 10'000;

    std::vector<std::thread> threads;

    // Writer threads: update config entries
    for (int w = 0; w < WRITERS; ++w) {
        threads.emplace_back([&, w] {
            for (int i = 0; i < OPS; ++i) {
                config.set("key" + std::to_string(w),
                           "val" + std::to_string(i));
            }
        });
    }

    // Reader threads: read config entries (never see torn state)
    std::atomic<int> reads{0};
    for (int r = 0; r < READERS; ++r) {
        threads.emplace_back([&, r] {
            for (int i = 0; i < OPS; ++i) {
                auto val = config.get("key" + std::to_string(r % WRITERS));
                // 'val' is always consistent - either old or new, never torn
                reads.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }

    for (auto& t : threads) t.join();
    std::cout << "Reads: " << reads.load() << " Config size: "
              << config.size() << "\n";
    // Output: Reads: 40000 Config size: 4
    // No crashes, no torn reads, no data races
}
```

`atomic<shared_ptr<const DataMap>>` allows readers to atomically grab a snapshot while writers create new copies and swap atomically via CAS. Readers always see a consistent state, and old data stays alive until the last reader's `shared_ptr` is destroyed - no manual memory management needed.

### Q2: Show the pre-C++20 equivalents (std::atomic_load, std::atomic_store) and why they were problematic

**Answer:**

The core issue with the old approach was that nothing in the type system told you a `shared_ptr` was being shared concurrently. You had to remember to use the special free functions every time - and forgetting was a silent data race.

```cpp
#include <atomic>
#include <memory>
#include <thread>
#include <iostream>

// === PRE-C++20: Free functions (deprecated in C++20) ===
namespace pre_cpp20 {
    std::shared_ptr<int> global_ptr = std::make_shared<int>(0);

    void reader() {
        // Must use atomic_load - plain copy is a data race!
        auto local = std::atomic_load(&global_ptr);
        std::cout << "Read: " << *local << "\n";
    }

    void writer(int val) {
        auto new_ptr = std::make_shared<int>(val);
        std::atomic_store(&global_ptr, new_ptr);
    }

    // Problems with the free-function approach:
    // 1. EASY TO FORGET: nothing prevents "auto p = global_ptr;" (data race!)
    // 2. No compile-time enforcement: looks like a normal shared_ptr
    // 3. Different syntax from other atomics (free functions vs methods)
    // 4. No CAS operations in early implementations
}

// === C++20: std::atomic<shared_ptr<T>> ===
namespace cpp20 {
    std::atomic<std::shared_ptr<int>> global_ptr{std::make_shared<int>(0)};

    void reader() {
        auto local = global_ptr.load(); // .load() makes intent clear
        std::cout << "Read: " << *local << "\n";
    }

    void writer(int val) {
        global_ptr.store(std::make_shared<int>(val));
    }

    // Advantages:
    // 1. TYPE SAFETY: can't accidentally do "auto p = global_ptr;" on atomic
    // 2. Consistent API: .load(), .store(), .compare_exchange_*()
    // 3. CAS support: enables lock-free algorithms on shared_ptrs
    // 4. Clear ownership semantics
}

int main() {
    // === Demonstrate the data race with plain shared_ptr ===
    {
        std::shared_ptr<int> ptr = std::make_shared<int>(0);

        // This is a DATA RACE:
        // std::thread t1([&] { ptr = std::make_shared<int>(1); }); // write
        // std::thread t2([&] { auto p = ptr; });                    // read
        // Two threads accessing non-atomic shared_ptr -> UB!

        // Even just READING shared_ptr from two threads while a third writes
        // is UB - the reference count update is NOT atomic at the shared_ptr
        // object level (the control block's ref count is atomic, but the
        // shared_ptr object itself has two pointers that aren't).
    }

    // === C++20 atomic shared_ptr: safe ===
    {
        std::atomic<std::shared_ptr<int>> aptr{std::make_shared<int>(0)};

        std::thread t1([&] { aptr.store(std::make_shared<int>(42)); });
        std::thread t2([&] { auto p = aptr.load(); std::cout << *p << "\n"; });
        t1.join();
        t2.join();
        // Always safe: prints 0 or 42, never crashes
    }
}
```

With the C++20 version, the compiler simply won't let you do `auto p = global_ptr` on an `atomic<shared_ptr<int>>` - the deleted copy constructor stops you at compile time. That is a huge improvement over free functions that looked exactly like normal function calls.

### Q3: Explain why `std::atomic<shared_ptr<T>>` being lock-free is not guaranteed

**Answer:**

This is worth understanding deeply, because people sometimes assume "atomic" means "lock-free." It does not. "Atomic" means the operation appears instantaneous to other threads. Whether that is achieved with hardware instructions or an internal lock is a separate question.

```cpp
#include <atomic>
#include <memory>
#include <iostream>

int main() {
    std::atomic<std::shared_ptr<int>> aptr{std::make_shared<int>(42)};

    std::cout << std::boolalpha;
    std::cout << "atomic<shared_ptr<int>> is lock-free: "
              << aptr.is_lock_free() << "\n";
    // Output (most implementations): false

    // === WHY IT'S NOT LOCK-FREE ===
    //
    // shared_ptr is a complex object:
    //   struct shared_ptr {
    //       T* ptr_;              // 8 bytes
    //       control_block* ctrl_; // 8 bytes
    //   };
    //   // Total: 16 bytes
    //
    // To atomically update BOTH fields simultaneously, you need:
    //
    // Option 1: 128-bit CAS (cmpxchg16b on x86)
    //   -> Available on most x86-64, but NOT on all platforms
    //   -> ARM doesn't have 128-bit CAS
    //   -> Even on x86: requires 16-byte alignment
    //
    // Option 2: Use a mutex internally
    //   -> This is what most implementations do!
    //   -> libstdc++, libc++, MSVC all use internal spinlocks
    //   -> Still thread-safe, but NOT lock-free
    //
    // Additionally, shared_ptr operations affect the CONTROL BLOCK:
    //   - load() increments reference count
    //   - store() decrements old ref count, may deallocate
    //   - These are additional atomic operations beyond just the pointer swap
    //
    // === COMPARISON ===
    //
    // Type                         | Lock-free? | Size
    // atomic<int>                  | Yes        | 4 bytes
    // atomic<long long>            | Yes        | 8 bytes
    // atomic<void*>                | Yes        | 8 bytes
    // atomic<shared_ptr<T>>        | Usually No | 16 bytes + ref count
    //
    // atomic<shared_ptr> is still USEFUL even when not lock-free:
    // - Correct by construction (type system prevents races)
    // - Internal lock is highly optimized (spinlock, not mutex)
    // - Much simpler than manual mutex + shared_ptr
    // - CAS operations enable algorithms impossible with mutex
    //
    // For truly lock-free shared pointers, you need:
    // - Hazard pointers (C++26)
    // - Epoch-based reclamation
    // - Custom reference counting with raw atomics

    // === Lock-free check for comparison ===
    std::atomic<int> ai{0};
    std::atomic<void*> ap{nullptr};
    std::cout << "atomic<int> lock-free:   " << ai.is_lock_free() << "\n";
    std::cout << "atomic<void*> lock-free: " << ap.is_lock_free() << "\n";
    // Output:
    // atomic<int> lock-free:   true
    // atomic<void*> lock-free: true
}
```

The key takeaway is that `atomic<shared_ptr>` being internally locked is not a dealbreaker. The internal spinlock is held only for the brief instant needed to swap the two pointers - far shorter than a typical mutex-protected critical section. For most real workloads, particularly read-heavy ones, it is much faster than a full `mutex` + `shared_ptr` combination.

---

## Notes

- **Performance:** `atomic<shared_ptr>` with internal locks is still much faster than `mutex` + `shared_ptr` for read-heavy workloads because the internal spinlock is held only for the brief pointer swap.
- **RCU pattern:** `atomic<shared_ptr<const T>>` naturally implements read-copy-update. Readers get a snapshot, writers create new copies and atomically publish them.
- **`std::atomic_is_lock_free(const shared_ptr<T>*)`** (deprecated in C++20) tested the free-function variant.
- **libstdc++ implementation (GCC):** Uses a fixed array of spinlocks indexed by the address of the `atomic<shared_ptr>` - potential for hash collisions under high contention.
- **Alternative: `atomic<T*>` + manual reference counting** is truly lock-free but requires correct memory reclamation.
- Compile with `-std=c++20 -O2 -pthread`.
