# Understand false sharing and cache line bouncing in multi-threaded code

**Category:** Performance and CPU Architecture  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size>  

---

## Topic Overview

Here is a multi-threading bug that shows up in profilers as "the threads are fast individually but terrible together," even though the threads are modifying completely different variables. The culprit is the cache line.

Cache coherence hardware - the MESI or MOESI protocol that keeps caches consistent across cores - operates on entire cache lines, not individual bytes. A cache line is typically 64 bytes. If two threads on different cores both write to variables that happen to share a cache line, every write by one thread invalidates the other thread's cached copy of that line. The other core then has to fetch the updated line before its next write. The result is that the cache line ping-pongs between cores, at roughly 40-100 cycles per round trip:

```cpp
Cache line (64 bytes):
+---------------------------+
| counter_A | counter_B     |  <- Thread 0 writes A, Thread 1 writes B
+---------------------------+
     |             |
  Core 0        Core 1
  writes A      writes B
     |             |
  Invalidate!   Invalidate!
     <----------->           <- cache line ping-pongs between cores
                               ~40-100 cycles per ping-pong!
```

This is called **false sharing** because the threads look like they are sharing data (they compete for the same cache line) even though they are logically accessing separate variables.

Here is a quick reference for the related terms:

| Term | Description |
| --- | --- |
| Cache line | Smallest unit of cache transfer (typically 64 bytes) |
| False sharing | Different variables on same cache line modified by different cores |
| True sharing | Same variable modified by multiple cores |
| `hardware_destructive_interference_size` | C++17: minimum offset to avoid false sharing |
| `hardware_constructive_interference_size` | C++17: max size for true sharing benefit |

---

## Self-Assessment

### Q1: Demonstrate false sharing

The two counters in `SharedCounters` sit at offset 0 and offset 8 within the struct. That puts them both inside the same 64-byte cache line. Two threads hammering them simultaneously triggers exactly the ping-pong described above.

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>
#include <atomic>

// FALSE SHARING: two counters on the SAME cache line
struct SharedCounters {
    long long counter_a;  // offset 0 (8 bytes)
    long long counter_b;  // offset 8 (same 64-byte cache line!)
};

void increment(long long& counter, int iterations) {
    for (int i = 0; i < iterations; ++i) {
        counter += i; // Each write invalidates the other core's cache line!
    }
}

int main() {
    constexpr int ITERS = 100'000'000;
    SharedCounters shared{};

    auto t0 = std::chrono::high_resolution_clock::now();

    std::thread t1(increment, std::ref(shared.counter_a), ITERS);
    std::thread t2(increment, std::ref(shared.counter_b), ITERS);
    t1.join();
    t2.join();

    auto t1_end = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1_end - t0);

    std::cout << "False sharing (adjacent): " << ms.count() << " ms\n";
    std::cout << "counter_a: " << shared.counter_a << '\n';
    std::cout << "counter_b: " << shared.counter_b << '\n';

    // Both counters are on the same 64-byte cache line.
    // Core 0 writes counter_a -> invalidates Core 1's copy
    // Core 1 writes counter_b -> invalidates Core 0's copy
    // Result: ~3-10x slower than without false sharing!
    //
    // Note: even though the threads access DIFFERENT variables,
    // the cache coherence protocol (MESI/MOESI) operates on
    // entire cache lines, not individual bytes.
}
```

The performance hit of 3-10x is typical. The exact factor depends on how many cores are involved and how hot the loop is.

### Q2: Fix with padding to separate cache lines

The fix is to put each counter on its own cache line using `alignas(64)`. This forces the struct layout to place each member at a 64-byte boundary, so they live on separate lines and their writes never interfere with each other.

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <new>  // hardware_destructive_interference_size

// Option 1: Manual padding
struct PaddedCounters {
    alignas(64) long long counter_a;  // own cache line
    alignas(64) long long counter_b;  // own cache line
};

// Option 2: Using C++17 constant (if supported)
// constexpr size_t CACHE_LINE = std::hardware_destructive_interference_size;
// Many compilers don't define it yet; use 64 directly.
constexpr size_t CACHE_LINE = 64;

struct PaddedCountersV2 {
    alignas(CACHE_LINE) long long counter_a;
    alignas(CACHE_LINE) long long counter_b;
};

// BAD: no padding (false sharing)
struct BadCounters {
    long long counter_a;
    long long counter_b;
};

void increment(long long& counter, int iters) {
    for (int i = 0; i < iters; ++i) counter += i;
}

int main() {
    constexpr int ITERS = 100'000'000;

    auto bench = [&](auto& counters, const char* label) {
        counters.counter_a = 0;
        counters.counter_b = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        std::thread t1(increment, std::ref(counters.counter_a), ITERS);
        std::thread t2(increment, std::ref(counters.counter_b), ITERS);
        t1.join(); t2.join();
        auto t1_end = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1_end - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    BadCounters bad{};
    PaddedCounters padded{};

    bench(bad,    "False sharing (adjacent)");
    bench(padded, "Fixed (padded to 64B)   ");

    std::cout << "\nSizes:\n";
    std::cout << "  BadCounters:    " << sizeof(BadCounters) << " bytes\n";    // 16
    std::cout << "  PaddedCounters: " << sizeof(PaddedCounters) << " bytes\n"; // 128
    // Typical results:
    // False sharing: ~800 ms
    // Fixed:         ~120 ms  (6-7x faster!)
    // Cost: 112 extra bytes per pair. Usually worth it.
}
```

The struct grows from 16 bytes to 128 bytes. That is a real cost if you have a large array of these, but 6-7x faster thread throughput is almost always worth 112 extra bytes.

### Q3: Detecting false sharing with `perf c2c`

The frustrating thing about false sharing is that it does not show up as high CPU usage or obvious contention. Both threads appear to be running. The regular cache miss counters may or may not spike. The right tool is `perf c2c`, which specifically tracks cache-to-cache transfers - the HITM (Hit In Modified) events that are the fingerprint of false sharing.

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <atomic>

// Program designed to exhibit false sharing for perf c2c detection.
struct FalseSharingDemo {
    std::atomic<long long> counters[8];  // 8 atomics, 64 bytes total
    // All 8 counters fit on ONE cache line!
};

void worker(std::atomic<long long>& counter, int iters) {
    for (int i = 0; i < iters; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    FalseSharingDemo demo{};
    constexpr int ITERS = 10'000'000;

    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back(worker, std::ref(demo.counters[i]), ITERS);
    }
    for (auto& t : threads) t.join();

    for (int i = 0; i < 4; ++i)
        std::cout << "Counter " << i << ": " << demo.counters[i] << '\n';

    // DETECTING with perf c2c:
    //
    // Step 1: Record cache-to-cache transfers
    //   perf c2c record -g ./false_sharing_demo
    //
    // Step 2: Analyze
    //   perf c2c report --stdio
    //
    // Output shows:
    //   =================================================
    //   Shared Data Cache Line Table
    //   =================================================
    //   Total records: 12345
    //   Total HITM:    8901     <-- cache-to-cache transfers (false sharing!)
    //
    //   Cacheline    HITM%   Address    Symbol
    //   0x7f1234560  89.1%   0x...      FalseSharingDemo::counters
    //   ^-- This line shows the hot cache line causing false sharing
    //
    // HITM = Hit In Modified (another core's cache)
    // High HITM% = false sharing!
    //
    // Fix: pad each atomic to its own cache line:
    //   struct alignas(64) PaddedAtomic {
    //       std::atomic<long long> value;
    //   };
}
```

The `counters[8]` array is 8 × 8 = 64 bytes, which lands all 8 atomics on exactly one cache line. Four threads hammering four of them simultaneously maximizes the HITM rate. The `perf c2c` output will point directly at the struct member. Once you see high HITM%, `alignas(64)` on each atomic is the fix.

---

## Notes

- Cache line size is 64 bytes on x86/ARM; use `alignas(64)` for padding.
- `std::hardware_destructive_interference_size` (C++17) provides the portable constant, but many compilers don't implement it yet - `64` is a safe fallback.
- False sharing is invisible in profilers that only show CPU time; use `perf c2c` specifically to find it.
- Common false sharing sites: thread-local counters in arrays, adjacent atomics, struct members accessed by different threads.
- `perf stat -e L1-dcache-load-misses` shows an elevated cache miss rate that can hint at false sharing, but `perf c2c` gives the definitive diagnosis.
