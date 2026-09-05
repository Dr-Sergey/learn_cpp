# Use parallel execution policies (C++17) for algorithm parallelism

**Category:** Standard Library Algorithms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag>  

---

## Topic Overview

C++17 added execution policies that let you parallelize standard algorithms by adding a single argument. The compiler and runtime handle thread management - you do not need to write a single thread or mutex yourself for the common cases. The tradeoff is that you must be careful about what operations you pass to parallel algorithms.

### The Four Execution Policies

| Policy | Header constant | Execution | Notes |
| --- | --- | --- | --- |
| **Sequential** | `std::execution::seq` | Single thread, in order | Same as no policy. Baseline. |
| **Parallel** | `std::execution::par` | Multiple threads | Element access functions must not cause data races. No vectorization. |
| **Parallel unsequenced** | `std::execution::par_unseq` | Multiple threads + SIMD | Must not use mutexes, allocate memory, or call non-lockfree atomics. |
| **Unsequenced** | `std::execution::unseq` (C++20) | Single thread + SIMD | Same restrictions as par_unseq but single-threaded. |

### Key Rule

With `par` or `par_unseq`, any operation you pass (predicates, comparators, projections) **must not cause data races**. The algorithm will invoke your callable from multiple threads simultaneously. This is the rule that trips people up most often, so the third question below shows exactly what a data race looks like and how to avoid it.

### Compiler Support

| Compiler | Backend | Flag |
| --- | --- | --- |
| GCC | Intel TBB | `-ltbb` |
| MSVC | Built-in PPL | (none needed) |
| Clang | Intel TBB | `-ltbb` |

---

## Self-Assessment

### Q1: Add std::execution::par to std::sort and measure speedup on a large vector

The change from sequential to parallel is literally one argument. The code below shows both versions side by side so you can measure the difference on your machine.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <execution>
#include <chrono>
#include <random>
#include <numeric>

int main() {
    constexpr size_t N = 10'000'000;  // 10 million elements

    // Generate random data
    std::vector<int> data(N);
    std::iota(data.begin(), data.end(), 0);
    std::mt19937 rng(42);
    std::shuffle(data.begin(), data.end(), rng);

    // Make copies for fair comparison
    auto data_seq = data;
    auto data_par = data;

    // === Sequential sort ===
    auto t1 = std::chrono::high_resolution_clock::now();
    std::sort(std::execution::seq, data_seq.begin(), data_seq.end());
    auto t2 = std::chrono::high_resolution_clock::now();

    auto ms_seq = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    std::cout << "Sequential sort: " << ms_seq << " ms\n";

    // === Parallel sort ===
    t1 = std::chrono::high_resolution_clock::now();
    std::sort(std::execution::par, data_par.begin(), data_par.end());
    t2 = std::chrono::high_resolution_clock::now();

    auto ms_par = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    std::cout << "Parallel sort:   " << ms_par << " ms\n";

    if (ms_par > 0) {
        std::cout << "Speedup: " << static_cast<double>(ms_seq) / ms_par << "x\n";
    }

    // Verify results are identical
    std::cout << "Results match: " << std::boolalpha
              << (data_seq == data_par) << "\n";

    // Typical output on 4-core machine:
    // Sequential sort: 1200 ms
    // Parallel sort:   380 ms
    // Speedup: 3.15x

    return 0;
}
// Compile: g++ -std=c++17 -O2 -ltbb parallel_sort.cpp
// MSVC:    cl /std:c++17 /O2 parallel_sort.cpp
```

Just add `std::execution::par` as the **first argument** to `std::sort`. The runtime spawns threads internally, typically from a thread pool. Speedup depends on data size, core count, and memory bandwidth - typically 2-4x on a 4-core machine for sort. For small datasets (fewer than roughly 10K elements), parallel overhead exceeds the benefit, so stick to sequential for small inputs.

### Q2: Explain the difference between par, seq, par_unseq, and unseq policies

The table below shows what each policy allows. The key restriction to understand is that `par_unseq` not only runs across threads but also allows SIMD interleaving within a single thread, which makes mutex locking impossible.

| Policy | Threads | SIMD | Safe to use mutexes? | Safe to call virtual functions? |
| --- | --- | --- | --- | --- |
| `seq` | 1 | No | Yes | Yes |
| `par` | Many | No | Yes (carefully) | Yes |
| `par_unseq` | Many | Yes | No (deadlock!) | No |
| `unseq` | 1 | Yes | No | No |

**Detailed explanation:**

- **`seq`** - Guarantees sequential execution in the calling thread. Identical to not passing any policy. Useful as a "switch" variable when you want to parametrize the execution policy at runtime.

- **`par`** - Operations may execute on multiple threads. Each individual element access is done from exactly one thread at a time (no interleaving within a single invocation of your callable). Mutexes are technically safe but will destroy performance - use atomics instead.

- **`par_unseq`** - Operations can execute across threads AND be interleaved (vectorized) within a thread. The reason you cannot use a mutex here is subtle: a single thread might lock the mutex, then get preempted mid-callable to start processing another element, and try to lock the same mutex again. That is a deadlock. Memory allocation also typically uses internal locks, so do not allocate in your callable either.

- **`unseq`** (C++20) - Single thread but allows SIMD vectorization. Same no-mutex, no-allocation restrictions as `par_unseq`.

```cpp
#include <algorithm>
#include <execution>
#include <vector>
#include <mutex>

int main() {
    std::vector<int> v(1000);

    // seq: anything goes
    int sum_seq = 0;
    std::for_each(std::execution::seq, v.begin(), v.end(),
                  [&](int x) { sum_seq += x; });  // Fine

    // par: no data races, but mutexes are OK
    int sum_par = 0;
    std::mutex mtx;
    std::for_each(std::execution::par, v.begin(), v.end(),
                  [&](int x) {
                      std::lock_guard lock(mtx);  // Safe but slow
                      sum_par += x;
                  });

    // par_unseq: NO mutexes, NO allocation, NO virtual calls
    // Use atomics (lock-free only) or avoid shared state entirely
    std::atomic<int> sum_par_unseq{0};
    std::for_each(std::execution::par_unseq, v.begin(), v.end(),
                  [&](int x) {
                      sum_par_unseq.fetch_add(x, std::memory_order_relaxed);
                  });

    return 0;
}
```

### Q3: Show a data race when using par incorrectly with a shared accumulator

This is the most common mistake with parallel algorithms: accumulating into a shared variable without synchronization. The result is always wrong, always different between runs, and the bug is invisible at compile time.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <execution>
#include <numeric>
#include <atomic>

int main() {
    std::vector<int> v(100'000, 1);  // 100K ones

    // === BUG: data race with par ===
    int sum_buggy = 0;
    std::for_each(std::execution::par, v.begin(), v.end(),
                  [&sum_buggy](int x) {
                      sum_buggy += x;  // DATA RACE! Multiple threads writing
                  });
    std::cout << "Buggy sum (should be 100000): " << sum_buggy << "\n";
    // Typical output: some random number like 73842, 91263, etc.
    // Different every run! This is UNDEFINED BEHAVIOR.

    // === Why it's a race ===
    // sum_buggy += x  is:  temp = sum_buggy; temp += x; sum_buggy = temp;
    // Thread A reads sum_buggy = 50
    // Thread B reads sum_buggy = 50  (before A writes back!)
    // Thread A writes 51
    // Thread B writes 51  -> lost update! Should be 52.

    // === Fix 1: Use std::atomic (works with par) ===
    std::atomic<int> sum_atomic{0};
    std::for_each(std::execution::par, v.begin(), v.end(),
                  [&sum_atomic](int x) {
                      sum_atomic.fetch_add(x, std::memory_order_relaxed);
                  });
    std::cout << "Atomic sum: " << sum_atomic.load() << "\n";  // 100000

    // === Fix 2: Use std::reduce (designed for parallelism) ===
    int sum_reduce = std::reduce(std::execution::par, v.begin(), v.end(), 0);
    std::cout << "Reduce sum: " << sum_reduce << "\n";  // 100000

    // === Fix 3: Use std::transform_reduce for more complex cases ===
    // e.g., sum of squares
    long long sum_sq = std::transform_reduce(
        std::execution::par,
        v.begin(), v.end(),
        0LL,                            // init
        std::plus<>{},                  // reduce op
        [](int x) -> long long { return static_cast<long long>(x) * x; }  // transform
    );
    std::cout << "Sum of squares: " << sum_sq << "\n";  // 100000

    return 0;
}
```

The race manifests because `sum += x` is a read-modify-write sequence, and it is not atomic. Multiple threads interleave their reads and writes, causing lost updates. The result is non-deterministic and always less than the correct answer. This is **undefined behavior** per the C++ standard, not just a logic bug. The best fix is not to accumulate into shared state at all - use `std::reduce` or `std::transform_reduce`, which are specifically designed for parallel reduction and handle the thread-safe accumulation internally.

---

## Notes

- **When to use parallel policies:** Only for large data sets (roughly 10K-100K elements or more, depending on the operation). Parallel overhead dominates for small inputs.
- **Exception handling:** Parallel algorithms call `std::terminate()` if any callable throws. They cannot propagate exceptions across threads.
- **No guarantees on thread count:** The implementation may fall back to sequential execution for any policy.
- **`std::sort` with `par`** - Typically uses parallel merge sort instead of introsort. May have different cache behavior.
- **Algorithms that support policies:** `sort`, `for_each`, `transform`, `reduce`, `find`, `count`, `copy`, `fill`, and most others in `<algorithm>` and `<numeric>`.
- **Compile flags:** GCC/Clang need `-ltbb` to link Intel TBB. MSVC uses PPL by default.
