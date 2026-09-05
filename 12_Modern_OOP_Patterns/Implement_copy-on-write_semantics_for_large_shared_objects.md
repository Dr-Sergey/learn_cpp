# Implement copy-on-write semantics for large shared objects

**Category:** Modern OOP Patterns  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/shared_ptr>  

---

## Topic Overview

**Copy-on-Write (COW)** is an optimization strategy: multiple owners share a single copy of data. A deep copy is made only when one owner needs to *modify* the data. Until then, everyone reads from the same allocation.

The motivation is simple. If you have a large object that's mostly read and only occasionally modified, paying for a full deep copy on every assignment is wasteful. COW defers that cost until it's actually necessary.

### COW State Diagram

Here's the lifecycle of a shared COW string. Notice that the copy is cheap until someone mutates:

```cpp
Initial:  a = COW("Hello World - a big expensive string")

          a --+
              v
         +───────────+
         | ref_count=1|
         | "Hello..." |
         +───────────+

After:    b = a;  (cheap: increment ref_count)

          a --+
              v
         +───────────+
         | ref_count=2|  <- shared, no copy yet
         | "Hello..." |
         +───────────+
              ^
          b --+

After:    b.mutate("World");  (copy triggered!)

          a --+           b --+
              v               v
         +───────────+  +───────────+
         | ref_count=1|  | ref_count=1|
         | "Hello..." |  | "World"   |
         +───────────+  +───────────+
                         <- deep copy only at mutation
```

### When to Use COW

| Scenario | COW Helpful? | Why |
| --- | --- | --- |
| Large objects, mostly read | **Yes** | Avoids unnecessary copies |
| Small objects (int, point) | No | Copy is cheaper than indirection |
| Frequent mutations | No | Every mutate triggers a copy anyway |
| Multi-threaded sharing | **Risky** | Needs atomic ref-count + careful sync |

---

## Self-Assessment

### Q1: Use `shared_ptr` + `unique()` check to implement lazy copying: share until mutation

The core of COW is the `detach()` method. Every mutating operation calls it first. If the data is shared (ref count > 1), detach makes a private copy before the mutation proceeds. If it's already private (ref count == 1), the data is already exclusively owned and no copy is needed.

**Solution - COW String:**

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

class CowString {
    std::shared_ptr<std::string> data_;

    // Detach: if shared, make a private copy before mutating
    void detach() {
        if (data_.use_count() > 1) {
            std::cout << "  [COW] Detaching - deep copy triggered\n";
            data_ = std::make_shared<std::string>(*data_);
        }
    }

public:
    // Construct
    CowString(const std::string& s = "")
        : data_(std::make_shared<std::string>(s)) {}

    // Read access - no copy needed
    const std::string& read() const { return *data_; }
    size_t length() const { return data_->size(); }

    // Write access - detach if shared
    void write(const std::string& s) {
        detach();
        *data_ = s;
    }

    void append(const std::string& s) {
        detach();
        *data_ += s;
    }

    // Diagnostics
    long use_count() const { return data_.use_count(); }
    const void* data_ptr() const { return data_.get(); }
};

int main() {
    CowString a("Hello, this is a large string payload");
    std::cout << "a: \"" << a.read() << "\"\n";
    std::cout << "a ref_count=" << a.use_count()
              << " ptr=" << a.data_ptr() << "\n\n";

    // Copy is CHEAP - just shared_ptr copy (ref_count++)
    CowString b = a;
    std::cout << "After b = a:\n";
    std::cout << "  a ref_count=" << a.use_count()
              << " ptr=" << a.data_ptr() << "\n";
    std::cout << "  b ref_count=" << b.use_count()
              << " ptr=" << b.data_ptr() << "\n";
    std::cout << "  Same pointer? " << (a.data_ptr() == b.data_ptr() ? "YES" : "NO")
              << "\n\n";

    // Mutation triggers deep copy
    std::cout << "Mutating b:\n";
    b.write("Modified string");
    std::cout << "  a: \"" << a.read() << "\" ptr=" << a.data_ptr() << "\n";
    std::cout << "  b: \"" << b.read() << "\" ptr=" << b.data_ptr() << "\n";
    std::cout << "  Same pointer? " << (a.data_ptr() == b.data_ptr() ? "YES" : "NO")
              << "\n";
}
// Expected output:
//   a: "Hello, this is a large string payload"
//   a ref_count=1 ptr=0x...
//
//   After b = a:
//     a ref_count=2 ptr=0x1234
//     b ref_count=2 ptr=0x1234    <- SAME pointer! No copy.
//     Same pointer? YES
//
//   Mutating b:
//     [COW] Detaching - deep copy triggered
//     a: "Hello, this is a large string payload" ptr=0x1234
//     b: "Modified string" ptr=0x5678   <- different pointer now
//     Same pointer? NO
```

**Note:** `shared_ptr::unique()` was deprecated in C++17 and removed in C++20. Use `use_count() == 1` or `use_count() > 1` instead.

---

### Q2: Show how COW avoids unnecessary deep copies when multiple owners only read

This example makes the performance difference concrete. Creating 100 "copies" costs essentially nothing when they're all just reading the same data.

**Solution - Measuring Copy Avoidance:**

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>
#include <chrono>

// Simulates a large config object
struct LargeConfig {
    std::vector<std::string> entries;

    LargeConfig() {
        entries.reserve(10000);
        for (int i = 0; i < 10000; ++i)
            entries.push_back("entry_" + std::to_string(i));
    }

    // Expensive deep copy
    LargeConfig(const LargeConfig&) = default;
};

class CowConfig {
    std::shared_ptr<LargeConfig> data_;

    void detach() {
        if (data_.use_count() > 1)
            data_ = std::make_shared<LargeConfig>(*data_);
    }

public:
    CowConfig() : data_(std::make_shared<LargeConfig>()) {}

    // Read-only access - never copies
    const std::string& get(size_t i) const { return data_->entries[i]; }
    size_t size() const { return data_->entries.size(); }

    // Write access - copies only when shared
    void set(size_t i, const std::string& val) {
        detach();
        data_->entries[i] = val;
    }

    long use_count() const { return data_.use_count(); }
};

int main() {
    CowConfig original;  // 10,000 entries
    std::cout << "Original created: " << original.size() << " entries\n\n";

    // Create 100 "copies" - all cheap (just shared_ptr copies)
    auto start = std::chrono::high_resolution_clock::now();
    std::vector<CowConfig> readers(100, original);
    auto end = std::chrono::high_resolution_clock::now();

    auto copy_us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    std::cout << "100 COW copies created in " << copy_us << " us\n";
    std::cout << "Ref count: " << original.use_count() << "\n";
    std::cout << "All reading from same data - zero deep copies\n\n";

    // Only one reader mutates - only that one triggers a deep copy
    std::cout << "Mutating readers[50]...\n";
    start = std::chrono::high_resolution_clock::now();
    readers[50].set(0, "MODIFIED");
    end = std::chrono::high_resolution_clock::now();

    auto mut_us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    std::cout << "Mutation took " << mut_us << " us (one deep copy)\n";
    std::cout << "Original entry[0]: " << original.get(0) << "\n";
    std::cout << "Modified entry[0]: " << readers[50].get(0) << "\n";
    std::cout << "Other readers[0]:  " << readers[0].get(0) << " (unchanged)\n";
}
// Expected output:
//   Original created: 10000 entries
//
//   100 COW copies created in ~X us (very fast, no deep copies)
//   Ref count: 101
//   All reading from same data - zero deep copies
//
//   Mutating readers[50]...
//   Mutation took ~Y us (one deep copy of 10000 entries)
//   Original entry[0]: entry_0
//   Modified entry[0]: MODIFIED
//   Other readers[0]:  entry_0 (unchanged)
```

The ref count reaches 101 (1 original + 100 readers) and only `readers[50]` breaks away from the pack when it mutates. The other 99 readers and the original remain on the same allocation.

---

### Q3: Explain why COW is broken in multi-threaded code without careful synchronization

This is a subtle bug that's easy to miss. The issue isn't with `shared_ptr`'s reference count - that's already atomic. The problem is that `detach()` is a *check-then-act* sequence, and that sequence is not atomic. Two threads can each check `use_count() > 1`, both decide they need to detach, and both make copies - which is wasteful. Worse scenarios can cause actual correctness bugs:

**The Race Condition:**

```cpp
Thread A:                    Thread B:
─────────                    ─────────
read use_count() -> 2        read use_count() -> 2
  (shared, need detach)        (shared, need detach)

detach(): make copy          detach(): make copy
  data_ = make_shared(copy)    data_ = make_shared(copy)

  now both have private copies... BUT:

What if Thread A reads use_count() BETWEEN Thread B's read and detach?

Timeline problem:
  B reads use_count() = 2   <- shared
  A reads use_count() = 2   <- shared
  B detaches -> use_count on original drops to 1
  A sees use_count was 2, but now original has count 1
  A detaches too -> unnecessary copy (harmless but wasteful)

WORSE scenario (without atomic ref-count):
  A reads ref_count -> 2
  B decrements ref_count -> 1 (non-atomic)
  A decrements ref_count -> 0 (non-atomic)  <- DOUBLE DECREMENT BUG
  Memory freed while B still using it! -> UNDEFINED BEHAVIOR
```

The good news is that `shared_ptr`'s reference count *is* atomic in C++, so the double-decrement bug doesn't happen with it. But the check-then-act problem remains, and the fix requires a mutex around the whole detach operation:

**Solution - Thread-Safe COW:**

```cpp
#include <iostream>
#include <memory>
#include <mutex>
#include <string>
#include <thread>
#include <vector>

class ThreadSafeCowString {
    std::shared_ptr<std::string> data_;
    mutable std::mutex mutex_;  // protects detach logic

    void detach() {
        // Lock ensures use_count check + copy is atomic
        // Note: shared_ptr ref-count is already atomic in C++,
        // but the check-then-act sequence is NOT atomic!
        if (data_.use_count() > 1) {
            data_ = std::make_shared<std::string>(*data_);
        }
    }

public:
    ThreadSafeCowString(const std::string& s = "")
        : data_(std::make_shared<std::string>(s)) {}

    // Thread-safe copy constructor
    ThreadSafeCowString(const ThreadSafeCowString& other) {
        std::lock_guard lock(other.mutex_);
        data_ = other.data_;
    }

    // Thread-safe read
    std::string read() const {
        std::lock_guard lock(mutex_);
        return *data_;  // return by value for thread safety
    }

    // Thread-safe write
    void write(const std::string& s) {
        std::lock_guard lock(mutex_);
        detach();
        *data_ = s;
    }
};

int main() {
    ThreadSafeCowString shared_str("Original data");

    std::vector<std::jthread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back([&shared_str, i] {
            // Multiple threads reading - shares one copy
            std::cout << "[" << i << "] Read: " << shared_str.read() << "\n";
        });
    }

    threads.clear();  // join all readers

    // One writer
    shared_str.write("Modified by writer");
    std::cout << "After write: " << shared_str.read() << "\n";
}
```

Once you add a mutex to protect the check-then-act, much of the performance win from COW evaporates - you're paying for synchronization on every read and write. This is exactly why `std::string` abandoned COW in C++11.

**Why `std::string` abandoned COW (C++11):**

| Issue | Explanation |
| --- | --- |
| **Iterator invalidation** | COW detach on `s[i]` for non-const would invalidate iterators held by other owners |
| **Thread safety cost** | Atomic ref-count + mutex for detach eliminates the performance win |
| **Move semantics** | C++11 move semantics make deep copies rare anyway |
| **ABI break** | GCC's libstdc++ had COW strings pre-C++11; switching was an ABI change |
| **Standard requirement** | C++11 §21.4.1/6 requires `operator[]` not to invalidate references - incompatible with COW |

---

## Notes

- **COW was the dominant `std::string` implementation** (GCC libstdc++) until C++11 mandated non-COW. SSO (Small String Optimization) replaced it.
- **Qt still uses COW** for `QString`, `QByteArray`, `QList`, and other container types - it's called "implicit sharing".
- **`shared_ptr` ref-count is atomic** in C++, so `use_count()` itself is thread-safe, but the *read-then-act* pattern (`if (use_count() > 1) detach()`) is NOT atomic without a mutex.
- **Immutable data structures** (functional programming style) avoid the COW problem entirely - no mutation means no detach needed.
- **`std::shared_ptr::use_count()` is approximate** in concurrent code - it's intended for debugging, not for logic. For production COW, protect with a mutex.
