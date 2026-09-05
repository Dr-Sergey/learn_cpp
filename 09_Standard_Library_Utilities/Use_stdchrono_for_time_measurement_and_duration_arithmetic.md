# Use std::chrono for time measurement and duration arithmetic

**Category:** Standard Library Utilities  
**Standard:** C++11, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/chrono>  

---

## Topic Overview

The `<chrono>` library provides type-safe time utilities: **clocks** (source of time), **time points** (an instant in time), and **durations** (a span of time). The type system prevents accidental mixing of incompatible time units.

The reason this library feels a bit verbose at first is that it encodes dimensional analysis into the type system. You cannot accidentally pass seconds where milliseconds are expected, and you cannot silently lose precision by assigning a millisecond value to a seconds variable. The compiler catches those mistakes for you - the price is that you sometimes have to write `duration_cast` explicitly, but that is a feature: it makes lossy conversions visible.

### Three Main Clocks

| Clock | Properties | Use for |
| --- | --- | --- |
| `std::chrono::system_clock` | Wall clock, adjustable, epoch = Unix epoch | Calendar dates, file timestamps |
| `std::chrono::steady_clock` | Monotonic, not adjustable | Measuring elapsed time, timeouts |
| `std::chrono::high_resolution_clock` | Alias for one of the above | Avoid - use `steady_clock` explicitly |

### Duration Types

These are all aliases for the same `duration<Rep, Period>` template. The key thing to understand is that the `Rep` is the underlying numeric type and `Period` is the ratio to one second.

```cpp
using nanoseconds  = duration<int64_t, std::nano>;
using microseconds = duration<int64_t, std::micro>;
using milliseconds = duration<int64_t, std::milli>;
using seconds      = duration<int64_t>;
using minutes      = duration<int64_t, std::ratio<60>>;
using hours        = duration<int64_t, std::ratio<3600>>;
```

### C++20 Additions

- Calendar types: `year`, `month`, `day`, `year_month_day`
- Time zones: `std::chrono::zoned_time`, `current_zone()`
- Formatting: `std::format("{:%Y-%m-%d}", tp)`

### Core Syntax

Here is the full basic pattern in one place: take a timestamp, do some work, take another, subtract them, cast to the unit you want, and read `.count()`.

```cpp
#include <chrono>
#include <iostream>
#include <thread>

int main() {
    using namespace std::chrono;
    using namespace std::chrono_literals;  // for 1s, 500ms, etc.

    // --- Measure elapsed time ---
    auto start = steady_clock::now();
    std::this_thread::sleep_for(100ms);
    auto end = steady_clock::now();

    auto elapsed = duration_cast<microseconds>(end - start);
    std::cout << "elapsed: " << elapsed.count() << " µs\n";
    // Output: elapsed: ~100000 µs

    // --- Duration arithmetic ---
    auto total = 2h + 30min + 15s;
    std::cout << "total seconds: " << duration_cast<seconds>(total).count() << "\n";
    // Output: total seconds: 9015

    // --- Type-safe: prevents truncation without explicit cast ---
    seconds s = 65s;
    // minutes m = s;             // ERROR: lossy conversion
    minutes m = duration_cast<minutes>(s);  // OK: explicit
    std::cout << "65s = " << m.count() << " min (truncated)\n";
    // Output: 65s = 1 min (truncated)
}
```

---

## Self-Assessment

### Q1: Measure the runtime of a function using steady_clock with microsecond resolution

**Answer:**

The comment in the code is important: `steady_clock` is the right choice for elapsed-time measurement because it is guaranteed to be monotonic. `system_clock` can jump backwards if an NTP sync happens mid-measurement.

```cpp
#include <chrono>
#include <iostream>
#include <vector>
#include <numeric>
#include <algorithm>

// Function to benchmark
long long compute(int n) {
    std::vector<int> v(n);
    std::iota(v.begin(), v.end(), 1);
    std::sort(v.begin(), v.end(), std::greater<>());
    return std::accumulate(v.begin(), v.end(), 0LL);
}

// Generic benchmark utility
template<typename Func>
auto benchmark_us(Func&& f) {
    auto start = std::chrono::steady_clock::now();
    auto result = f();
    auto end = std::chrono::steady_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    return std::pair{result, us};
}

int main() {
    // Simple measurement
    auto t1 = std::chrono::steady_clock::now();
    auto result = compute(1'000'000);
    auto t2 = std::chrono::steady_clock::now();

    auto elapsed = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1);
    std::cout << "compute(1M): " << elapsed.count() << " µs, result = " << result << "\n";

    // Using the benchmark utility
    auto [val, time] = benchmark_us([&]{ return compute(500'000); });
    std::cout << "compute(500K): " << time.count() << " µs, result = " << val << "\n";

    // Why steady_clock and not system_clock:
    // - steady_clock is monotonic - cannot go backwards (no NTP adjustments)
    // - system_clock can jump forward/backward due to time sync
    // - For measuring durations, steady_clock is the only correct choice

    static_assert(std::chrono::steady_clock::is_steady,
                  "steady_clock must be monotonic");
}
```

---

### Q2: Use duration_cast to convert between seconds, milliseconds, and nanoseconds

**Answer:**

The rule is straightforward once you see it: widening (smaller unit to larger precision) is implicit and safe; narrowing (larger unit to smaller precision, which truncates) requires an explicit `duration_cast`. C++17 also added `floor`, `ceil`, and `round` for when truncation toward zero is not what you want.

```cpp
#include <chrono>
#include <iostream>

int main() {
    using namespace std::chrono;
    using namespace std::chrono_literals;

    // --- Widening conversions: implicit (no loss) ---
    seconds s = 3s;
    milliseconds ms = s;        // OK: 3000ms, no precision loss
    microseconds us = ms;       // OK: 3000000µs
    nanoseconds ns = us;        // OK: 3000000000ns

    std::cout << s.count()  << " s  = "
              << ms.count() << " ms = "
              << us.count() << " µs = "
              << ns.count() << " ns\n";
    // Output: 3 s  = 3000 ms = 3000000 µs = 3000000000 ns

    // --- Narrowing conversions: require duration_cast (lossy) ---
    milliseconds ms2 = 2567ms;
    seconds s2 = duration_cast<seconds>(ms2);   // truncates to 2
    std::cout << "2567ms -> " << s2.count() << "s (truncated)\n";
    // Output: 2567ms -> 2s (truncated)

    // --- floor, ceil, round (C++17) ---
    auto floored = floor<seconds>(ms2);    // 2s  (toward negative infinity)
    auto ceiled  = ceil<seconds>(ms2);     // 3s  (toward positive infinity)
    auto rounded = round<seconds>(ms2);    // 3s  (nearest, round half to even)

    std::cout << "floor: " << floored.count() << "s\n";
    std::cout << "ceil:  " << ceiled.count()  << "s\n";
    std::cout << "round: " << rounded.count() << "s\n";
    // Output:
    // floor: 2s
    // ceil:  3s
    // round: 3s

    // --- Mixed arithmetic ---
    auto total = 1h + 30min + 45s + 500ms;
    std::cout << "total ms: " << duration_cast<milliseconds>(total).count() << "\n";
    // Output: total ms: 5445500

    // Convert to floating-point seconds
    std::chrono::duration<double> fp_seconds = total;
    std::cout << "total: " << fp_seconds.count() << " seconds\n";
    // Output: total: 5445.5 seconds
}
```

---

### Q3: Show the type-safe duration arithmetic that prevents accidentally mixing seconds and milliseconds

**Answer:**

This is one of those features where the benefit only becomes obvious when you imagine the bug it prevents. A function that takes `milliseconds` simply will not compile if you pass a raw integer or a duration of the wrong type without an explicit acknowledgment.

```cpp
#include <chrono>
#include <iostream>

void process_timeout(std::chrono::milliseconds timeout) {
    std::cout << "timeout: " << timeout.count() << " ms\n";
}

int main() {
    using namespace std::chrono;
    using namespace std::chrono_literals;

    // --- Type safety: cannot pass raw integers ---
    // process_timeout(5000);       // COMPILE ERROR: int is not milliseconds
    process_timeout(5000ms);        // OK: explicitly milliseconds
    process_timeout(5s);            // OK: implicit widening (5s -> 5000ms)

    // --- Cannot accidentally lose precision ---
    milliseconds ms = 1500ms;
    // seconds s = ms;              // COMPILE ERROR: would lose the .5
    seconds s = duration_cast<seconds>(ms);  // OK: explicit truncation

    // --- Arithmetic preserves the finest resolution ---
    auto result = 1s + 500ms;  // type is milliseconds (finest common unit)
    std::cout << "1s + 500ms = " << result.count() << " ms\n";
    // Output: 1s + 500ms = 1500 ms

    auto mix = 1min + 30s + 250ms;
    std::cout << "1min + 30s + 250ms = "
              << duration_cast<milliseconds>(mix).count() << " ms\n";
    // Output: 1min + 30s + 250ms = 90250 ms

    // --- Comparison is type-safe ---
    if (500ms < 1s) {
        std::cout << "500ms < 1s\n";  // works across types
    }

    // --- Custom duration types ---
    using frames = duration<int, std::ratio<1, 60>>;    // 1/60 second
    using ticks  = duration<int, std::ratio<1, 1000>>;  // 1ms ticks

    frames f{120};  // 120 frames at 60fps = 2 seconds
    auto as_ms = duration_cast<milliseconds>(f);
    std::cout << "120 frames @ 60fps = " << as_ms.count() << " ms\n";
    // Output: 120 frames @ 60fps = 2000 ms

    // Cannot mix incompatible duration types without cast:
    // ticks t = f;  // May or may not compile depending on ratio conversion
    ticks t = duration_cast<ticks>(f);
    std::cout << "as ticks: " << t.count() << "\n";
    // Output: as ticks: 2000
}
```

**Key type-safety benefits:**

- Raw `int` values cannot be accidentally used as durations.
- Widening conversions (seconds -> milliseconds) are implicit and safe.
- Narrowing conversions (milliseconds -> seconds) require explicit `duration_cast`.
- Mixed arithmetic returns the finest common unit.
- Comparisons work correctly across different duration types.

---

## Notes

- Always use `steady_clock` for measuring elapsed time - `system_clock` can jump due to NTP.
- Use `std::chrono_literals` (`1s`, `500ms`, `100us`, `50ns`) for readable code.
- `duration<double>` allows floating-point durations without casting.
- C++20 adds `std::chrono::days`, `std::chrono::weeks`, `std::chrono::months`, `std::chrono::years`.
- Avoid `.count()` where possible - work with duration types to maintain type safety. Use `.count()` only at system boundaries (printing, APIs that take raw numbers).
- `high_resolution_clock` is implementation-defined - on some platforms it's `system_clock` (non-monotonic). Always prefer `steady_clock`.
