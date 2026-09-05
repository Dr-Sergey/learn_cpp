# Avoid false sharing with cache-line padding in concurrent data structures

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size>  

---

## Topic Overview

**False sharing** is one of those performance bugs that looks completely innocent on paper but can kill your throughput in practice. It occurs when two threads access different variables that happen to live on the same CPU cache line. There is no logical data race - the threads are touching different things - but the hardware cache coherency protocol (for example, MESI) treats the entire cache line as a single unit. When Thread 1 writes its variable, the hardware must invalidate Thread 2's copy of the whole cache line, forcing Thread 2 to reload it from a slower cache level the next time it touches its own variable. This "ping-pong" between cores can make code that should scale well actually run slower than single-threaded.

### How False Sharing Works

The diagram below shows the problem. Both counters fit on one 64-byte cache line, so every write by either thread disturbs the other:

```cpp
Cache Line (64 bytes typically):
+----------------------------------------------------+
|  counter_a (4 bytes) |  counter_b (4 bytes) |  padding... |
+----------------------------------------------------+
        ^ Thread 1 writes          ^ Thread 2 writes

When Thread 1 writes counter_a:
  -> Core 2's cache line is INVALIDATED
  -> Core 2 must re-fetch the ENTIRE line from L3/memory
  -> Even though counter_b hasn't changed!
```

### The Fix: Pad Each Variable to Its Own Cache Line

The solution is to force each variable onto its own cache line so there is nothing for the two threads to share:

```cpp
+----------------------------------------------------+
|  counter_a (4 bytes)  |  padding (60 bytes)               |
+----------------------------------------------------+
+----------------------------------------------------+
|  counter_b (4 bytes)  |  padding (60 bytes)               |
+----------------------------------------------------+
  ^ Thread 1's cache line    ^ Thread 2's cache line - independent!
```

Now when Thread 1 writes `counter_a`, Thread 2's cache line is completely unaffected.

### C++17 Constants

C++17 gives you two named constants that make this portable. They live in `<new>`:

| Constant | Meaning | Typical value |
| --- | --- | --- |
| `hardware_destructive_interference_size` | Min offset to AVOID false sharing | 64 |
| `hardware_constructive_interference_size` | Max size to PROMOTE true sharing | 64 |

The names feel similar but the intent is opposite. Destructive means "keep apart." Constructive means "keep together."

---

## Self-Assessment

### Q1: Show a benchmark where two threads update adjacent atomic<int>s and suffer false sharing

**Answer:**

Let's put two counters right next to each other in memory and have two threads hammer them simultaneously. The slowdown is real and measurable:

```cpp
#include <atomic>
#include <thread>
#include <chrono>
#include <iostream>

struct NoPadding {
    std::atomic<int> a{0};  // Adjacent - likely same cache line!
    std::atomic<int> b{0};
};

constexpr int ITERATIONS = 100'000'000;

void bench(const char* label, auto& data) {
    data.a = 0;
    data.b = 0;

    auto start = std::chrono::steady_clock::now();

    std::thread t1([&] {
        for (int i = 0; i < ITERATIONS; ++i)
            data.a.fetch_add(1, std::memory_order_relaxed);
    });
    std::thread t2([&] {
        for (int i = 0; i < ITERATIONS; ++i)
            data.b.fetch_add(1, std::memory_order_relaxed);
    });

    t1.join();
    t2.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    std::cout << label << ": " << ms << " ms\n";
}

int main() {
    NoPadding np;
    bench("False sharing (adjacent)", np);
    // Typical output: ~800-1200 ms on modern x86
    // Cache line bouncing between cores causes massive slowdown

    // Compare: single-threaded baseline
    auto start = std::chrono::steady_clock::now();
    std::atomic<int> single{0};
    for (int i = 0; i < ITERATIONS * 2; ++i)
        single.fetch_add(1, std::memory_order_relaxed);
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();
    std::cout << "Single thread (2x iters): " << ms << " ms\n";
    // Often FASTER than the false-sharing version despite doing same total work!
}
```

The striking part is that single-threaded code doing twice the total work can actually be faster than two threads working in parallel. That is how bad false sharing can be - you added a second core and made things worse.

### Q2: Fix it by aligning each counter to alignas(std::hardware_destructive_interference_size)

**Answer:**

The fix is `alignas`. Force each counter to start at a cache-line boundary, and the two variables are guaranteed to live on separate cache lines:

```cpp
#include <atomic>
#include <thread>
#include <chrono>
#include <iostream>
#include <new> // for hardware_destructive_interference_size

struct Padded {
    alignas(std::hardware_destructive_interference_size)
    std::atomic<int> a{0};

    alignas(std::hardware_destructive_interference_size)
    std::atomic<int> b{0};
};

// Fallback for compilers that don't define it:
// #ifndef __cpp_lib_hardware_interference_size
// constexpr std::size_t hardware_destructive_interference_size = 64;
// #endif

constexpr int ITERATIONS = 100'000'000;

int main() {
    // Verify padding:
    std::cout << "sizeof(NoPadding struct): " << 2 * sizeof(std::atomic<int>) << " bytes\n";
    std::cout << "sizeof(Padded):           " << sizeof(Padded) << " bytes\n";
    std::cout << "Cache line size:          "
              << std::hardware_destructive_interference_size << " bytes\n";
    // Typical output:
    // sizeof(NoPadding struct): 8 bytes
    // sizeof(Padded):           128 bytes   (each on its own cache line)
    // Cache line size:          64 bytes

    Padded data;

    auto start = std::chrono::steady_clock::now();

    std::thread t1([&] {
        for (int i = 0; i < ITERATIONS; ++i)
            data.a.fetch_add(1, std::memory_order_relaxed);
    });
    std::thread t2([&] {
        for (int i = 0; i < ITERATIONS; ++i)
            data.b.fetch_add(1, std::memory_order_relaxed);
    });

    t1.join();
    t2.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    std::cout << "Padded (no false sharing): " << ms << " ms\n";
    // Typical: ~300-400 ms - 2-4x faster than the false-sharing version!

    // The improvement comes from each core's cache being independent:
    //   Before: Core 1 writes a -> invalidates Core 2's line -> Core 2 refetch
    //   After:  Core 1 writes a -> only its own cache line -> Core 2 unaffected
}
```

The struct grew from 8 bytes to 128 bytes. That is a 16x size increase, but it is a completely worthwhile trade-off for data that is frequently written by different threads. You are paying in memory to buy independence between cores.

### Q3: Explain std::hardware_constructive_interference_size and when to use it instead

**Answer:**

The two constants serve opposite goals. Destructive means "keep these apart so they don't interfere." Constructive means "keep these together so a single cache fetch gets everything the same thread needs at once":

```cpp
#include <new> // hardware_constructive_interference_size
#include <iostream>
#include <atomic>

// hardware_DESTRUCTIVE_interference_size:
//   "Keep these APART - they're accessed by DIFFERENT threads"
//   Use: padding between thread-private counters
//
// hardware_CONSTRUCTIVE_interference_size:
//   "Keep these TOGETHER - they're accessed by the SAME thread"
//   Use: ensure data accessed together fits in ONE cache line

// Example: A node that one thread accesses frequently
struct Node {
    int key;
    int value;
    Node* next;
    // Total ~20 bytes on 64-bit - fits in one cache line, good for cache hits
};

// Use constructive size to verify data fits one cache line:
static_assert(sizeof(Node) <= std::hardware_constructive_interference_size,
              "Node should fit in a single cache line for optimal access");

// Example: Producer token and its counter should be on the SAME line
struct alignas(std::hardware_constructive_interference_size) ProducerState {
    std::atomic<int> sequence{0}; // These two are always accessed together
    int cached_value{0};           // by the same producer thread
    // Both fit in one cache line -> single cache fetch = fast
};

// But two ProducerStates for DIFFERENT threads need DESTRUCTIVE padding:
struct alignas(std::hardware_destructive_interference_size) PaddedProducer {
    ProducerState state;
};

int main() {
    std::cout << "Destructive interference size: "
              << std::hardware_destructive_interference_size << "\n";
    std::cout << "Constructive interference size: "
              << std::hardware_constructive_interference_size << "\n";
    // Both are typically 64 on x86 (same cache line size)

    // Summary:
    // DESTRUCTIVE: alignas to SEPARATE  -> avoid false sharing between threads
    // CONSTRUCTIVE: pack together within -> promote true sharing within a thread

    // Real-world example: thread pool with per-thread work queues
    // - Queue header (head + tail pointers) -> constructive (same thread reads both)
    // - Different threads' queues -> destructive (pad between them)
}
```

On most architectures both constants equal 64 bytes, but they represent opposite design intentions. Think of it this way: constructive interference is about making the cache work for you by keeping related things close together. Destructive interference is about stopping the cache from working against you by keeping unrelated things far apart.

---

## Notes

- Portability: Not all compilers define these constants (GCC/Clang require `-std=c++17`; some older versions may not support them). Use `#ifdef __cpp_lib_hardware_interference_size` and fall back to 64.
- Memory trade-off: Padding increases struct size (for example, 8 bytes to 128 bytes). Only pad data that is frequently written by different threads.
- `perf c2c`: Linux tool to detect false sharing at runtime.
- Read-only data does not cause false sharing: if all threads only read, cache lines are shared without invalidation (MESI Shared state).
- Compile with `-std=c++17 -O2 -pthread`.
