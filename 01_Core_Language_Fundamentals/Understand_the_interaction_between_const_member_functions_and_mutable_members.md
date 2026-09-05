# Understand the interaction between const member functions and mutable members

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/cv>  

---

## Topic Overview

A `const` member function promises not to modify the object's observable state. The `mutable` keyword exempts specific members from this restriction - they can be modified even through `const` access. The primary use cases are **thread synchronization** (mutexes) and **caching**.

### const Member Functions

Inside a `const` method, `this` is treated as a pointer to a const object. Any attempt to assign a data member is a compile error.

```cpp
class Widget {
    int value_ = 0;
public:
    int get() const {
        // 'this' is const Widget* — cannot modify members
        // value_ = 42;  // ERROR: cannot modify in const method
        return value_;
    }
    void set(int v) {
        value_ = v;  // OK: non-const method
    }
};
```

### The mutable Keyword

Sometimes a member logically doesn't affect the object's state - a cache or a mutex, for instance. Marking it `mutable` lets you change it even inside `const` methods.

```cpp
class CachedWidget {
    int value_ = 0;
    mutable int cached_result_ = -1;   // Can be modified in const methods
    mutable bool cache_valid_ = false;  // Can be modified in const methods
public:
    int compute() const {
        if (!cache_valid_) {
            cached_result_ = value_ * value_;  // OK: mutable member
            cache_valid_ = true;               // OK: mutable member
        }
        return cached_result_;
    }
};
```

From the caller's perspective, `compute()` is a pure query - it always returns the same value for the same `value_`. The cache is an implementation detail that doesn't count as observable state.

### mutable Mutex for Thread-Safe const Access

The mutex case is the most important in modern C++. Locking a mutex modifies it internally, but that's not part of your class's logical state - so `mutable` is the right tool.

```cpp
class ThreadSafeCounter {
    int count_ = 0;
    mutable std::mutex mtx_;  // Must be mutable for const methods
public:
    int get() const {
        std::lock_guard lock(mtx_);  // Lock in const method — OK because mutable
        return count_;
    }
    void increment() {
        std::lock_guard lock(mtx_);
        ++count_;
    }
};
```

Without `mutable`, locking the mutex in `get()` would be a compile error because `lock()` is a non-const operation.

---

## Self-Assessment

### Q1: Use `mutable` on a mutex member to allow locking in a const method without UB

A `Config` object is often accessed by many reader threads simultaneously. Making the mutex `mutable` lets the `const` getters lock safely without changing the design.

```cpp
#include <iostream>
#include <mutex>
#include <string>
#include <thread>
#include <vector>

class Config {
    std::string host_ = "localhost";
    int port_ = 8080;
    mutable std::mutex mtx_;  // mutable: can lock in const methods
public:
    // const method needs to lock the mutex for thread safety
    std::string get_host() const {
        std::lock_guard<std::mutex> lock(mtx_);  // OK: mtx_ is mutable
        return host_;
    }

    int get_port() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return port_;
    }

    void set_host(const std::string& h) {
        std::lock_guard<std::mutex> lock(mtx_);
        host_ = h;
    }

    void set_port(int p) {
        std::lock_guard<std::mutex> lock(mtx_);
        port_ = p;
    }
};

int main() {
    Config config;

    // Multiple threads reading (const access) — thread-safe via mutable mutex
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([&config]() {
            for (int j = 0; j < 100; ++j) {
                auto host = config.get_host();  // calls const method
                auto port = config.get_port();  // calls const method
            }
        });
    }

    // Writer thread
    threads.emplace_back([&config]() {
        config.set_host("example.com");
        config.set_port(443);
    });

    for (auto& t : threads) t.join();

    std::cout << config.get_host() << ":" << config.get_port() << "\n";
}
```

All four reader threads call `const` methods; they can do so on the same object simultaneously because the mutex serializes the actual reads safely.

**How this works:**

- Without `mutable`, `std::mutex` couldn't be locked in a `const` method - `lock()` modifies the mutex.
- The mutex is NOT part of the object's logical state - it's a synchronization mechanism.
- `mutable` tells the compiler: "this member can change even in const contexts."
- This is the most important use of `mutable` in modern C++.

### Q2: Show a lazy-initialization cache using mutable that is logically const but physically modifiable

The trick here is that repeated calls to `result()` always return the same number - the computation happens only once, the first time.

```cpp
#include <iostream>
#include <string>
#include <optional>
#include <cmath>

class ExpensiveCalculator {
    double input_;
    mutable std::optional<double> cached_result_;  // Lazy cache
    mutable int compute_count_ = 0;                // Track recomputations
public:
    explicit ExpensiveCalculator(double x) : input_(x) {}

    // Logically const: doesn't change observable state
    // Physically modifies: updates cache on first call
    double result() const {
        if (!cached_result_.has_value()) {
            // Expensive computation — only done once
            ++compute_count_;
            double r = 0;
            for (int i = 0; i < 1000; ++i) {
                r += std::sin(input_ * i) * std::cos(input_ * i);
            }
            cached_result_ = r;
            std::cout << "  (computed from scratch)\n";
        }
        return *cached_result_;
    }

    int computations() const { return compute_count_; }

    // Non-const: changing input invalidates cache
    void set_input(double x) {
        input_ = x;
        cached_result_.reset();  // Invalidate cache
    }
};

int main() {
    const ExpensiveCalculator calc(1.5);

    // First call: computes
    std::cout << "Result: " << calc.result() << "\n";

    // Second call: uses cache
    std::cout << "Result: " << calc.result() << "\n";

    // Third call: still uses cache
    std::cout << "Result: " << calc.result() << "\n";

    std::cout << "Total computations: " << calc.computations() << "\n";  // 1
}
```

Notice that even though `calc` is declared `const`, the cache fills in on the first call. The `mutable` keyword makes this physically possible while keeping the type logically const.

**How this works:**

- `cached_result_` and `compute_count_` are `mutable` - they can change in `const` methods.
- The object's logical state (the input and its mathematical result) is unchanged by caching.
- Only the physical state (cache presence) changes - this is "logically const."
- Pattern is common in: database query caches, hash code memoization, lazy initialization.

### Q3: Explain the thread-safety implications of mutable members in const methods

**Answer:**

The C++ standard says:
> "const member functions shall be safe for simultaneous execution on the same object" (§[res.on.data.races])

This means `const` methods should be thread-safe. But `mutable` members can be modified in `const` methods, creating a hidden mutation point.

**The problem:** If multiple threads call `const` methods simultaneously, and those methods modify `mutable` members without synchronization, you have a **data race** (undefined behavior).

The two classes below contrast the unsafe and safe approaches - the only structural difference is the added mutex.

```cpp
#include <iostream>
#include <mutex>
#include <optional>
#include <thread>

// UNSAFE: mutable cache without synchronization
class UnsafeCache {
    int data_ = 42;
    mutable std::optional<int> cache_;  // mutable but unprotected!
public:
    int get() const {
        if (!cache_) {
            cache_ = data_ * 2;  // DATA RACE if called from multiple threads!
        }
        return *cache_;
    }
};

// SAFE: mutable cache with mutable mutex
class SafeCache {
    int data_ = 42;
    mutable std::optional<int> cache_;
    mutable std::mutex mtx_;  // Protects the mutable cache
public:
    int get() const {
        std::lock_guard lock(mtx_);
        if (!cache_) {
            cache_ = data_ * 2;  // Protected by mutex
        }
        return *cache_;
    }
};

int main() {
    SafeCache sc;

    std::thread t1([&]{ for (int i=0; i<1000; ++i) sc.get(); });
    std::thread t2([&]{ for (int i=0; i<1000; ++i) sc.get(); });
    t1.join(); t2.join();

    std::cout << "Result: " << sc.get() << "\n";  // Always 84
}
```

Every `mutable` member that a `const` method can modify is a potential data race. Adding the mutex - itself `mutable` - closes that gap.

**Rules:**

1. If a `const` method modifies `mutable` state, it MUST synchronize (mutex, atomic, etc.).
2. `std::mutex` itself should be `mutable` so it can be locked in `const` methods.
3. Consider `std::atomic` for simple mutable counters/flags instead of a full mutex.
4. The standard library guarantees `const` member functions are thread-safe - your classes should too.

---

## Notes

- `mutable` should be rare - it's appropriate for mutexes, caches, and instrumentation counters.
- Overusing `mutable` defeats the purpose of `const` - if everything is mutable, `const` is meaningless.
- `mutable` lambdas (`[x]() mutable { x++; }`) are a separate concept - they allow modifying captured-by-value variables.
- Herb Sutter's guideline: "`const` means thread-safe" - if your `const` methods aren't thread-safe with respect to `mutable` state, you have a bug.
- `std::atomic<T>` members are naturally compatible with `const` access - they handle synchronization internally.
