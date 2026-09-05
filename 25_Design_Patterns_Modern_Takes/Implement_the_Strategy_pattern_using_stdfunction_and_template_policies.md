# Implement the Strategy pattern using std::function and template policies

**Category:** Design Patterns Modern Takes  
**Standard:** C++11  
**Reference:** <https://en.wikipedia.org/wiki/Strategy_pattern>  

---

## Topic Overview

The **Strategy pattern** externalizes an algorithm so it can be swapped at runtime or compile time. Modern C++ offers two approaches: `std::function` for **runtime flexibility** (swap strategies dynamically) and **template policies** for **zero-overhead compile-time binding** (the compiler inlines the strategy entirely).

The choice between them comes down to one question: do you need to change the strategy *while the program is running*, or do you know the strategy at compile time? If you know it at compile time, templates give you code that is just as fast as writing the algorithm directly. If you need runtime swapping - for plugins, user config, or dynamic dispatch - `std::function` is the tool.

### Two Approaches Compared

| Aspect | `std::function` | Template Policy |
| --- | --- | --- |
| Binding time | Runtime | Compile time |
| Overhead | Type erasure (~40 bytes, virtual call) | Zero - fully inlined |
| Swap strategy? | Yes, at runtime | No, fixed at compile time |
| Binary size | Smaller (one instantiation) | Larger (one per policy) |
| Use case | Plugin systems, user config | Performance-critical hot paths |

---

## Self-Assessment

### Q1: Implement a Sorter class that accepts a comparison strategy as a std::function<bool(T,T)>

Here the `Sorter` class stores the strategy as a `std::function`. The nice part is that you can pass any callable - a lambda, a functor, a function pointer - and the `Sorter` doesn't need to know what it is. It can also be replaced at runtime via `set_strategy`, which is something templates simply can't do.

**Answer:**

```cpp
#include <iostream>
#include <vector>
#include <functional>
#include <algorithm>
#include <string>

// Runtime Strategy with std::function
template<typename T>
class Sorter {
    std::function<bool(const T&, const T&)> compare_;
    std::string strategy_name_;

public:
    Sorter(std::function<bool(const T&, const T&)> cmp, std::string name = "custom")
        : compare_(std::move(cmp)), strategy_name_(std::move(name)) {}

    void sort(std::vector<T>& data) {
        std::sort(data.begin(), data.end(), compare_);
    }

    // Swap strategy at runtime
    void set_strategy(std::function<bool(const T&, const T&)> cmp,
                      std::string name = "custom") {
        compare_ = std::move(cmp);
        strategy_name_ = std::move(name);
    }

    const std::string& name() const { return strategy_name_; }
};

int main() {
    std::vector<int> data = {5, 2, 8, 1, 9, 3};

    // Strategy 1: ascending
    Sorter<int> sorter([](int a, int b) { return a < b; }, "ascending");
    sorter.sort(data);
    std::cout << sorter.name() << ": ";
    for (int x : data) std::cout << x << ' ';  // 1 2 3 5 8 9
    std::cout << '\n';

    // Swap to strategy 2: descending - at runtime!
    sorter.set_strategy([](int a, int b) { return a > b; }, "descending");
    sorter.sort(data);
    std::cout << sorter.name() << ": ";
    for (int x : data) std::cout << x << ' ';  // 9 8 5 3 2 1
    std::cout << '\n';

    // Strategy 3: by absolute distance from 5
    sorter.set_strategy([](int a, int b) {
        return std::abs(a - 5) < std::abs(b - 5);
    }, "nearest-to-5");
    sorter.sort(data);
    std::cout << sorter.name() << ": ";
    for (int x : data) std::cout << x << ' ';
    std::cout << '\n';
}
```

Watch how `set_strategy` replaces the entire comparison function between sort calls. The `Sorter` object doesn't change - only what's stored inside it does. That's the runtime flexibility `std::function` buys you.

### Q2: Replace std::function with a template policy parameter and benchmark the inlining benefit

When the strategy is baked into the type at compile time, the compiler can see through it. There's no function pointer to follow, no type erasure to undo. The call to `cmp()` inside `std::sort` gets inlined directly, which also enables vectorization and better branch prediction.

The benchmark here isn't just academic. On sorting 1 million elements, the `std::function` overhead can be 15-30% - because every comparison makes an indirect call that prevents the optimizer from doing its best work.

**Answer:**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>
#include <functional>

// Template policy: zero-overhead strategy
template<typename T, typename ComparePolicy>
class PolicySorter {
public:
    void sort(std::vector<T>& data) {
        ComparePolicy cmp;
        std::sort(data.begin(), data.end(), cmp);
        // cmp is inlined - no indirect call, no type erasure
    }
};

// Policy types
struct Ascending {
    bool operator()(int a, int b) const { return a < b; }
};
struct Descending {
    bool operator()(int a, int b) const { return a > b; }
};

// Benchmark: std::function vs template policy
int main() {
    constexpr int N = 1'000'000;
    std::vector<int> data(N);

    // Setup
    auto fill_random = [&] {
        for (int i = 0; i < N; ++i) data[i] = N - i;
    };

    // Benchmark std::function
    std::function<bool(int, int)> runtime_cmp = [](int a, int b) { return a < b; };
    fill_random();
    auto t1 = std::chrono::high_resolution_clock::now();
    std::sort(data.begin(), data.end(), runtime_cmp);
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark template policy
    PolicySorter<int, Ascending> policy_sorter;
    fill_random();
    auto t3 = std::chrono::high_resolution_clock::now();
    policy_sorter.sort(data);
    auto t4 = std::chrono::high_resolution_clock::now();

    // Benchmark raw lambda (baseline)
    fill_random();
    auto t5 = std::chrono::high_resolution_clock::now();
    std::sort(data.begin(), data.end(), [](int a, int b) { return a < b; });
    auto t6 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::duration<double, std::milli>;
    std::cout << "std::function: " << ms(t2 - t1).count() << " ms\n";
    std::cout << "template policy: " << ms(t4 - t3).count() << " ms\n";
    std::cout << "raw lambda:      " << ms(t6 - t5).count() << " ms\n";
    // Typical: template policy ~= raw lambda < std::function
    // std::function overhead: ~15-30% slower due to indirect call preventing inlining
}
```

You should see template policy and raw lambda come out roughly even, while `std::function` lags noticeably. The template policy is not faster than a raw lambda because it *is* essentially a raw lambda - the compiler can see the exact type and eliminate the abstraction layer entirely.

### Q3: Show that template policies enable zero-overhead abstraction while std::function enables runtime flexibility

This example puts both approaches side by side in a realistic scenario: a logging policy for a service. With templates, `NullLog` compiles to absolutely nothing - the compiler sees an empty function and removes every log call. With `std::function`, you can swap the logger based on runtime conditions like configuration or environment.

**Answer:**

```cpp
#include <iostream>
#include <string>
#include <functional>
#include <memory>

// Use case: logging with strategy

// APPROACH 1: Template policy - zero overhead, compile-time fixed
struct ConsoleLog {
    void log(const std::string& msg) const { std::cout << "[CONSOLE] " << msg << '\n'; }
};
struct FileLog {
    void log(const std::string& msg) const { std::cout << "[FILE] " << msg << '\n'; }
};
struct NullLog {
    void log(const std::string&) const {}  // Completely optimized away!
};

template<typename LogPolicy = ConsoleLog>
class Service {
    LogPolicy logger_;
public:
    void process(const std::string& data) {
        logger_.log("Processing: " + data);  // Inlined; NullLog = zero cost
        // ... actual work ...
        logger_.log("Done: " + data);
    }
};

// APPROACH 2: std::function - runtime swappable
class FlexibleService {
    std::function<void(const std::string&)> log_;
public:
    explicit FlexibleService(std::function<void(const std::string&)> logger)
        : log_(std::move(logger)) {}

    void set_logger(std::function<void(const std::string&)> logger) {
        log_ = std::move(logger);  // Swap at runtime!
    }

    void process(const std::string& data) {
        log_("Processing: " + data);  // Indirect call through std::function
        // ... actual work ...
        log_("Done: " + data);
    }
};

int main() {
    // Template policy - logger chosen at compile time
    Service<ConsoleLog> s1;
    s1.process("request A");

    Service<NullLog> s2;       // NullLog::log is empty - compiler removes all log calls!
    s2.process("request B");   // Zero logging overhead

    // std::function - logger swapped at runtime
    FlexibleService fs([](const std::string& msg) {
        std::cout << "[RUNTIME] " << msg << '\n';
    });
    fs.process("request C");

    // Swap to file logging at runtime
    fs.set_logger([](const std::string& msg) {
        std::cout << "[SWITCHED] " << msg << '\n';
    });
    fs.process("request D");
}
```

The `NullLog` trick is particularly elegant. In a test build or a production build where you want to strip all logging, you instantiate `Service<NullLog>` and the optimizer removes every log statement. There's no `#ifdef` needed - the zero-cost abstraction does it for you.

---

## Notes

- **Rule of thumb**: use template policies for hot paths where performance matters; use `std::function` for configuration/plugin points.
- `NullLog` policy demonstrates true zero-overhead: the compiler can eliminate dead code entirely.
- Template policies work naturally with CRTP, concepts, and deduction guides.
- `std::function` captures state (closures) which template policies cannot without a functor member - use `std::function` when strategies need captured runtime context that wasn't known at compile time.
