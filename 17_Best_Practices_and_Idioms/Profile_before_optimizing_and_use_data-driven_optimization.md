# Profile before optimizing and use data-driven optimization

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/performance>  

---

## Topic Overview

*"Premature optimization is the root of all evil"* - Donald Knuth. The reason this quote has lasted fifty years is that programmers are reliably bad at guessing where the time actually goes. **Always measure first, then optimize the measured hotspot.** Time spent optimizing a function that accounts for 2% of runtime is almost always time wasted.

### The Optimization Workflow

The discipline of data-driven optimization looks like this:

```cpp
1. Write correct code first
2. Profile under realistic workload
3. Identify the hotspot (top 1-3 functions)
4. Hypothesize improvement
5. Implement change
6. Re-profile to verify improvement
7. If not faster, revert!
```

Step 7 is the one people skip. If your change did not move the numbers, revert it - complexity without benefit is pure cost.

### Profiling Tools

| Tool | Platform | What it measures |
| --- | --- | --- |
| `perf stat/record` | Linux | CPU cycles, instructions, cache misses |
| Valgrind/Cachegrind | Linux | Cache misses, instruction count |
| VTune | Cross-platform | CPU microarchitecture, threading |
| `std::chrono` | Portable | Wall clock time (micro-benchmarks) |
| Google Benchmark | Portable | Statistical micro-benchmarking |
| Visual Studio Profiler | Windows | CPU sampling, memory |

---

## Self-Assessment

### Q1: Profile a program and identify the hottest function

This example uses `std::chrono` as a simple manual profiler. In real code you would reach for `perf` or VTune, but the principle is the same - time each function separately and let the numbers tell you where to focus.

```cpp
#include <chrono>
#include <cmath>
#include <iostream>
#include <numeric>
#include <vector>
#include <algorithm>
#include <utility>

// Simulate a program with multiple functions
void fast_function(std::vector<double>& data) {
    for (auto& x : data) x = x + 1.0;
}

void slow_function(std::vector<double>& data) {
    for (auto& x : data) x = std::sin(x) * std::cos(x) * std::exp(-std::abs(x) * 0.001);
}

void medium_function(std::vector<double>& data) {
    std::sort(data.begin(), data.end());
}

int main() {
    constexpr size_t N = 1'000'000;
    std::vector<double> data(N);
    std::iota(data.begin(), data.end(), 0.0);

    // Manual profiling with chrono
    auto measure = [](auto&& func, auto&&... args) {
        auto start = std::chrono::high_resolution_clock::now();
        func(std::forward<decltype(args)>(args)...);
        auto end = std::chrono::high_resolution_clock::now();
        return std::chrono::duration<double, std::milli>(end - start).count();
    };

    double t1 = measure(fast_function, data);
    double t2 = measure(slow_function, data);     // <-- likely hottest
    double t3 = measure(medium_function, data);

    std::cout << "fast_function:   " << t1 << " ms\n";
    std::cout << "slow_function:   " << t2 << " ms (HOTSPOT)\n";
    std::cout << "medium_function: " << t3 << " ms\n";
    std::cout << "Total: " << (t1 + t2 + t3) << " ms\n";
    std::cout << "Hotspot is " << (t2 / (t1 + t2 + t3) * 100) << "% of total\n";
}
// Typical output:
// fast_function:   ~1 ms
// slow_function:   ~50 ms (HOTSPOT)
// medium_function: ~70 ms
// Hotspot is ~40% of total
//
// Linux: perf record ./program && perf report
// Shows: slow_function at top of profile
```

Once you have the numbers, the decision of where to invest effort is obvious - not a guess.

### Q2: Show a premature optimization that hurt readability without measurable benefit

The reason this trip-up is so common is that "clever" low-level tricks feel fast. But modern compilers are sophisticated enough to produce the same machine code from the readable version - so you pay the readability cost for nothing.

```cpp
#include <chrono>
#include <iostream>
#include <vector>

// "Optimized" version: bit tricks, unreadable
int count_set_bits_clever(int n) {
    n = n - ((n >> 1) & 0x55555555);
    n = (n & 0x33333333) + ((n >> 2) & 0x33333333);
    return (((n + (n >> 4)) & 0x0F0F0F0F) * 0x01010101) >> 24;
}

// Simple version: readable
int count_set_bits_simple(int n) {
    int count = 0;
    while (n) { count += n & 1; n >>= 1; }
    return count;
}

// Modern C++20: use the standard library
// #include <bit>
// int count_set_bits_modern(int n) { return std::popcount(static_cast<unsigned>(n)); }

int main() {
    constexpr int N = 10'000'000;
    int sink = 0;

    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) sink += count_set_bits_clever(i);
    auto t2 = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < N; ++i) sink += count_set_bits_simple(i);
    auto t3 = std::chrono::high_resolution_clock::now();

    auto clever_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto simple_ms = std::chrono::duration<double, std::milli>(t3 - t2).count();

    std::cout << "Clever: " << clever_ms << " ms\n";
    std::cout << "Simple: " << simple_ms << " ms\n";
    std::cout << "(sink=" << sink << ")\n";
    // Modern compilers optimize the simple version equally well!
    // The clever version added complexity for no benefit.
}
// Typical output (-O2):
// Clever: ~25 ms
// Simple: ~25 ms   <-- same speed! Compiler optimizes both.
// Lesson: the compiler is smarter than you think.
```

The modern C++20 answer (`std::popcount`) is both the most readable and likely the fastest - it can compile directly to a single hardware instruction. Start with the clearest version; only complicate it when profiling shows you need to.

### Q3: Apply a data-oriented refactoring to a hot loop and measure the speedup

This is where profiling-driven optimization really pays off. The Array-of-Structs (AoS) layout scatters the hot data (positions and velocities) across large structs, which kills cache efficiency. Switching to Struct-of-Arrays (SoA) packs the hot fields together - the CPU's prefetcher loves it.

```cpp
#include <chrono>
#include <cmath>
#include <iostream>
#include <vector>

// BEFORE: AoS with pointer chasing and virtual calls
struct ParticleSlow {
    double x, y, z;
    double vx, vy, vz;
    double mass;
    char padding[64];  // simulate real-world bloated struct
};

void update_slow(std::vector<ParticleSlow>& particles, double dt) {
    for (auto& p : particles) {
        p.x += p.vx * dt;  // accesses scattered through 128-byte structs
        p.y += p.vy * dt;
        p.z += p.vz * dt;
    }
}

// AFTER: SoA, hot data contiguous
struct ParticlesFast {
    std::vector<double> x, y, z;
    std::vector<double> vx, vy, vz;
    // mass, padding not in hot path
};

void update_fast(ParticlesFast& p, double dt, size_t n) {
    for (size_t i = 0; i < n; ++i) p.x[i] += p.vx[i] * dt;
    for (size_t i = 0; i < n; ++i) p.y[i] += p.vy[i] * dt;
    for (size_t i = 0; i < n; ++i) p.z[i] += p.vz[i] * dt;
}

int main() {
    constexpr size_t N = 2'000'000;
    constexpr double dt = 0.016;

    // Setup AoS
    std::vector<ParticleSlow> slow(N);
    for (size_t i = 0; i < N; ++i)
        slow[i] = {double(i), 0, 0, 1, 2, 3, 1, {}};

    // Setup SoA
    ParticlesFast fast;
    fast.x.resize(N); fast.y.resize(N); fast.z.resize(N);
    fast.vx.resize(N); fast.vy.resize(N); fast.vz.resize(N);
    for (size_t i = 0; i < N; ++i) {
        fast.x[i] = double(i); fast.vx[i] = 1;
        fast.vy[i] = 2; fast.vz[i] = 3;
    }

    auto t1 = std::chrono::high_resolution_clock::now();
    update_slow(slow, dt);
    auto t2 = std::chrono::high_resolution_clock::now();
    update_fast(fast, dt, N);
    auto t3 = std::chrono::high_resolution_clock::now();

    auto ms_slow = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto ms_fast = std::chrono::duration<double, std::milli>(t3 - t2).count();

    std::cout << "AoS (slow): " << ms_slow << " ms\n";
    std::cout << "SoA (fast): " << ms_fast << " ms\n";
    std::cout << "Speedup: " << (ms_slow / ms_fast) << "x\n";
}
// Typical output:
// AoS (slow): ~30 ms
// SoA (fast): ~8 ms
// Speedup: ~3.7x
```

A nearly 4x speedup from a layout change, with no algorithmic change at all - this is exactly the kind of win that profiling reveals and that you would never find by guessing.

---

## Notes

- **Amdahl's Law:** If a function is 5% of runtime, a 2x speedup saves only 2.5% total time. Optimize the 80%, not the 5%.
- Always compile with `-O2` or `-O3` for profiling. Debug builds have different hotspots.
- Use `[[gnu::noinline]]` or `__attribute__((noinline))` to prevent inlining when profiling specific functions.
- Google Benchmark provides statistical analysis: mean, median, stddev across runs.
