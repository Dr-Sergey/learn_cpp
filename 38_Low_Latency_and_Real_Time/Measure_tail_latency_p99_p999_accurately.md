# Measure Tail Latency (p99/p99.9) Accurately

**Category:** Low Latency and Real Time  
**Standard:** C++17, C++20  
**Reference:** [HDR Histogram](http://hdrhistogram.org/), [Gil Tene - How NOT to Measure Latency](https://www.youtube.com/watch?v=lJ8ydIuPFeU)  

---

## Topic Overview

Averages lie in low-latency systems. A function with 1µs average latency but 500µs p99.9 will cause periodic user-visible stalls. **Tail latencies** - percentiles like p99 (99th percentile) and p99.9 - capture the worst-case experience for 1-in-100 and 1-in-1000 operations. Measuring them accurately requires careful timestamping, appropriate data structures, and awareness of measurement pitfalls.

The reason averages are deceptive is mathematical: if 999 out of 1000 operations take 1µs but one takes 1ms, the average is about 2µs - which sounds fine. But your user experiences that 1ms stall once every second if you're processing 1000 operations per second. Tail percentiles capture that story much more honestly. The p99.9 in this example would be 1ms, immediately flagging the problem.

The primary timing source on x86 is `rdtsc` (read timestamp counter), which provides cycle-accurate resolution (~0.3ns at 3GHz) with minimal overhead (<10ns per call). `std::chrono::high_resolution_clock` wraps OS facilities that may add 20-50ns overhead. For sub-microsecond measurements, `rdtsc` is preferred; for portability, `steady_clock` is acceptable.

**Coordinated omission** is the most common measurement error in latency benchmarks, and it's genuinely subtle. When the system under test slows down, a naïve test harness also slows down, systematically underweighting slow responses. Here's the intuition: imagine you're supposed to send one request every 1ms, but the 500th request takes 100ms. A naïve harness waits for that response before sending the next request, so requests 501-600 are never sent. Those 99 requests that would have been issued during the 100ms stall are simply missing from your measurements. You record one slow response instead of 100 late responses. The corrected approach accounts for all the slots that should have been served during the stall period.

| Clock Source | Resolution | Overhead | Portability | Use Case |
| --- | --- | --- | --- | --- |
| `rdtsc` / `rdtscp` | ~0.3 ns | <10 ns | x86 only | Intra-function timing |
| `std::chrono::steady_clock` | 1-100 ns | 20-50 ns | Portable | General benchmarking |
| `clock_gettime(MONOTONIC)` | 1-20 ns | 20-40 ns | POSIX | System-level timing |
| `QueryPerformanceCounter` | ~100 ns | ~30 ns | Windows | Win32 benchmarking |

Here's what a typical latency distribution looks like. The median might be in the microseconds, but the tail stretches orders of magnitude higher as you hit GC pauses, page faults, and scheduler preemptions:

```cpp
LATENCY DISTRIBUTION (typical low-latency system):

Count │██████████████████████████████  <- p50: 1µs
      │████████████████████
      │████████████
      │███████
      │████
      │██
      │█                                <- p99: 15µs
      │░                                <- p99.9: 150µs
      │░                                <- p99.99: 2ms (GC, page fault)
      └──────────────────────────────
        1µs    10µs   100µs   1ms   10ms
```

---

## Self-Assessment

### Q1: Implement a high-resolution latency recorder using `rdtsc` with an HDR histogram that reports p50/p99/p99.9 percentiles

The key design challenge here is that you need to record millions of samples without losing precision at high values. A simple sorted array would work but can't be queried in constant time. This HDR-inspired histogram uses linear buckets for small values and logarithmically-spaced buckets for large values, giving useful precision across the entire range from nanoseconds to seconds.

```cpp
#include <cstdint>
#include <cstdio>
#include <algorithm>
#include <array>
#include <vector>
#include <chrono>
#include <cmath>

#ifdef _MSC_VER
#include <intrin.h>
#else
#include <x86intrin.h>
#endif

// Lightweight rdtsc wrapper
inline uint64_t rdtsc_now() noexcept {
    unsigned int aux;
    return __rdtscp(&aux);  // serializing version
}

// Estimate TSC frequency
double calibrate_tsc_ghz() {
    auto t0 = std::chrono::steady_clock::now();
    uint64_t c0 = rdtsc_now();

    // Busy-wait ~50ms
    while (std::chrono::steady_clock::now() - t0 < std::chrono::milliseconds(50)) {}

    auto t1 = std::chrono::steady_clock::now();
    uint64_t c1 = rdtsc_now();

    double seconds = std::chrono::duration<double>(t1 - t0).count();
    return (c1 - c0) / (seconds * 1e9);
}

// Simple HDR-like histogram: log-linear buckets
class LatencyHistogram {
    // Buckets: 0-1023ns in 1ns steps, then log2-scaled up to ~1s
    static constexpr int kLinearBuckets = 1024;
    static constexpr int kLogBuckets = 30;  // up to 2^30 ns ~= 1s
    static constexpr int kTotalBuckets = kLinearBuckets + kLogBuckets;

    std::array<uint64_t, kTotalBuckets> counts_{};
    uint64_t total_ = 0;

    int bucket_for(uint64_t ns) const noexcept {
        if (ns < kLinearBuckets) return static_cast<int>(ns);
        int log = 63 - __builtin_clzll(ns);  // floor(log2(ns))
        int bucket = kLinearBuckets + (log - 10);  // 1024 = 2^10
        return std::min(bucket, kTotalBuckets - 1);
    }

    uint64_t bucket_value(int bucket) const noexcept {
        if (bucket < kLinearBuckets) return bucket;
        return 1ULL << (bucket - kLinearBuckets + 10);
    }

public:
    void record(uint64_t nanoseconds) noexcept {
        ++counts_[bucket_for(nanoseconds)];
        ++total_;
    }

    uint64_t percentile(double p) const noexcept {
        uint64_t target = static_cast<uint64_t>(total_ * p / 100.0);
        uint64_t cumulative = 0;
        for (int i = 0; i < kTotalBuckets; ++i) {
            cumulative += counts_[i];
            if (cumulative >= target) return bucket_value(i);
        }
        return bucket_value(kTotalBuckets - 1);
    }

    void report() const {
        std::printf("┌────────────┬────────────┐\n");
        std::printf("│ Percentile │ Latency    │\n");
        std::printf("├────────────┼────────────┤\n");
        std::printf("│ p50        │ %6lu ns  │\n", (unsigned long)percentile(50));
        std::printf("│ p90        │ %6lu ns  │\n", (unsigned long)percentile(90));
        std::printf("│ p99        │ %6lu ns  │\n", (unsigned long)percentile(99));
        std::printf("│ p99.9      │ %6lu ns  │\n", (unsigned long)percentile(99.9));
        std::printf("│ p99.99     │ %6lu ns  │\n", (unsigned long)percentile(99.99));
        std::printf("│ max        │ %6lu ns  │\n", (unsigned long)percentile(100));
        std::printf("└────────────┴────────────┘\n");
        std::printf("Total samples: %lu\n", (unsigned long)total_);
    }
};

int main() {
    double tsc_ghz = calibrate_tsc_ghz();
    std::printf("TSC frequency: %.3f GHz\n", tsc_ghz);

    LatencyHistogram hist;

    // Benchmark: measure cost of a simple operation
    for (int i = 0; i < 1'000'000; ++i) {
        uint64_t start = rdtsc_now();
        // Simulate work: atomic increment
        volatile int x = 0;
        x = x + 1;
        uint64_t end = rdtsc_now();
        uint64_t cycles = end - start;
        uint64_t ns = static_cast<uint64_t>(cycles / tsc_ghz);
        hist.record(ns);
    }

    hist.report();
}
```

The `__rdtscp` instruction is the serializing form of `rdtsc`. The "p" stands for "processor" - it also writes the current CPU ID to `aux`, which is useful for detecting if the thread migrated CPUs mid-measurement (which would make the TSC values incomparable). The serialization means the instruction completes in order, preventing out-of-order execution from pulling the timestamp before the work you're measuring.

### Q2: Demonstrate coordinated omission and implement a corrected measurement that accounts for missed request slots

This is one of the most important benchmarking lessons in systems programming. The naïve approach records what actually happened. The corrected approach records what should have happened from a user's perspective - including all the requests that were invisibly delayed.

```cpp
#include <chrono>
#include <cstdio>
#include <thread>
#include <vector>
#include <algorithm>
#include <numeric>
#include <cmath>
#include <random>

using Clock = std::chrono::high_resolution_clock;
using ns = std::chrono::nanoseconds;

// Simulated service: usually 1µs, but every 1000th call takes 1ms
uint64_t simulate_service(int call_id) {
    if (call_id % 1000 == 999) {
        auto t0 = Clock::now();
        // Simulate stall
        while (Clock::now() - t0 < std::chrono::microseconds(1000)) {}
        return 1'000'000;  // 1ms
    }
    auto t0 = Clock::now();
    while (Clock::now() - t0 < std::chrono::microseconds(1)) {}
    return 1'000;  // 1µs
}

struct PercentileResult {
    double p50, p99, p999;
};

PercentileResult compute_percentiles(std::vector<uint64_t>& data) {
    std::sort(data.begin(), data.end());
    auto pct = [&](double p) -> double {
        size_t idx = static_cast<size_t>(data.size() * p / 100.0);
        idx = std::min(idx, data.size() - 1);
        return static_cast<double>(data[idx]);
    };
    return {pct(50), pct(99), pct(99.9)};
}

int main() {
    constexpr int N = 100'000;
    constexpr uint64_t expected_interval_ns = 10'000;  // 10µs between requests

    // Naive measurement (has coordinated omission)
    std::vector<uint64_t> naive_latencies;
    naive_latencies.reserve(N);

    for (int i = 0; i < N; ++i) {
        uint64_t lat = simulate_service(i);
        naive_latencies.push_back(lat);
    }

    // Corrected measurement: account for missed intervals
    std::vector<uint64_t> corrected_latencies;
    corrected_latencies.reserve(N * 2);

    uint64_t expected_start = 0;
    uint64_t actual_time = 0;

    for (int i = 0; i < N; ++i) {
        uint64_t service_time = simulate_service(i);
        actual_time += service_time;

        // For each missed interval, record the overdue latency
        while (expected_start + expected_interval_ns <= actual_time) {
            uint64_t corrected = actual_time - expected_start;
            corrected_latencies.push_back(corrected);
            expected_start += expected_interval_ns;
        }
    }

    auto naive = compute_percentiles(naive_latencies);
    auto corrected = compute_percentiles(corrected_latencies);

    std::printf("┌─────────────┬──────────────┬──────────────┐\n");
    std::printf("│ Percentile  │ Naive (ns)   │ Corrected    │\n");
    std::printf("├─────────────┼──────────────┼──────────────┤\n");
    std::printf("│ p50         │ %10.0f   │ %10.0f   │\n", naive.p50, corrected.p50);
    std::printf("│ p99         │ %10.0f   │ %10.0f   │\n", naive.p99, corrected.p99);
    std::printf("│ p99.9       │ %10.0f   │ %10.0f   │\n", naive.p999, corrected.p999);
    std::printf("└─────────────┴──────────────┴──────────────┘\n");
    std::printf("\nNaive samples: %zu, Corrected samples: %zu\n",
                naive_latencies.size(), corrected_latencies.size());
}
```

Notice that the corrected dataset has more samples than the naive dataset - the extra samples represent the requests that were invisibly delayed by the stall. In the naive version, the 1ms stall appears once. In the corrected version, it fans out into many late-arriving responses, which is exactly what a real client would experience.

### Q3: Build a real-time latency monitor that continuously tracks p99 over a sliding window and triggers an alert when it exceeds a threshold

Production systems need continuous latency monitoring, not just offline analysis. This sliding window tracker is designed so the hot path (the `record()` call) is fast and the slow analysis (sorting and computing percentiles) is done on a separate monitoring thread.

```cpp
#include <array>
#include <atomic>
#include <cstdint>
#include <cstdio>
#include <chrono>
#include <thread>
#include <algorithm>

// Lock-free sliding window percentile tracker
class SlidingWindowP99 {
    static constexpr int kWindowSize = 10'000;
    std::array<uint64_t, kWindowSize> samples_{};
    std::atomic<int> write_pos_{0};
    std::atomic<int> count_{0};
    uint64_t threshold_ns_;

public:
    explicit SlidingWindowP99(uint64_t threshold_ns)
        : threshold_ns_(threshold_ns) {
        samples_.fill(0);
    }

    // Called from hot path — must be fast
    void record(uint64_t latency_ns) noexcept {
        int pos = write_pos_.fetch_add(1, std::memory_order_relaxed) % kWindowSize;
        samples_[pos] = latency_ns;
        int c = count_.load(std::memory_order_relaxed);
        if (c < kWindowSize)
            count_.store(c + 1, std::memory_order_relaxed);
    }

    // Called from monitoring thread — can be slower
    struct Stats {
        uint64_t p50, p99, p999, max;
        bool alert;
    };

    Stats compute() const {
        int n = count_.load(std::memory_order_acquire);
        if (n == 0) return {0, 0, 0, 0, false};

        // Copy and sort current window
        std::vector<uint64_t> sorted(n);
        for (int i = 0; i < n; ++i)
            sorted[i] = samples_[i];
        std::sort(sorted.begin(), sorted.end());

        auto pct = [&](double p) -> uint64_t {
            size_t idx = static_cast<size_t>(n * p / 100.0);
            return sorted[std::min(idx, sorted.size() - 1)];
        };

        uint64_t p99_val = pct(99);
        return {
            pct(50), p99_val, pct(99.9),
            sorted.back(),
            p99_val > threshold_ns_
        };
    }
};

int main() {
    SlidingWindowP99 tracker(50'000);  // alert if p99 > 50µs
    std::atomic<bool> running{true};

    // Simulated hot-path thread
    std::thread producer([&] {
        int i = 0;
        while (running.load(std::memory_order_relaxed)) {
            auto t0 = std::chrono::high_resolution_clock::now();
            // Simulated work
            volatile int x = 0;
            for (int j = 0; j < 100; ++j) x += j;
            if (i++ % 500 == 0) {
                // Occasional spike
                for (int j = 0; j < 10'000; ++j) x += j;
            }
            auto t1 = std::chrono::high_resolution_clock::now();
            uint64_t ns = std::chrono::duration_cast<
                std::chrono::nanoseconds>(t1 - t0).count();
            tracker.record(ns);
        }
    });

    // Monitoring thread: check every 100ms
    for (int epoch = 0; epoch < 10; ++epoch) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        auto stats = tracker.compute();
        std::printf("Epoch %2d | p50: %5lu ns | p99: %5lu ns | "
                    "p99.9: %5lu ns | max: %6lu ns %s\n",
                    epoch,
                    (unsigned long)stats.p50,
                    (unsigned long)stats.p99,
                    (unsigned long)stats.p999,
                    (unsigned long)stats.max,
                    stats.alert ? "ALERT" : "OK");
    }
    running.store(false);
    producer.join();
}
```

The `record()` function is a single `fetch_add` plus a non-atomic array write - it's about as fast as possible. The `compute()` function does a sort, which is expensive and not suitable for the hot path, but it runs on the monitoring thread where that's fine. The trade-off is that `compute()` sees a snapshot of the samples array rather than a perfectly consistent view, but for latency monitoring that approximation is acceptable.

---

## Notes

- **`rdtscp`** (serializing) is preferred over `rdtsc` for measurement - it prevents out-of-order execution from skewing timestamps. Fence with `lfence` before if using plain `rdtsc`.
- **Coordinated omission** makes naive benchmarks report 10-100x lower tail latencies than reality. Always use rate-based load generation.
- **HDR Histogram** (hdrhistogram.org) provides O(1) recording and O(buckets) percentile queries with configurable precision - the gold standard for latency recording.
- **Warm-up period**: Discard the first 10-30 seconds of samples to avoid JIT, cache cold-start, and OS settling effects.
- Do not use `std::sort` in the hot path - record into a histogram or reservoir sample instead.
- Export percentile data to time-series databases (Prometheus histograms, Grafana) for production monitoring.
