# Use Cache Warming and Prefetching in Latency-Critical Code

**Category:** Low Latency and Real Time  
**Standard:** C++17, C++20  
**Reference:** [Intel 64 Optimization Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html), [What Every Programmer Should Know About Memory - U. Drepper](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)  

---

## Topic Overview

A last-level cache (LLC) miss on modern x86 costs 60-100ns for local DRAM, versus 1-4ns for an L1 hit - a 30x penalty. In latency-critical code where every nanosecond matters, **cache warming** preloads data into cache during idle periods, and **software prefetching** issues explicit hints to the CPU to fetch data ahead of when it is needed.

Cache warming is a cold-path activity: you periodically touch all hot data structures to keep them resident in cache. This is essential after context switches, long idle periods, or when your working set competes with other processes. The key insight is that merely reading data is sufficient - the CPU caches the line on a read, so you don't have to do any meaningful computation during a warming pass.

Software prefetch instructions (`_mm_prefetch` / `__builtin_prefetch`) issue non-blocking loads into a specified cache level. **Temporal prefetch** (`_MM_HINT_T0`) brings data into L1; **non-temporal** (`_MM_HINT_NTA`) brings data into LLC without polluting L1, which is useful for streaming access patterns where you only need the data once. The prefetch distance - how far ahead you prefetch - depends on the memory latency and iteration time. Typical starting values are 200-400ns worth of loop iterations ahead. Too close and the data isn't ready; too far and it gets evicted before you use it.

| Cache Level | Latency | Size (typical) | Prefetch Hint |
| --- | --- | --- | --- |
| L1d | 1-2 ns | 32-48 KB | `_MM_HINT_T0` |
| L2 | 4-7 ns | 256 KB-1 MB | `_MM_HINT_T1` |
| L3 (LLC) | 10-40 ns | 8-64 MB | `_MM_HINT_T2` |
| DRAM | 60-100 ns | GBs | - (miss) |
| Non-temporal | LLC bypass | - | `_MM_HINT_NTA` |

Here is the mental model for how prefetch distance works. You issue a prefetch for element `i + distance` while you are processing element `i`, so by the time your loop reaches `i + distance` the data is already sitting in L1:

```cpp
PREFETCH PIPELINE:
  Iteration i:    process(data[i])
  Prefetch:       _mm_prefetch(data[i + distance])
                        │
                        ▼
  Iteration i+distance: process(data[i+distance])  // data already in L1
```

---

## Self-Assessment

### Q1: Implement a cache warmer for an order book that periodically touches all price levels to keep them in L1/L2 cache

Each `PriceLevel` is padded to exactly 64 bytes - one cache line. The warmer only needs to read one word per entry to pull the entire line into cache. Notice that `warm_order_book` does the minimum possible work: just a timestamp read. The goal is cache residency, not computation:

```cpp
#include <array>
#include <cstdint>
#include <cstdio>
#include <chrono>
#include <immintrin.h>  // _mm_prefetch

struct PriceLevel {
    double price;
    int64_t quantity;
    uint64_t order_count;
    uint64_t timestamp;
    // Pad to 64 bytes (one cache line)
    char padding_[64 - sizeof(double) - sizeof(int64_t) -
                  2 * sizeof(uint64_t)];
};
static_assert(sizeof(PriceLevel) == 64, "PriceLevel must be one cache line");

constexpr int kMaxLevels = 1024;

struct OrderBook {
    std::array<PriceLevel, kMaxLevels> bids;
    std::array<PriceLevel, kMaxLevels> asks;
    int bid_count = 0;
    int ask_count = 0;
};

// Cache warmer: touch every cache line in the order book.
// Call this periodically (e.g., every 100µs) during idle time.
void warm_order_book(const OrderBook& book) noexcept {
    volatile uint64_t sink = 0;

    // Read one word per cache line - this loads the entire line into L1
    for (int i = 0; i < book.bid_count; ++i) {
        sink += book.bids[i].timestamp;
    }
    for (int i = 0; i < book.ask_count; ++i) {
        sink += book.asks[i].timestamp;
    }
    (void)sink;  // prevent optimization
}

// Selective prefetch for the top N levels (most accessed)
void prefetch_top_of_book(const OrderBook& book, int depth = 5) noexcept {
    for (int i = 0; i < depth && i < book.bid_count; ++i) {
        _mm_prefetch(reinterpret_cast<const char*>(&book.bids[i]),
                     _MM_HINT_T0);
    }
    for (int i = 0; i < depth && i < book.ask_count; ++i) {
        _mm_prefetch(reinterpret_cast<const char*>(&book.asks[i]),
                     _MM_HINT_T0);
    }
}

int main() {
    OrderBook book;
    book.bid_count = 500;
    book.ask_count = 500;
    for (int i = 0; i < 500; ++i) {
        book.bids[i] = {100.0 - i * 0.01, 1000 + i, 5, 0};
        book.asks[i] = {100.0 + i * 0.01, 1000 + i, 5, 0};
    }

    // Benchmark: cold vs warm access
    auto bench = [&](const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        double best_bid = book.bids[0].price;
        double best_ask = book.asks[0].price;
        volatile double spread = best_ask - best_bid;
        auto t1 = std::chrono::high_resolution_clock::now();
        (void)spread;
        auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0).count();
        std::printf("%-12s spread access: %lld ns\n", label, (long long)ns);
    };

    bench("Cold:");
    warm_order_book(book);
    bench("Warm:");
    prefetch_top_of_book(book);
    bench("Prefetched:");
}
```

The benchmark shows three tiers: cold (data nowhere near the CPU), warm (full book in cache), and explicitly prefetched (top of book in L1). The warm and prefetched numbers should be much lower than cold.

### Q2: Use software prefetching in a loop that processes a large array, measuring the throughput gain vs no prefetch

The three variants here cover the spectrum from "no hint at all" to "hint into L1" to "hint but don't pollute L1." The non-temporal hint is specifically useful when you're streaming through data and won't need it again - it prevents the prefetch from kicking useful data out of L1:

```cpp
#include <vector>
#include <chrono>
#include <cstdio>
#include <cstdlib>
#include <immintrin.h>
#include <numeric>

constexpr std::size_t kElements = 16 * 1024 * 1024;  // 128 MB of doubles
constexpr int kPrefetchDistance = 8;  // 8 * 64B = 512B ahead

double process_no_prefetch(const double* data, std::size_t n) {
    double sum = 0;
    for (std::size_t i = 0; i < n; ++i) {
        sum += data[i] * data[i];
    }
    return sum;
}

double process_with_prefetch(const double* data, std::size_t n) {
    double sum = 0;
    for (std::size_t i = 0; i < n; ++i) {
        // Prefetch kPrefetchDistance cache lines ahead
        if (i + kPrefetchDistance * 8 < n) {
            _mm_prefetch(reinterpret_cast<const char*>(
                &data[i + kPrefetchDistance * 8]), _MM_HINT_T0);
        }
        sum += data[i] * data[i];
    }
    return sum;
}

double process_nontemporal(const double* data, std::size_t n) {
    double sum = 0;
    for (std::size_t i = 0; i < n; ++i) {
        // NTA: don't pollute L1/L2, use only LLC - good for streaming
        if (i + kPrefetchDistance * 8 < n) {
            _mm_prefetch(reinterpret_cast<const char*>(
                &data[i + kPrefetchDistance * 8]), _MM_HINT_NTA);
        }
        sum += data[i] * data[i];
    }
    return sum;
}

int main() {
    std::vector<double> data(kElements);
    std::iota(data.begin(), data.end(), 1.0);

    auto bench = [&](auto fn, const char* label) {
        // Cold run
        volatile double result = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        result = fn(data.data(), data.size());
        auto t1 = std::chrono::high_resolution_clock::now();
        double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
        double gbps = (kElements * sizeof(double)) / (ms * 1e6);
        std::printf("%-20s %8.2f ms  %.2f GB/s  (sum=%.0f)\n",
                    label, ms, gbps, (double)result);
    };

    bench(process_no_prefetch,   "No prefetch");
    bench(process_with_prefetch, "SW prefetch T0");
    bench(process_nontemporal,   "SW prefetch NTA");
}
```

How much gain you see depends heavily on the array size relative to your LLC. Once the whole array fits in L3, the hardware prefetcher may already be doing a good job on a sequential loop and software hints may add little. The real win from software prefetch comes with irregular or pointer-chasing patterns.

### Q3: Design a hot-data layout that keeps frequently accessed fields in the same cache line and cold fields in separate storage

This is the classic hot/cold struct split. The `OrderBad` layout forces the CPU to fetch the 64-byte `description` field just to get at `price` and `quantity`, even though you never read the description in the hot path. The split layout puts only the three hot fields in a 64-byte-aligned struct:

```cpp
#include <cstdint>
#include <cstdio>
#include <new>
#include <array>
#include <chrono>
#include <immintrin.h>

// BAD: mixed hot/cold fields waste cache space
struct OrderBad {
    double price;         // HOT
    int32_t quantity;     // HOT
    uint64_t order_id;    // HOT
    char description[64]; // COLD - rarely accessed, wastes 64B in hot cache line
    uint64_t created_at;  // COLD
    char client_id[32];   // COLD
};

// GOOD: split into hot (cache-line-aligned) and cold structures
struct alignas(64) OrderHot {
    double price;
    int32_t quantity;
    uint64_t order_id;
    uint32_t cold_index;  // index into cold storage
    // 64 - 8 - 4 - 8 - 4 = 40 bytes padding (fits one cache line)
};
static_assert(sizeof(OrderHot) == 64);

struct OrderCold {
    char description[64];
    uint64_t created_at;
    char client_id[32];
};

constexpr int kOrders = 100'000;

// Benchmark: iterate only over hot data
double sum_prices_hot(const std::array<OrderHot, kOrders>& orders) {
    double sum = 0;
    for (std::size_t i = 0; i < kOrders; ++i) {
        sum += orders[i].price * orders[i].quantity;
    }
    return sum;
}

double sum_prices_bad(const std::array<OrderBad, kOrders>& orders) {
    double sum = 0;
    for (std::size_t i = 0; i < kOrders; ++i) {
        sum += orders[i].price * orders[i].quantity;
    }
    return sum;
}

int main() {
    static std::array<OrderHot, kOrders> hot_orders;
    static std::array<OrderBad, kOrders> bad_orders;

    for (int i = 0; i < kOrders; ++i) {
        hot_orders[i] = {100.0 + i, i % 1000, (uint64_t)i, (uint32_t)i};
        bad_orders[i].price = 100.0 + i;
        bad_orders[i].quantity = i % 1000;
    }

    auto bench = [](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile double r = fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::printf("%-20s %6lld µs  (result=%.0f)\n", label, (long long)us, (double)r);
    };

    bench([&] { return sum_prices_hot(hot_orders); }, "Hot/cold split");
    bench([&] { return sum_prices_bad(bad_orders); }, "Mixed layout");
}
```

The `OrderBad` array packs cold bytes into every cache line fetch, which means each load pulls in dead weight and the effective cache capacity for useful data drops. The split version lets you iterate through 100k orders touching only the fields you actually need, so you get more useful data per cache fill.

---

## Notes

- **Prefetch distance** must be tuned empirically: too short and the data isn't ready; too far and it's evicted before use. Start at 4-8 cache lines and benchmark.
- **`__builtin_prefetch(addr, rw, locality)`** is the GCC/Clang intrinsic: `rw=0` for read, `rw=1` for write; `locality` 0-3 maps to NTA/T2/T1/T0.
- Cache warming is most useful for **pointer-chasing** data structures (linked lists, trees) where hardware prefetchers fail because they can't predict the next address.
- **Hardware prefetchers** handle sequential and strided access well - software prefetch is most valuable for irregular or pointer-based access patterns.
- Use `perf stat -e L1-dcache-load-misses,LLC-load-misses` to measure cache miss rates before and after optimization.
- Hot/cold splitting is also called **structure splitting** or **data-oriented design** - it's the foundation of ECS (Entity Component System) in game engines.
