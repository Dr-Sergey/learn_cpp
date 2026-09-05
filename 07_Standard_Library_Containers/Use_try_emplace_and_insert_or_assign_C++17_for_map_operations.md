# Use try_emplace and insert_or_assign (C++17) for map operations

**Category:** Standard Library Containers  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/container/map/try_emplace>  

---

## Topic Overview

C++17 added `try_emplace` and `insert_or_assign` to `std::map` and `std::unordered_map`. These two functions fill genuine gaps in the pre-C++17 API - gaps that caused subtle inefficiencies and real bugs with move-only types like `unique_ptr`. If you've ever done a map insertion and wondered whether it was safe, or whether you were wasting a constructor call, these are the tools that answer those questions cleanly.

### The Problem with Existing Functions

The core issue is that the older functions (`operator[]`, `insert`, `emplace`) all construct the value object at the wrong time - before they know whether the key is already in the map. That leads to wasted work, or worse:

```cpp
std::map<std::string, std::unique_ptr<Widget>> m;

// operator[]: default-constructs if key missing -> can't work with move-only types easily
m["key"] = std::make_unique<Widget>();  // works but default-constructs first if key absent

// emplace: ALWAYS constructs the value, even if key exists -> wasteful
m.emplace("key", std::make_unique<Widget>());
// If "key" exists: the unique_ptr was constructed and immediately destroyed! Resource leak potential.

// insert: same problem
m.insert({"key", std::make_unique<Widget>()});
// If "key" exists: the pair was constructed but not inserted -> value destroyed
```

The `unique_ptr` case is the most visible, but the problem exists for any type that is expensive to construct or has side effects in its constructor.

### C++17 Solutions

Here's the full comparison. The two highlighted rows are the C++17 additions and the point where the "value construction" column finally says something sensible:

| Function | Key exists | Key absent | Value construction |
| --- | --- | --- | --- |
| `operator[]` | Returns reference | Default-constructs, returns reference | Always (if absent) |
| `insert({k,v})` | No-op, returns {it, false} | Inserts | Always (pair is constructed before insert check) |
| `emplace(k, v)` | No-op, returns {it, false} | Inserts | May construct then destroy |
| **`try_emplace(k, args...)`** | **No-op, returns {it, false}** | **Constructs in-place** | **Only if key absent** |
| **`insert_or_assign(k, v)`** | **Assigns v to existing** | **Inserts** | **Always** |

### Core API

Here's both functions in action together before we dig into the details:

```cpp
#include <map>
#include <string>
#include <iostream>

int main() {
    std::map<std::string, int> m;

    // === try_emplace: insert only if key doesn't exist ===
    auto [it1, inserted1] = m.try_emplace("alice", 95);
    // inserted1 == true, m["alice"] == 95

    auto [it2, inserted2] = m.try_emplace("alice", 100);
    // inserted2 == false, m["alice"] still 95 (value 100 was never constructed as int)

    // === insert_or_assign: insert or update ===
    auto [it3, inserted3] = m.insert_or_assign("bob", 87);
    // inserted3 == true, m["bob"] == 87

    auto [it4, inserted4] = m.insert_or_assign("bob", 92);
    // inserted4 == false, m["bob"] now 92 (assigned)

    for (auto& [k, v] : m)
        std::cout << k << ": " << v << "\n";
    // alice: 95
    // bob: 92

    return 0;
}
```

---

## Self-Assessment

### Q1: Show how try_emplace avoids constructing the value when the key already exists

The instrumented `ExpensiveValue` type below lets you see exactly which constructors fire and when. Watch how `emplace` triggers a construction + destruction pair when the key is already present, while `try_emplace` fires nothing at all.

```cpp
#include <iostream>
#include <map>
#include <string>
#include <memory>

struct ExpensiveValue {
    std::string data;

    ExpensiveValue(std::string d) : data(std::move(d)) {
        std::cout << "  [CONSTRUCTED] ExpensiveValue(\"" << data << "\")\n";
    }
    ExpensiveValue(const ExpensiveValue& other) : data(other.data) {
        std::cout << "  [COPY] ExpensiveValue(\"" << data << "\")\n";
    }
    ExpensiveValue(ExpensiveValue&& other) noexcept : data(std::move(other.data)) {
        std::cout << "  [MOVE] ExpensiveValue\n";
    }
    ~ExpensiveValue() {
        std::cout << "  [DESTROYED] ExpensiveValue(\"" << data << "\")\n";
    }
};

int main() {
    std::map<std::string, ExpensiveValue> cache;

    // === emplace: ALWAYS constructs the value ===
    std::cout << "1) emplace(\"key\", \"first\"):\n";
    cache.emplace("key", "first");
    // Output: [CONSTRUCTED] ExpensiveValue("first")

    std::cout << "\n2) emplace(\"key\", \"second\") - key exists:\n";
    cache.emplace("key", "second");
    // Output: [CONSTRUCTED] ExpensiveValue("second")
    //         [DESTROYED] ExpensiveValue("second")
    // The value was constructed, then discarded! Wasteful.

    cache.clear();
    std::cout << "\n--- cleared ---\n\n";

    // === try_emplace: only constructs if key absent ===
    std::cout << "3) try_emplace(\"key\", \"first\"):\n";
    auto [it1, ok1] = cache.try_emplace("key", "first");
    std::cout << "   inserted: " << ok1 << "\n";
    // Output: [CONSTRUCTED] ExpensiveValue("first")
    //         inserted: 1

    std::cout << "\n4) try_emplace(\"key\", \"second\") - key exists:\n";
    auto [it2, ok2] = cache.try_emplace("key", "second");
    std::cout << "   inserted: " << ok2 << "\n";
    // Output: inserted: 0
    // NO construction! "second" was never used.

    std::cout << "\n5) Value is still: " << it2->second.data << "\n";
    // Output: "first"

    // === Critical for unique_ptr ===
    std::cout << "\n--- unique_ptr example ---\n";
    std::map<int, std::unique_ptr<int>> ptrs;

    // try_emplace: safe with move-only types
    ptrs.try_emplace(1, std::make_unique<int>(42));
    ptrs.try_emplace(1, std::make_unique<int>(99));  // NOT constructed if key exists
    std::cout << "ptr[1] = " << *ptrs[1] << "\n";  // 42

    // emplace with unique_ptr: if key exists, the unique_ptr is destroyed -> resource leak pattern
    // (not a real leak since unique_ptr cleans up, but the allocation was wasted)

    return 0;
}
```

The `unique_ptr` case at the end drives home why this matters beyond just performance: with `emplace`, if the key exists, you've woken up a resource (allocated memory, opened a file, started a connection) only to immediately destroy it. `try_emplace` simply doesn't touch the arguments when it doesn't need to.

- `emplace(key, args...)` constructs a `pair<const Key, Value>` first, *then* checks if the key exists. If it does, the pair is destroyed - the value was constructed for nothing.
- `try_emplace(key, args...)` checks the key **first**. If the key exists, it returns `{iterator, false}` immediately and never touches the constructor arguments.
- This is especially important for expensive types (database connections, file handles) and move-only types (`unique_ptr`, `thread`).

### Q2: Compare operator[], insert, emplace, try_emplace, and insert_or_assign in terms of construction semantics

It's useful to have a single experiment that shows all five operations side by side so you can build the mental model for when to reach for each one.

| Operation | Key absent | Key present | Value construction | Return type |
| --- | --- | --- | --- | --- |
| **`m[key]`** | Default-constructs value, returns ref | Returns ref to existing | Default ctor if absent | `V&` |
| **`m[key] = val`** | Default-constructs + assigns | Assigns to existing | Default ctor + assign | `V&` |
| **`insert({k,v})`** | Inserts pair | No-op | **Always** (pair constructed before check) | `pair<it, bool>` |
| **`emplace(k,v)`** | Inserts | No-op | **May construct then destroy** | `pair<it, bool>` |
| **`try_emplace(k, args)`** | Constructs in-place | **No-op, no construction** | **Only if absent** | `pair<it, bool>` |
| **`insert_or_assign(k,v)`** | Inserts | **Assigns** new value | Always | `pair<it, bool>` |

```cpp
#include <iostream>
#include <map>
#include <string>

struct Verbose {
    int val;
    Verbose() : val(0)               { std::cout << "  default-ctor\n"; }
    Verbose(int v) : val(v)          { std::cout << "  ctor(" << v << ")\n"; }
    Verbose(const Verbose& o) : val(o.val) { std::cout << "  copy(" << val << ")\n"; }
    Verbose(Verbose&& o) : val(o.val) { std::cout << "  move(" << val << ")\n"; }
    Verbose& operator=(const Verbose& o) { val = o.val; std::cout << "  assign(" << val << ")\n"; return *this; }
    Verbose& operator=(Verbose&& o)     { val = o.val; std::cout << "  move-assign(" << val << ")\n"; return *this; }
};

int main() {
    // === operator[] ===
    std::cout << "operator[] (absent):\n";
    { std::map<int,Verbose> m; m[1]; }       // default-ctor
    std::cout << "operator[] = val (absent):\n";
    { std::map<int,Verbose> m; m[1] = Verbose(5); }  // default-ctor + move-assign

    // === insert ===
    std::cout << "\ninsert (absent):\n";
    { std::map<int,Verbose> m; m.insert({1, Verbose(10)}); }  // ctor + move(s)
    std::cout << "insert (present):\n";
    { std::map<int,Verbose> m; m.try_emplace(1, 10);
      m.insert({1, Verbose(20)}); }                            // ctor + move + destroy

    // === try_emplace ===
    std::cout << "\ntry_emplace (absent):\n";
    { std::map<int,Verbose> m; m.try_emplace(1, 10); }        // ctor(10) only
    std::cout << "try_emplace (present):\n";
    { std::map<int,Verbose> m; m.try_emplace(1, 10);
      m.try_emplace(1, 20); }                                  // NOTHING for 20

    // === insert_or_assign ===
    std::cout << "\ninsert_or_assign (absent):\n";
    { std::map<int,Verbose> m; m.insert_or_assign(1, Verbose(10)); }  // ctor + move
    std::cout << "insert_or_assign (present):\n";
    { std::map<int,Verbose> m; m.try_emplace(1, 10);
      m.insert_or_assign(1, Verbose(30)); }                           // ctor + move-assign

    return 0;
}
```

The `try_emplace (present)` case is the one to watch: the second call produces no output at all. The `20` argument never becomes a `Verbose` object. Compare that to `insert (present)`, which constructs and then throws away the value.

- **`try_emplace`** is the most efficient for "insert if absent": zero wasted constructions.
- **`insert_or_assign`** is the cleanest "upsert" (insert or update): it replaces `m[key] = val` without the default-construction overhead when the key is absent.
- **`operator[]`** default-constructs when the key is absent, making it inappropriate for types without a default constructor or when default construction is expensive.

### Q3: Write a cache-or-compute pattern using try_emplace to construct only on cache miss

This is the pattern you'll reach for most often in production code. There's a subtlety though - `try_emplace(key, compute(key))` is *not* truly lazy, because C++ evaluates all function arguments before the call. The example shows both the naive version and the correct lazy fix.

```cpp
#include <iostream>
#include <map>
#include <string>
#include <chrono>
#include <thread>
#include <functional>

// === Expensive computation ===
std::string compute_result(int key) {
    std::cout << "  [COMPUTING] result for key " << key << "...\n";
    // Simulate expensive work
    return "result_" + std::to_string(key * key);
}

// === Cache-or-compute with try_emplace ===
class ComputeCache {
    std::map<int, std::string> cache_;

public:
    const std::string& get(int key) {
        // try_emplace: if key absent, constructs value from compute_result
        // if key present, does NOTHING - no computation, no construction
        auto [it, inserted] = cache_.try_emplace(key, compute_result(key));
        // Note: compute_result is ALWAYS called because it's evaluated before try_emplace decides.
        // This is actually NOT ideal - see the improved version below.
        return it->second;
    }

    // === IMPROVED: truly lazy computation ===
    const std::string& get_lazy(int key) {
        auto it = cache_.find(key);
        if (it != cache_.end())
            return it->second;

        // Only compute on cache miss
        auto [new_it, _] = cache_.try_emplace(key, compute_result(key));
        return new_it->second;
    }

    // === ALTERNATIVE: use try_emplace with piecewise construct ===
    // For types that can be constructed from args directly:
    // auto [it, ok] = cache_.try_emplace(key);  // default-constructs string
    // if (ok) it->second = compute_result(key);  // assign only on miss

    void clear() { cache_.clear(); }
    size_t size() const { return cache_.size(); }
};

int main() {
    ComputeCache cache;

    std::cout << "=== First access (cache miss) ===\n";
    std::cout << "get(5) = " << cache.get_lazy(5) << "\n";
    // Output:
    // [COMPUTING] result for key 5...
    // get(5) = result_25

    std::cout << "\n=== Second access (cache hit) ===\n";
    std::cout << "get(5) = " << cache.get_lazy(5) << "\n";
    // Output:
    // get(5) = result_25
    // (no computation!)

    std::cout << "\n=== New key (cache miss) ===\n";
    std::cout << "get(3) = " << cache.get_lazy(3) << "\n";
    // Output:
    // [COMPUTING] result for key 3...
    // get(3) = result_9

    std::cout << "\nCache size: " << cache.size() << "\n";  // 2

    // === Real-world pattern: memoized Fibonacci ===
    std::map<int, long long> fib_cache;
    std::function<long long(int)> fib = [&](int n) -> long long {
        if (n <= 1) return n;

        // Check cache first
        auto it = fib_cache.find(n);
        if (it != fib_cache.end()) return it->second;

        // Compute and cache
        long long result = fib(n - 1) + fib(n - 2);
        fib_cache.try_emplace(n, result);
        return result;
    };

    std::cout << "\nfib(40) = " << fib(40) << "\n";   // 102334155 (instant with memoization)
    std::cout << "fib cache entries: " << fib_cache.size() << "\n";

    return 0;
}
```

The `get_lazy` function shows the idiomatic pattern: `find` first, and only fall through to `try_emplace` when you know the key is absent. At that point there's no ambiguity - you're inserting, not checking - so `try_emplace` is the right call. The memoized Fibonacci at the end shows the same pattern applied recursively.

- **`try_emplace` for caching:** When the key exists, `try_emplace` returns the existing iterator without constructing anything. When absent, it constructs in-place.
- **Lazy computation caveat:** `try_emplace(key, compute(key))` still calls `compute()` because C++ evaluates all arguments before calling the function. For truly lazy behavior, do a `find()` first and only call `try_emplace` on miss.
- **Return value:** `try_emplace` returns `{iterator, bool}`. The iterator always points to the element (existing or newly inserted). The bool tells you whether insertion happened.
- **Memoization:** The `find` + `try_emplace` combination is a clean pattern for memoized recursive functions - it avoids recomputation and avoids constructing values you don't need.

---

## Notes

- **`try_emplace` key forwarding:** The key is forwarded separately from the value args: `try_emplace(key, arg1, arg2, ...)` forwards the args directly to the value's constructor. This is different from `emplace(key, value)` which constructs a pair first.
- **`insert_or_assign` vs `operator[]`:** Both update existing values, but `insert_or_assign` tells you whether it inserted or assigned (via the bool), and doesn't require the value type to be default-constructible.
- **Hint versions:** Both functions have hint overloads - `try_emplace(hint, key, args...)` and `insert_or_assign(hint, key, val)` - for O(1) amortized insertion when you know approximately where the element belongs.
- **Works with `unordered_map` too:** Both functions are available on `std::unordered_map` with identical semantics.
- **Thread safety:** Neither function is thread-safe. Use a mutex or a concurrent container for shared-state caches.
