# Write a memoization wrapper using templates and std::unordered_map

**Category:** Lambda and Functional  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/container/unordered_map>  

---

## Topic Overview

**Memoization** caches the results of expensive function calls so that repeated calls with the same arguments return instantly from the cache. Rather than writing caching logic by hand inside every function you want to speed up, you can build a generic `memoize(f)` wrapper that works for any function whose arguments can be used as a map key.

The mental model is a simple call tree. Imagine calling `memoized_fib(5)` for the first time:

```cpp
memoized_fib(5)
  |
  +- cache miss -> compute fib(5), store result
  +- fib(5) calls memoized_fib(4)
  |    +- cache miss -> compute, store
  |    +- ...
  +- fib(5) calls memoized_fib(3)
  |    +- CACHE HIT! -> return immediately
  +- return result
```

Each unique set of arguments is computed once and stored. Every subsequent call with the same arguments short-circuits directly to the cached value.

---

## Self-Assessment

### Q1: Implement `memoize(f)` that caches results keyed on arguments

The tricky technical piece here is the key type. `std::unordered_map` needs a hash for its key type, and `std::tuple` doesn't come with one in the standard library. The `TupleHash` struct below solves that by combining the individual element hashes using a seed-mixing technique borrowed from Boost.

```cpp
#include <iostream>
#include <unordered_map>
#include <functional>
#include <tuple>

// Hash for std::tuple (needed as unordered_map key)
struct TupleHash {
    template <typename... Ts>
    size_t operator()(const std::tuple<Ts...>& t) const {
        return std::apply([](const auto&... args) {
            size_t seed = 0;
            ((seed ^= std::hash<std::decay_t<decltype(args)>>{}(args)
                       + 0x9e3779b9 + (seed << 6) + (seed >> 2)), ...);
            return seed;
        }, t);
    }
};

// Generic memoize wrapper
template <typename F>
auto memoize(F f) {
    return [f = std::move(f)](auto... args) mutable {
        using Key = std::tuple<decltype(args)...>;
        using Result = decltype(f(args...));
        static std::unordered_map<Key, Result, TupleHash> cache;

        Key key{args...};
        if (auto it = cache.find(key); it != cache.end()) {
            return it->second;
        }
        auto result = f(args...);
        cache[key] = result;
        return result;
    };
}

int main() {
    // Memoize an expensive function
    int call_count = 0;
    auto expensive = memoize([&call_count](int x, int y) {
        ++call_count;
        std::cout << "  Computing " << x << " + " << y << "\n";
        return x + y;
    });

    std::cout << "First call (3,4): " << expensive(3, 4) << "\n";
    std::cout << "Second call (3,4): " << expensive(3, 4) << "\n";
    std::cout << "Third call (5,6): " << expensive(5, 6) << "\n";
    std::cout << "Fourth call (3,4): " << expensive(3, 4) << "\n";
    std::cout << "Total computations: " << call_count << "\n";
}
// Expected output:
//     Computing 3 + 4
//   First call (3,4): 7
//   Second call (3,4): 7
//     Computing 5 + 6
//   Third call (5,6): 11
//   Fourth call (3,4): 7
//   Total computations: 2
```

Only two actual computations happen for four calls - the second and fourth calls to `(3,4)` both hit the cache.

---

### Q2: Show that the memoized version returns cached results without recomputing

This example makes the performance difference visible by simulating an expensive computation and timing each call. The cached calls come back essentially instantly while the first call for any argument takes real time.

```cpp
#include <iostream>
#include <unordered_map>
#include <functional>
#include <chrono>
#include <thread>

// Simple memoize for single-argument functions
template <typename F>
auto memoize_single(F f) {
    using Arg = int;
    using Result = decltype(f(Arg{}));
    std::unordered_map<Arg, Result> cache;

    return [f = std::move(f), cache = std::move(cache)](Arg x) mutable -> Result {
        if (auto it = cache.find(x); it != cache.end()) {
            std::cout << "  [CACHE HIT] ";
            return it->second;
        }
        std::cout << "  [COMPUTING] ";
        auto result = f(x);
        cache[x] = result;
        return result;
    };
}

int main() {
    // Simulate expensive computation
    auto slow_square = memoize_single([](int x) -> long long {
        // Simulated expensive work
        volatile long long result = 0;
        for (int i = 0; i < 1000000; ++i)
            result += x;
        return result / 1000000 * x;
    });

    auto time_it = [](auto& fn, int x) {
        auto start = std::chrono::steady_clock::now();
        auto result = fn(x);
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "f(" << x << ") = " << result << "  (" << us << " us)\n";
    };

    std::cout << "Call 1:"; time_it(slow_square, 42);
    std::cout << "Call 2:"; time_it(slow_square, 42);  // instant!
    std::cout << "Call 3:"; time_it(slow_square, 7);
    std::cout << "Call 4:"; time_it(slow_square, 7);   // instant!
    std::cout << "Call 5:"; time_it(slow_square, 42);  // still cached!
}
// Expected output (approximate):
//   Call 1:  [COMPUTING] f(42) = 1764  (1234 us)
//   Call 2:  [CACHE HIT] f(42) = 1764  (1 us)     <- ~1000x faster!
//   Call 3:  [COMPUTING] f(7) = 49  (987 us)
//   Call 4:  [CACHE HIT] f(7) = 49  (0 us)
//   Call 5:  [CACHE HIT] f(42) = 1764  (0 us)
```

---

### Q3: Explain thread-safety concerns with a shared cache and how to add mutex protection

The simple `memoize` above uses a `static` cache inside a lambda, which means all callers share the same map. That's fine in single-threaded code, but in a multi-threaded program two threads can race on reads and writes to the same map, which is undefined behavior. The solution is to guard the cache with a lock.

The reason this warrants more than a simple `std::mutex` is that reads are far more common than writes - once a key is computed it stays in the cache forever. A `std::shared_mutex` lets multiple threads read concurrently and only forces exclusive access when a new result needs to be written.

```cpp
#include <iostream>
#include <unordered_map>
#include <functional>
#include <mutex>
#include <shared_mutex>
#include <thread>
#include <vector>

// Thread-safe memoization with reader-writer lock
template <typename F>
auto memoize_threadsafe(F f) {
    return [f = std::move(f)](auto... args) mutable {
        using Key = std::tuple<decltype(args)...>;
        using Result = decltype(f(args...));

        struct Cache {
            std::unordered_map<int, Result> map;  // simplified to int key
            mutable std::shared_mutex mtx;
        };
        static Cache cache;

        int key = std::get<0>(std::tuple{args...});  // simplified

        // Fast path: shared (read) lock for cache hits
        {
            std::shared_lock lock(cache.mtx);
            if (auto it = cache.map.find(key); it != cache.map.end()) {
                return it->second;
            }
        }

        // Slow path: exclusive (write) lock for cache misses
        std::unique_lock lock(cache.mtx);
        // Double-check after acquiring write lock!
        if (auto it = cache.map.find(key); it != cache.map.end()) {
            return it->second;
        }
        auto result = f(args...);
        cache.map[key] = result;
        return result;
    };
}

int main() {
    auto memo_square = memoize_threadsafe([](int x) {
        return x * x;
    });

    // Multiple threads using the same memoized function
    std::vector<std::thread> threads;
    for (int i = 0; i < 8; ++i) {
        threads.emplace_back([&memo_square, i]() {
            for (int j = 0; j < 100; ++j) {
                auto result = memo_square(j % 10);
                (void)result;  // suppress unused warning
            }
        });
    }

    for (auto& t : threads) t.join();
    std::cout << "All threads completed safely\n";
}
// Expected output:
//   All threads completed safely
```

The double-check after acquiring the write lock is important. Between releasing the shared lock and acquiring the exclusive lock, another thread might have already computed and stored the same key. Without the double-check, you'd compute it twice and overwrite the result unnecessarily (harmless here, but wasteful - and for side-effecting functions it could be a real bug).

**Thread-safety concerns:**

| Problem | Consequence | Solution |
| --- | --- | --- |
| Concurrent read+write to map | Undefined behavior | Mutex protection |
| Two threads compute same key | Wasted work (but correct) | Double-check after lock |
| Lock contention on writes | Performance bottleneck | `shared_mutex` (read/write lock) |
| Cache growing unbounded | Memory leak | TTL, LRU eviction, or `weak_ptr` |

**Lock strategies:**

1. **`std::mutex`** - simple, exclusive lock for all access
2. **`std::shared_mutex`** - multiple readers OR one writer (better for read-heavy caches)
3. **Lock-free** - `concurrent_hash_map` (TBB) or atomic operations
4. **Thread-local caches** - no locking needed, but duplicates work across threads

---

## Notes

- **Recursive memoization** requires the cache lookup to go through the same wrapper. Pass the memoized function as an argument or use `std::function` for self-reference.
- **Key hashing:** `std::tuple` has no default hash in the standard library - you must provide one (like the `TupleHash` above).
- **Cache invalidation:** "There are only two hard things in computer science: cache invalidation and naming things." Consider TTL or manual invalidation strategies for data that can go stale.
- **Memory:** Unbounded caches can consume unlimited memory. Consider `std::map` with LRU eviction for bounded caches.
- **`constexpr` functions** don't need runtime memoization - they're computed at compile time.
