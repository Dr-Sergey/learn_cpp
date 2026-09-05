# Use std::atomic_ref for atomic access to non-atomic objects (C++20)

**Category:** Concurrency and Parallelism  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic_ref>  

---

## Topic Overview

`std::atomic_ref<T>` provides atomic operations on an **existing, non-atomic** object. It does not own the object — it creates a temporary "atomic view" of a regular variable.

### Why atomic_ref

```cpp

Problem: You have a plain array, struct, or legacy API that uses non-atomic types.
         You need atomic access in some contexts but not others.

Without atomic_ref:
  Option A: Make everything atomic<T>  → overhead everywhere, changes API
  Option B: Use mutex                  → too heavy for simple counters
  Option C: Cast tricks               → undefined behavior

With atomic_ref (C++20):
  int plain_array[1000];               // regular memory
  std::atomic_ref<int> ref(plain_array[i]); // atomic view on demand
  ref.fetch_add(1);                    // lock-free atomic increment
  // plain_array[i] is still a plain int (no permanent overhead)

```

### Requirements

| Requirement | Why |
| --- | --- |
| Object must be suitably aligned | `atomic_ref<T>::required_alignment` (often > `alignof(T)`) |
| Object must outlive all `atomic_ref`s | `atomic_ref` stores a reference, not a copy |
| ALL concurrent access through `atomic_ref` | Mixing atomic_ref and plain access = data race = UB |

---

## Self-Assessment

### Q1: Use std::atomic_ref<int> to atomically update a plain int in a shared array

**Answer:**

```cpp

#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <numeric>

int main() {
    // Plain array — no std::atomic anywhere
    alignas(std::atomic_ref<int>::required_alignment)
    int counters[4] = {0, 0, 0, 0};

    // 8 threads each increment a counter based on their ID
    std::vector<std::thread> threads;
    for (int t = 0; t < 8; ++t) {
        threads.emplace_back([&counters, t] {
            int bucket = t % 4;
            for (int i = 0; i < 100'000; ++i) {
                // Create an atomic_ref on-the-fly for the specific element
                std::atomic_ref<int> ref(counters[bucket]);
                ref.fetch_add(1, std::memory_order_relaxed);
                // ref goes out of scope here — no overhead persists
            }
        });
    }
    for (auto& th : threads) th.join();

    // Each bucket was incremented by 2 threads × 100,000
    for (int i = 0; i < 4; ++i)
        std::cout << "counters[" << i << "] = " << counters[i] << "\n";

    int total = std::accumulate(std::begin(counters), std::end(counters), 0);
    std::cout << "Total: " << total << " (expected 800000)\n";

    // Output:
    // counters[0] = 200000
    // counters[1] = 200000
    // counters[2] = 200000
    // counters[3] = 200000
    // Total: 800000 (expected 800000)
}

```

**Explanation:** `atomic_ref<int>` wraps a plain `int` with atomic operations. Multiple `atomic_ref` objects referencing the same `int` all route through the same atomic mechanism. The array stays as plain `int[]` — zero overhead when you don't need atomic access.

### Q2: Explain why atomic_ref does not own the referenced object and lifetimes must be managed carefully

**Answer:**

```cpp

#include <atomic>
#include <thread>
#include <iostream>
#include <vector>

// atomic_ref is a NON-OWNING reference wrapper.
// Think of it like std::string_view for atomics.

// === DANGER 1: Dangling reference ===
std::atomic_ref<int> make_dangling() {
    int local = 42;
    return std::atomic_ref<int>(local);
    // BUG: local is destroyed, but atomic_ref still points to it
    // Using the returned ref = undefined behavior
}

// === DANGER 2: Mixing atomic and non-atomic access ===
void mixed_access_bug() {
    alignas(std::atomic_ref<int>::required_alignment) int x = 0;

    std::thread t1([&] {
        std::atomic_ref<int> ref(x);
        ref.store(1, std::memory_order_release);
    });

    std::thread t2([&] {
        // BUG: reading x directly (non-atomic) while t1 writes atomically
        // This is a DATA RACE — undefined behavior!
        int val = x; // ← WRONG! Should use atomic_ref here too
    });

    t1.join();
    t2.join();
}

// === CORRECT: All concurrent access through atomic_ref ===
void correct_pattern() {
    alignas(std::atomic_ref<int>::required_alignment) int x = 0;

    std::thread t1([&] {
        std::atomic_ref<int>(x).store(1, std::memory_order_release);
    });

    std::thread t2([&] {
        // CORRECT: also using atomic_ref for the read
        int val = std::atomic_ref<int>(x).load(std::memory_order_acquire);
        std::cout << "x = " << val << "\n"; // 0 or 1 (both valid, no UB)
    });

    t1.join();
    t2.join();
}

// === Rule: Once ANY thread uses atomic_ref on an object,
//     ALL concurrent accesses must use atomic_ref ===

int main() {
    // make_dangling(); // DON'T DO THIS
    // mixed_access_bug(); // DON'T DO THIS
    correct_pattern();
}

```

**Lifetime rules:**

1. The referenced object must outlive **all** `atomic_ref` instances that reference it.
2. Any concurrent access to the same object must go through `atomic_ref` — mixing plain reads/writes with atomic_ref reads/writes is UB.
3. `atomic_ref` is cheap to copy — copying it creates another reference to the same object.

### Q3: Show how atomic_ref enables lock-free algorithms on contiguous memory

**Answer:**

```cpp

#include <atomic>
#include <vector>
#include <thread>
#include <iostream>
#include <algorithm>
#include <numeric>
#include <cstdint>

// Parallel histogram: many threads increment bins in a plain int array
// atomic_ref lets us do this without:
//   - Making the array atomic<int>[] (changes API, prevents SIMD, wastes memory)
//   - Using a mutex per bin (too heavy)
//   - Using thread-local arrays + merge (complex)

void parallel_histogram() {
    constexpr int BINS = 256;
    constexpr int N_THREADS = 8;
    constexpr int SAMPLES = 1'000'000;

    // Plain array — can be mmap'd, SIMD-processed, etc.
    alignas(std::atomic_ref<int>::required_alignment)
    int histogram[BINS] = {};

    std::vector<std::thread> threads;
    for (int t = 0; t < N_THREADS; ++t) {
        threads.emplace_back([&histogram, t] {
            // Simple pseudo-random number generator
            uint32_t seed = t * 12345 + 67890;
            for (int i = 0; i < SAMPLES / N_THREADS; ++i) {
                seed = seed * 1103515245 + 12345;
                int bin = (seed >> 16) % BINS;

                // Lock-free increment on contiguous plain memory
                std::atomic_ref<int>(histogram[bin])
                    .fetch_add(1, std::memory_order_relaxed);
            }
        });
    }
    for (auto& th : threads) th.join();

    int total = std::accumulate(std::begin(histogram),
                                std::end(histogram), 0);
    std::cout << "Total samples: " << total << "\n";
    std::cout << "Bin[0] = " << histogram[0] << "\n";
    std::cout << "Bin[128] = " << histogram[128] << "\n";

    // After parallel phase, access histogram as plain array (no overhead)
    int max_bin = *std::max_element(std::begin(histogram),
                                     std::end(histogram));
    std::cout << "Max bin count: " << max_bin << "\n";
}

// atomic_ref on struct fields: useful for existing data structures
struct Particle {
    alignas(std::atomic_ref<float>::required_alignment)
    float x, y, z;
    alignas(std::atomic_ref<int>::required_alignment)
    int active_count;
};

void update_particle_count(Particle& p) {
    // Atomically update just one field of a plain struct
    std::atomic_ref<int>(p.active_count)
        .fetch_add(1, std::memory_order_relaxed);
    // Other fields remain non-atomic (no overhead)
}

int main() {
    parallel_histogram();

    Particle p{1.0f, 2.0f, 3.0f, 0};
    std::vector<std::thread> threads;
    for (int i = 0; i < 100; ++i)
        threads.emplace_back(update_particle_count, std::ref(p));
    for (auto& t : threads) t.join();
    std::cout << "Particle active_count: " << p.active_count << "\n";

    // Output:
    // Total samples: 1000000
    // Bin[0] = 3912       (varies)
    // Bin[128] = 3887     (varies)
    // Max bin count: 4100  (varies)
    // Particle active_count: 100
}

```

**Why atomic_ref is better than alternatives for contiguous memory:**

- `std::atomic<int>[]` — can't be `mmap`'d, can't be SIMD-processed, potentially different size/alignment
- Per-element mutex — enormous overhead (40+ bytes per element vs 0)
- `atomic_ref` — zero storage overhead, lock-free, works on existing allocations

---

## Notes

- **Alignment:** `atomic_ref<T>::required_alignment` may be greater than `alignof(T)`. For `int`, it's usually the same. For `double` on 32-bit platforms, it may differ. Always use `alignas`.
- **is_lock_free():** `atomic_ref<T>::is_always_lock_free` tells you at compile time. If false, the implementation uses an internal hash table of mutexes.
- **Copying atomic_ref:** Copies point to the same object. `atomic_ref<int> a(x); auto b = a;` — both `a` and `b` access `x` atomically.
- **Floating point:** `atomic_ref<float>` supports `fetch_add` and `fetch_sub` for lock-free floating-point accumulation.
- **Interop with C APIs:** `atomic_ref` is perfect for atomically accessing fields in C structs or shared memory regions that can't use `std::atomic<T>`.
- Compile with `-std=c++20 -O2 -pthread`.
