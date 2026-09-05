# Understand memory fences and std::atomic_thread_fence (Part 2 - Advanced)

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence>  

---

## Topic Overview

This is **Part 2** focusing on advanced fence patterns: replacing per-operation ordering with standalone fences, the subtle semantic differences, and practical patterns where fences outperform per-variable ordering.

The core idea to keep in mind throughout this section: a fence is a *global* ordering constraint on a thread's memory operations, while a per-variable memory order is a *local* constraint attached to one specific atomic. Fences are more powerful, but also trickier to reason about.

### Fence Placement Rules

The diagram below shows where you put fences relative to your stores and loads, and what those fences guarantee:

```cpp
Release fence:                        Acquire fence:
┌────────────┐                        ┌────────────┐
│ store A     │                       │ load X      │
│ store B     │  <- these are         │ load Y      │  <- relaxed loads
│ store C     │    ordered before      │ load Z      │    that triggered
│─────────────│                       │─────────────│
│ RELEASE     │  <- fence             │ ACQUIRE     │  <- fence
│─────────────│                       │─────────────│
│ store flag  │  <- notification      │ read A      │  <- guaranteed to
│ (relaxed)   │                       │ read B      │    see producer's
└────────────┘                        │ read C      │    stores
                                      └────────────┘
```

### Fence Pairing for Synchronization

For synchronization to actually occur, a release fence in one thread must pair with an acquire fence in another thread, through a relaxed atomic that the second thread reads. The pairing is what creates the happens-before edge:

```cpp
Thread A (producer):              Thread B (consumer):
  write data                        read flag (relaxed) -> sees 1
  RELEASE FENCE                     ACQUIRE FENCE
  flag.store(1, relaxed)            read data -> guaranteed visible
```

The reason the acquire fence must come *after* the flag load is that the fence orders everything *below* it relative to what came *above* it. If you placed the acquire fence before the flag load, it would not cover the data reads that come after.

---

## Self-Assessment

### Q1: Show a case where per-operation memory orders can be relaxed but a fence enforces ordering

**Answer:**

Here all individual atomic operations use `memory_order_relaxed` - the cheapest possible ordering. A single release fence and a single acquire fence handle all the synchronization. Notice how the fence neatly separates "data work" from "synchronization":

```cpp
#include <atomic>
#include <thread>
#include <cassert>
#include <iostream>
#include <string>

// Scenario: publish 3 pieces of data + a "ready" flag
// Using per-variable ordering, you'd need release on the flag store
// and acquire on the flag load. With fences, ALL operations are relaxed.

std::atomic<bool> ready{false};
std::atomic<int> x{0};
std::atomic<int> y{0};
std::string message; // non-atomic

void producer() {
    // All stores use relaxed - no per-operation ordering
    x.store(42, std::memory_order_relaxed);
    y.store(99, std::memory_order_relaxed);
    message = "hello from producer";

    // ONE release fence orders ALL stores above
    std::atomic_thread_fence(std::memory_order_release);

    ready.store(true, std::memory_order_relaxed);
}

void consumer() {
    while (!ready.load(std::memory_order_relaxed))
        ; // busy wait

    // ONE acquire fence orders ALL loads below
    std::atomic_thread_fence(std::memory_order_acquire);

    // All three reads see the producer's values
    assert(x.load(std::memory_order_relaxed) == 42);
    assert(y.load(std::memory_order_relaxed) == 99);
    assert(message == "hello from producer");
    std::cout << "x=" << x.load(std::memory_order_relaxed)
              << " y=" << y.load(std::memory_order_relaxed)
              << " msg=" << message << "\n";
    // Output: x=42 y=99 msg=hello from producer
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();

    // === COMPARISON: per-operation approach ===
    // To achieve the same without fences:
    //   x.store(42, relaxed);      <- still relaxed, ok
    //   y.store(99, relaxed);      <- still relaxed, ok
    //   message = "hello";         <- non-atomic, needs ordering
    //   ready.store(true, RELEASE); <- must be release
    //
    //   while(!ready.load(ACQUIRE)); <- must be acquire
    //   ... reads are safe ...
    //
    // Both approaches are correct. The fence version:
    // + Keeps atomic operations simple (all relaxed)
    // + Clear separation of "data writes" from "synchronization"
    // - Slightly harder to reason about
    // - On x86, no performance difference (acquire/release are free)
}
```

**Explanation:** With fences, every individual atomic operation uses `memory_order_relaxed` - the cheapest option. The release fence batches all prior stores, and the acquire fence batches all subsequent loads. This is semantically equivalent to using release on the flag store and acquire on the flag load, but scales better when many variables need ordering.

### Q2: Explain the difference between a fence and a memory_order on an atomic operation

**Answer:**

Here is the distinction expressed in code. First the conceptual summary, then a case where per-variable ordering is genuinely insufficient and a fence is the right tool:

```cpp
SEMANTIC DIFFERENCE:

Per-variable memory_order:
  x.store(1, release)  <-  "order this store relative to prior operations"

  The ordering is ANCHORED to variable x. Only operations that
  synchronize through x (via acquire load of x) are affected.

Standalone fence:
  atomic_thread_fence(release)  <-  "order ALL prior operations"

  The ordering applies to EVERYTHING above the fence, regardless
  of which variables they touch.

═══════════════════════════════════════════════════════════
PRACTICAL DIFFERENCE - multiple synchronization variables:
═══════════════════════════════════════════════════════════

Without fences (need acquire/release on EACH variable):
  // Producer:
  data = 42;
  flag1.store(true, release);  // orders data w.r.t. flag1
  flag2.store(true, release);  // orders data w.r.t. flag2

  // Consumer 1:
  if (flag1.load(acquire)) { use(data); } // ok

  // Consumer 2:
  if (flag2.load(acquire)) { use(data); } // ok

With fences (ONE fence covers ALL variables):
  // Producer:
  data = 42;
  atomic_thread_fence(release);  // orders ALL prior writes
  flag1.store(true, relaxed);
  flag2.store(true, relaxed);

  // Consumer 1:
  if (flag1.load(relaxed)) {
    atomic_thread_fence(acquire);  // orders ALL subsequent reads
    use(data); // ok
  }
```

Here is the case where per-variable ordering is strictly insufficient and a fence is the correct solution:

```cpp
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<int> a{0}, b{0};
int non_atomic = 0;

// Per-variable approach - INSUFFICIENT for this pattern:
void thread1_per_var() {
    non_atomic = 42;
    a.store(1, std::memory_order_release); // orders non_atomic before a
    // But does NOT order non_atomic before b!
    b.store(1, std::memory_order_relaxed);
}

// Fence approach - correctly orders everything:
void thread1_fence() {
    non_atomic = 42;
    std::atomic_thread_fence(std::memory_order_release);
    // Orders non_atomic before BOTH a and b
    a.store(1, std::memory_order_relaxed);
    b.store(1, std::memory_order_relaxed);
}

void thread2_fence() {
    if (b.load(std::memory_order_relaxed) == 1) {
        std::atomic_thread_fence(std::memory_order_acquire);
        // If we saw b==1, the fence guarantees non_atomic==42
        assert(non_atomic == 42);
    }
}

int main() {
    std::thread t1(thread1_fence);
    std::thread t2(thread2_fence);
    t1.join();
    t2.join();
}
```

### Q3: Demonstrate acquire fence after a relaxed load and release fence before a relaxed store

**Answer:**

Seqlocks are a perfect showcase for fence placement, because they need multiple fences in careful positions to protect multi-word data. Watch how each fence is placed to prevent a specific kind of reordering:

```cpp
#include <atomic>
#include <thread>
#include <cassert>
#include <iostream>
#include <array>

// === Pattern: Seqlock using fences ===
// A seqlock allows lock-free reads of multi-word data.
// Writer: increment sequence (odd=writing), write data, increment (even=done)
// Reader: read sequence, read data, check sequence unchanged

struct SeqLock {
    std::atomic<unsigned> seq{0};
    // Protected data (multiple fields - fences cover all of them)
    int x{0}, y{0}, z{0};

    void write(int nx, int ny, int nz) {
        unsigned s = seq.load(std::memory_order_relaxed);
        seq.store(s + 1, std::memory_order_relaxed); // odd = writing

        // RELEASE FENCE: all data writes below are ordered AFTER seq write above
        // Wait - actually we need the fence BEFORE the data writes to prevent
        // them from moving before the seq update. Let's do it correctly:
        std::atomic_thread_fence(std::memory_order_release);
        // ^ prevents x,y,z stores from being reordered before seq = s+1

        x = nx; y = ny; z = nz;

        // RELEASE FENCE: all data writes above are ordered BEFORE seq write below
        std::atomic_thread_fence(std::memory_order_release);

        seq.store(s + 2, std::memory_order_relaxed); // even = done
    }

    bool read(int& rx, int& ry, int& rz) {
        unsigned s1 = seq.load(std::memory_order_relaxed);
        if (s1 & 1) return false; // writer in progress

        // ACQUIRE FENCE: data reads below see stores before producer's release fence
        std::atomic_thread_fence(std::memory_order_acquire);

        rx = x; ry = y; rz = z;

        // ACQUIRE FENCE: ensures seq re-read below sees any concurrent write
        std::atomic_thread_fence(std::memory_order_acquire);

        unsigned s2 = seq.load(std::memory_order_relaxed);
        return s1 == s2; // true if no writer interfered
    }
};

int main() {
    SeqLock lock;
    std::atomic<bool> done{false};

    // Writer thread
    std::thread writer([&] {
        for (int i = 0; i < 1'000'000; ++i) {
            lock.write(i, i * 2, i * 3);
        }
        done.store(true, std::memory_order_release);
    });

    // Reader thread
    std::thread reader([&] {
        int good = 0, retry = 0;
        while (!done.load(std::memory_order_acquire) || retry < 100) {
            int x, y, z;
            if (lock.read(x, y, z)) {
                // Consistent snapshot: y == 2*x and z == 3*x
                assert(y == 2 * x);
                assert(z == 3 * x);
                ++good;
            } else {
                ++retry;
            }
        }
        std::cout << "Good reads: " << good << ", retries: " << retry << "\n";
    });

    writer.join();
    reader.join();
    // Output: Good reads: ~900000, retries: ~100000
    // Every successful read is a consistent snapshot - guaranteed by fences
}
```

**Explanation:** The seqlock perfectly demonstrates fence placement:

- **Release fence before relaxed store:** Ensures all data writes are visible before the sequence number update appears to readers.
- **Acquire fence after relaxed load:** Ensures all subsequent data reads see the writes that happened before the producer's release fence.
- All atomic operations use `relaxed` - the fences handle all ordering.

---

## Notes

- **Fence cost on x86:** `atomic_thread_fence(acquire)` and `atomic_thread_fence(release)` compile to nothing on x86 (already guaranteed by hardware). Only `seq_cst` emits `MFENCE`.
- **Fence cost on ARM:** `acquire` -> `DMB ISHLD`, `release` -> `DMB ISH`. Real instructions with measurable cost.
- **`atomic_signal_fence`:** Same syntax but only prevents compiler reordering (no hardware barrier). Used for signal handlers within the same thread.
- **Fence + relaxed vs per-variable ordering:** On x86, identical code generation. On ARM, fences may be cheaper when synchronizing many variables through one barrier.
- **Seqlocks** are widely used in the Linux kernel for read-mostly data (e.g., `jiffies`, time-of-day).
- Compile with `-std=c++11 -O2 -pthread`.
