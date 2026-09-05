# Apply the policy-based design pattern with templates

**Category:** Modern OOP Patterns  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Policy-based_design>  

---

## Topic Overview

**Policy-based design** (pioneered by Andrei Alexandrescu in *Modern C++ Design*) uses **template parameters as policies** - small classes that encapsulate a single behavioral aspect. The host class composes its behavior from policies selected at compile time, achieving customization with **zero runtime overhead**.

The mental model is simple: instead of using virtual functions to plug in different behaviors, you plug in different types. The compiler sees the exact concrete type at instantiation, inlines everything, and produces code as fast as if you'd written it by hand.

### Architecture

Here's why this matters. Compare the two approaches directly:

```cpp
Host<PolicyA, PolicyB>                    vs      Virtual Dispatch
┌──────────────────────────┐             ┌──────────────────────────┐
│ class Logger<Output, Time>│             │ class Logger {           │
│   : public Output,        │             │   IOutput* out_;  <-vtable│
│     public Time {          │             │   ITime*   time_; <-vtable│
│   void log(msg) {         │             │   void log(msg) {       │
│     auto t = timestamp(); │ <- inlined   │     auto t = time_->now()│ <- virtual call
│     write(t + msg);       │ <- inlined   │     out_->write(t+msg); │ <- virtual call
│   }                       │             │   }                     │
│ };                        │             │ };                      │
└──────────────────────────┘             └──────────────────────────┘
  Compile-time composition                 Runtime composition
  Zero overhead (inlined)                  vtable + indirection overhead
  Types differ: Logger<A,B> != Logger<C,D>  Single type: Logger
```

The trade-off is that each policy combination produces a different type. `Logger<ConsoleOutput, FullTimestamp>` is not the same type as `Logger<FileOutput, NoTimestamp>`. You lose runtime flexibility in exchange for compile-time performance.

### Policy vs Strategy

| Aspect | Policy (Compile-Time) | Strategy (Runtime) |
| --- | --- | --- |
| Selection time | Compile time | Runtime |
| Mechanism | Template parameter | Virtual function / `std::function` |
| Overhead | Zero (inlined) | Virtual call / type erasure |
| Flexibility | Fixed at compile time | Switchable at runtime |
| Type identity | Each combination is a different type | Same type |

---

## Self-Assessment

### Q1: Write a `Logger<OutputPolicy, TimestampPolicy>` that composes behavior from policy templates

Notice that the host class inherits *privately* from both policies. This is the canonical policy-based composition technique - it gives the host access to the policy methods without exposing them in the public interface.

**Solution - Policy-Based Logger:**

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <chrono>
#include <ctime>
#include <sstream>

// === Output Policies ===
struct ConsoleOutput {
    void write(const std::string& msg) {
        std::cout << msg << "\n";
    }
};

struct FileOutput {
    std::ofstream file;
    FileOutput() : file("app.log", std::ios::app) {}
    void write(const std::string& msg) {
        file << msg << "\n";
    }
};

struct NullOutput {
    void write(const std::string&) {}  // discard
};

// === Timestamp Policies ===
struct FullTimestamp {
    std::string timestamp() {
        auto now = std::chrono::system_clock::now();
        auto time_t = std::chrono::system_clock::to_time_t(now);
        char buf[64];
        std::strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", std::localtime(&time_t));
        return buf;
    }
};

struct ShortTimestamp {
    std::string timestamp() {
        auto now = std::chrono::system_clock::now();
        auto time_t = std::chrono::system_clock::to_time_t(now);
        char buf[32];
        std::strftime(buf, sizeof(buf), "%H:%M:%S", std::localtime(&time_t));
        return buf;
    }
};

struct NoTimestamp {
    std::string timestamp() { return ""; }
};

// === Host class: composes policies ===
template <typename OutputPolicy, typename TimestampPolicy>
class Logger : private OutputPolicy, private TimestampPolicy {
public:
    void log(const std::string& message) {
        std::string ts = this->timestamp();
        std::string formatted = ts.empty() ? message : "[" + ts + "] " + message;
        this->write(formatted);
    }

    // Expose policies for customization
    OutputPolicy& output_policy() { return *this; }
    TimestampPolicy& timestamp_policy() { return *this; }
};

int main() {
    // Different logger configurations - all resolved at compile time
    Logger<ConsoleOutput, FullTimestamp> prod_logger;
    prod_logger.log("Application started");
    prod_logger.log("Processing request #42");

    Logger<ConsoleOutput, NoTimestamp> simple_logger;
    simple_logger.log("Simple message without timestamp");

    Logger<NullOutput, NoTimestamp> null_logger;
    null_logger.log("This is discarded");  // zero overhead

    // Each combination is a DIFFERENT type:
    // Logger<ConsoleOutput, FullTimestamp> != Logger<FileOutput, ShortTimestamp>
    static_assert(!std::is_same_v<decltype(prod_logger), decltype(simple_logger)>);
}
// Expected output (timestamps will vary):
//   [2024-01-15 14:30:00] Application started
//   [2024-01-15 14:30:00] Processing request #42
//   Simple message without timestamp
```

The `NullOutput` + `NoTimestamp` combination is a neat trick: the compiler sees that `write` does nothing and `timestamp` returns an empty string, and it can optimize the entire `log` call away to nothing. That's the "zero overhead" promise in action.

---

### Q2: Compare policy-based design with virtual dispatch for a sorting strategy

Seeing the two approaches for the same problem makes the trade-off concrete. Virtual dispatch wins on flexibility; policy wins on performance.

**Solution - Same Problem, Two Approaches:**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>
#include <chrono>

// ===== Approach 1: Virtual Dispatch (Strategy Pattern) =====

class ISortStrategy {
public:
    virtual ~ISortStrategy() = default;
    virtual void sort(std::vector<int>& data) = 0;
};

class AscendingSort : public ISortStrategy {
public:
    void sort(std::vector<int>& data) override {
        std::sort(data.begin(), data.end());
    }
};

class DescendingSort : public ISortStrategy {
public:
    void sort(std::vector<int>& data) override {
        std::sort(data.begin(), data.end(), std::greater<>{});
    }
};

class VirtualSorter {
    ISortStrategy* strategy_;
public:
    explicit VirtualSorter(ISortStrategy* s) : strategy_(s) {}
    void set_strategy(ISortStrategy* s) { strategy_ = s; }  // can change at runtime
    void sort(std::vector<int>& data) { strategy_->sort(data); }  // virtual call
};

// ===== Approach 2: Policy-Based Design =====

struct AscPolicy {
    static void sort(std::vector<int>& data) {
        std::sort(data.begin(), data.end());
    }
};

struct DescPolicy {
    static void sort(std::vector<int>& data) {
        std::sort(data.begin(), data.end(), std::greater<>{});
    }
};

template <typename SortPolicy>
class PolicySorter {
public:
    void sort(std::vector<int>& data) {
        SortPolicy::sort(data);  // static call - inlined!
    }
    // Cannot change policy at runtime (it's a template parameter)
};

int main() {
    std::vector<int> data = {5, 2, 8, 1, 9, 3};

    // Virtual dispatch: can switch strategy at runtime
    AscendingSort asc;
    DescendingSort desc;
    VirtualSorter v_sorter(&asc);
    v_sorter.sort(data);  // ascending via virtual call
    std::cout << "Virtual (asc): ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";

    v_sorter.set_strategy(&desc);  // switch at runtime!
    v_sorter.sort(data);
    std::cout << "Virtual (desc): ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";

    // Policy-based: resolved at compile time
    PolicySorter<AscPolicy> p_asc;
    PolicySorter<DescPolicy> p_desc;
    data = {5, 2, 8, 1, 9, 3};
    p_asc.sort(data);  // static call - inlined, no vtable
    std::cout << "Policy (asc): ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";

    p_desc.sort(data);
    std::cout << "Policy (desc): ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";
}
// Expected output:
//   Virtual (asc): 1 2 3 5 8 9
//   Virtual (desc): 9 8 5 3 2 1
//   Policy (asc): 1 2 3 5 8 9
//   Policy (desc): 9 8 5 3 2 1
```

**Comparison:**

| Aspect | Virtual Strategy | Policy Template |
| --- | --- | --- |
| Change at runtime | Yes - `set_strategy(new_strat)` | No - Fixed at compile time |
| Performance | Virtual call per invocation | Inlined - zero overhead |
| Binary size | One class, multiple strategies | One class *per policy combination* |
| Testability | Easy mock via interface | Easy mock via policy struct |

---

### Q3: Show that policy-based design achieves zero-overhead abstraction via inlining

This example measures the performance difference. The key thing to understand is *why* the policy version is faster: the compiler can see the body of `SumPolicy::combine` at the call site, so it substitutes it directly into the loop.

**Solution - Assembly-Level Proof:**

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <chrono>

// Policy: how to combine elements
struct SumPolicy {
    static int combine(int a, int b) { return a + b; }
    static int identity() { return 0; }
};

struct ProductPolicy {
    static int combine(int a, int b) { return a * b; }
    static int identity() { return 1; }
};

// Policy-based accumulator - calls are inlined
template <typename CombinePolicy>
int policy_accumulate(const std::vector<int>& data) {
    int result = CombinePolicy::identity();
    for (int x : data)
        result = CombinePolicy::combine(result, x);  // INLINED: just `result += x`
    return result;
}

// Virtual-dispatch accumulator - calls go through vtable
class ICombiner {
public:
    virtual ~ICombiner() = default;
    virtual int combine(int a, int b) = 0;
    virtual int identity() = 0;
};

class SumCombiner : public ICombiner {
public:
    int combine(int a, int b) override { return a + b; }
    int identity() override { return 0; }
};

int virtual_accumulate(const std::vector<int>& data, ICombiner& comb) {
    int result = comb.identity();         // virtual call
    for (int x : data)
        result = comb.combine(result, x); // virtual call per element!
    return result;
}

int main() {
    std::vector<int> data(10000);
    std::iota(data.begin(), data.end(), 1);

    // Policy version: the compiler sees SumPolicy::combine = a + b
    // and inlines it -> loop becomes: result += x (identical to hand-written)
    auto t1 = std::chrono::steady_clock::now();
    volatile int sum1 = 0;
    for (int i = 0; i < 10000; ++i)
        sum1 = policy_accumulate<SumPolicy>(data);
    auto t2 = std::chrono::steady_clock::now();

    // Virtual version: each combine() is a virtual call (pointer dereference + indirect jump)
    SumCombiner comb;
    volatile int sum2 = 0;
    auto t3 = std::chrono::steady_clock::now();
    for (int i = 0; i < 10000; ++i)
        sum2 = virtual_accumulate(data, comb);
    auto t4 = std::chrono::steady_clock::now();

    auto policy_ms = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();
    auto virtual_ms = std::chrono::duration_cast<std::chrono::microseconds>(t4 - t3).count();

    std::cout << "Policy result:  " << sum1 << " (" << policy_ms << " us)\n";
    std::cout << "Virtual result: " << sum2 << " (" << virtual_ms << " us)\n";
    std::cout << "Ratio: " << static_cast<double>(virtual_ms) / policy_ms << "x\n";

    // On typical hardware with -O2:
    // Policy version: ~same as hand-written loop (compiler inlines combine)
    // Virtual version: 2-5x slower due to:
    //   - Indirect function call per iteration
    //   - Branch predictor pollution
    //   - Cannot auto-vectorize through virtual calls
}
// Expected output (approximate, platform-dependent):
//   Policy result:  50005000 (15000 us)
//   Virtual result: 50005000 (45000 us)
//   Ratio: 3x
```

The reason the virtual version can't be vectorized is important. SIMD vectorization works by processing multiple elements in parallel, but it requires the compiler to understand the loop body. A virtual call hides the body behind an indirect pointer - the compiler can't see through it and has to conservatively avoid vectorization. The policy version has no such barrier.

**Why Policy Calls Are Inlined:**

```cpp
Template instantiation:
  policy_accumulate<SumPolicy>(data)
  -> compiler generates:
    int result = 0;                    // SumPolicy::identity() inlined
    for (int x : data)
      result = result + x;            // SumPolicy::combine(a,b) inlined to a+b

    // Identical to: std::accumulate(data.begin(), data.end(), 0)
    // Zero abstraction overhead!

Virtual call:
  comb.combine(result, x)
  -> compiler generates:
    mov rax, [rcx]           // load vtable pointer
    call [rax + 8]           // indirect call through vtable
    // Cannot inline - target unknown at compile time
    // Cannot vectorize - dependency on virtual call result
```

---

## Notes

- **Alexandrescu's "Modern C++ Design"** introduced policy-based design to the mainstream C++ community.
- **`std::allocator`** is the canonical STL policy - `std::vector<T, Allocator>` is policy-based.
- **Multiple policies compose well:** `SmartPtr<T, OwnershipPolicy, CheckingPolicy, StoragePolicy>`.
- **Empty Base Optimization (EBO):** If a policy has no data members, inheriting from it (as the Logger does) costs zero bytes.
- **Concepts (C++20)** can constrain policy parameters: `template <SortPolicy P>` ensures the policy satisfies requirements at compile time.
- **Combine with CRTP** for maximum flexibility: the policy can call back into the host via `static_cast<Host&>(*this)`.
