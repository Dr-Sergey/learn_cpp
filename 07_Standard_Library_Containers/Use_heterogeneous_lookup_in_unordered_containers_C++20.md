# Use heterogeneous lookup in unordered containers (C++20)

**Category:** Standard Library Containers  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/container/unordered_map/find>  

---

## Topic Overview

**Heterogeneous lookup** lets you search an `unordered_map<std::string, V>` using a `std::string_view` or `const char*` **without constructing a temporary `std::string`**. Before C++20, every `find()`, `count()`, or `contains()` call with a non-key type forced an implicit conversion to the key type - meaning a heap allocation for each lookup.

The reason this matters in practice: if you have a hot path doing millions of map lookups using string views or raw string literals, you're paying for millions of unnecessary heap allocations. Each one is small, but they add up, and they also fragment your allocator's free list. Heterogeneous lookup eliminates every single one of those allocations.

### The Problem (pre-C++20)

```cpp
std::unordered_map<std::string, int> m = {{"hello", 1}, {"world", 2}};

std::string_view sv = "hello";

// Before C++20 heterogeneous lookup:
m.find(sv);   // Implicitly constructs std::string("hello") - HEAP ALLOCATION!
m.count(sv);  // Another temporary string
m.contains(sv); // Another temporary string
// For hot loops doing millions of lookups, this is catastrophic.
```

### The Solution (C++20)

To enable heterogeneous lookup in unordered containers, you need two things:

1. A **transparent hash** - a hash functor with a nested `is_transparent` type alias that can hash different types.
2. A **transparent equality** - an equality functor with `is_transparent` that can compare different types.

The `is_transparent` tag is the signal to the container: "I can handle types other than the key type, so please let `find()` accept them directly without constructing a `Key`."

```cpp
Without heterogeneous lookup:        With heterogeneous lookup:

  find("hello")                        find("hello")
       |                                    |
       v                                    v
  string("hello")  <- allocation!       hash(const char*)  <- no allocation
       |                                    |
       v                                    v
  hash(string)                          bucket lookup
       |                                    |
       v                                    v
  bucket lookup                         equal(string, const char*)
       |                                    |
       v                                    v
  equal(string, string)                 found!
       |
       v
  found!
```

### Enabling Heterogeneous Lookup

The trick for `std::string` keys is that both `std::string` and `const char*` implicitly convert to `std::string_view`, so a single `operator()` overload accepting `string_view` handles all three types:

```cpp
#include <iostream>
#include <unordered_map>
#include <string>
#include <string_view>

// Transparent hash: can hash string, string_view, const char*
struct StringHash {
    using is_transparent = void;  // <- Required tag type

    size_t operator()(std::string_view sv) const {
        return std::hash<std::string_view>{}(sv);
    }
    // string and const char* implicitly convert to string_view
};

// Transparent equality: can compare across types
struct StringEqual {
    using is_transparent = void;  // <- Required tag type

    bool operator()(std::string_view a, std::string_view b) const {
        return a == b;
    }
};

int main() {
    // Use custom hash + equal as template parameters
    std::unordered_map<std::string, int, StringHash, StringEqual> m;
    m["hello"] = 1;
    m["world"] = 2;

    // Now these work WITHOUT creating temporary std::string:
    std::string_view sv = "hello";
    auto it = m.find(sv);           // No allocation!
    bool has = m.contains("world"); // No allocation!
    size_t n = m.count(sv);         // No allocation!

    if (it != m.end())
        std::cout << it->first << " = " << it->second << "\n";
    // Output: hello = 1

    return 0;
}
```

Notice how clean the call sites look - they're identical to normal `find()` calls. All the machinery is in the type parameters of the map declaration.

### Which Operations Support Heterogeneous Lookup (C++20)

| Operation | Supports heterogeneous? |
| --- | --- |
| `find(K)` | Yes |
| `count(K)` | Yes |
| `contains(K)` | Yes |
| `equal_range(K)` | Yes |
| `operator[](K)` | **No** - always requires key type |
| `at(K)` | **No** |
| `erase(K)` | **C++23** only |
| `insert_or_assign(K,V)` | **No** |

The read-only operations all support it; the mutating operations mostly don't. This makes sense: you're looking things up with a lightweight view, not inserting or erasing with one.

---

## Self-Assessment

### Q1: Enable heterogeneous lookup in std::unordered_map with a transparent hash and equal

This walks through the three-step setup and then verifies the critical invariant: the hash must produce the same value for `string`, `string_view`, and `const char*` representations of the same text:

```cpp
#include <iostream>
#include <unordered_map>
#include <string>
#include <string_view>

// Step 1: Transparent hash
struct TransparentHash {
    using is_transparent = void;

    // Hash anything convertible to string_view
    size_t operator()(std::string_view sv) const noexcept {
        return std::hash<std::string_view>{}(sv);
    }

    // Explicit overloads for clarity (optional - all convert to string_view):
    size_t operator()(const std::string& s) const noexcept {
        return std::hash<std::string_view>{}(s);
    }
    size_t operator()(const char* s) const noexcept {
        return std::hash<std::string_view>{}(s);
    }
};

// Step 2: Transparent equality
struct TransparentEqual {
    using is_transparent = void;

    bool operator()(std::string_view a, std::string_view b) const noexcept {
        return a == b;
    }
};

int main() {
    // Step 3: Declare map with both transparent hash + equal
    using TransMap = std::unordered_map<
        std::string,        // Key type
        int,                // Value type
        TransparentHash,    // Hash (with is_transparent)
        TransparentEqual    // Equal (with is_transparent)
    >;

    TransMap m;
    m.emplace("alpha", 1);
    m.emplace("beta", 2);
    m.emplace("gamma", 3);

    // === Lookup with string_view - no allocation ===
    std::string_view sv = "beta";
    if (auto it = m.find(sv); it != m.end()) {
        std::cout << "Found '" << it->first << "' = " << it->second << "\n";
    }
    // Output: Found 'beta' = 2

    // === Lookup with const char* - no allocation ===
    if (m.contains("gamma")) {
        std::cout << "Contains 'gamma'\n";
    }
    // Output: Contains 'gamma'

    // === Lookup with std::string - also no extra allocation ===
    std::string key = "alpha";
    std::cout << "count('alpha') = " << m.count(key) << "\n";
    // Output: count('alpha') = 1

    // === Without transparent hash, this would allocate ===
    // std::unordered_map<std::string, int> normal_m;
    // normal_m.find(sv);  // Constructs temporary string from sv!

    // === Proof: hash consistency check ===
    TransparentHash h;
    std::string s = "test";
    std::string_view sv2 = "test";
    const char* cs = "test";
    std::cout << "\nHash consistency:\n";
    std::cout << "  hash(string):      " << h(s) << "\n";
    std::cout << "  hash(string_view): " << h(sv2) << "\n";
    std::cout << "  hash(const char*): " << h(cs) << "\n";
    // All three must produce the same hash value!

    return 0;
}
```

The hash consistency check at the end is not just a demo - it is a correctness requirement. If `hash(a) != hash(b)` for two values that `equal(a, b)` considers equal, the container will silently fail to find elements. Always verify this when writing a custom transparent hash.

**How it works:**

- The `is_transparent` tag type in both hash and equal tells the container "I can handle types other than the key type."
- Without this tag, `find()` only accepts `const Key&` and silently constructs a temporary.
- The hash functor accepts `std::string_view`, and since both `std::string` and `const char*` implicitly convert to `string_view`, a single overload handles all three.
- **Critical invariant:** For any two values `a` and `b`, if `equal(a, b)` is true, then `hash(a)` must equal `hash(b)`. Using `std::hash<std::string_view>` for all types ensures this.

### Q2: Look up a std::string_view key in an unordered_map<std::string, int> without constructing a string

This benchmark uses long keys (to make the allocation cost visible) and many repetitions so the difference in allocation overhead dominates the timing:

```cpp
#include <iostream>
#include <unordered_map>
#include <string>
#include <string_view>
#include <chrono>

struct StringHash {
    using is_transparent = void;
    size_t operator()(std::string_view sv) const noexcept {
        return std::hash<std::string_view>{}(sv);
    }
};

struct StringEqual {
    using is_transparent = void;
    bool operator()(std::string_view a, std::string_view b) const noexcept {
        return a == b;
    }
};

int main() {
    constexpr int N = 1'000'000;

    // Build a map with long string keys (heap-allocated)
    std::unordered_map<std::string, int, StringHash, StringEqual> hetero_map;
    std::unordered_map<std::string, int> normal_map;

    for (int i = 0; i < 1000; ++i) {
        std::string key = "key_number_" + std::to_string(i) + "_with_some_extra_padding";
        hetero_map[key] = i;
        normal_map[key] = i;
    }

    std::string_view lookup_key = "key_number_500_with_some_extra_padding";

    // === Normal map: find(string_view) -> allocates temporary string ===
    {
        auto start = std::chrono::steady_clock::now();
        volatile int sum = 0;
        for (int i = 0; i < N; ++i) {
            auto it = normal_map.find(std::string(lookup_key));  // ALLOCATION each time!
            if (it != normal_map.end()) sum += it->second;
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "Normal map (allocating):      " << ms << " ms\n";
    }

    // === Heterogeneous map: find(string_view) -> zero allocation ===
    {
        auto start = std::chrono::steady_clock::now();
        volatile int sum = 0;
        for (int i = 0; i < N; ++i) {
            auto it = hetero_map.find(lookup_key);  // NO allocation!
            if (it != hetero_map.end()) sum += it->second;
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "Heterogeneous map (no alloc): " << ms << " ms\n";
    }

    // === Also works with const char* ===
    {
        const char* raw = "key_number_999_with_some_extra_padding";
        auto it = hetero_map.find(raw);  // const char* -> string_view, no allocation
        if (it != hetero_map.end())
            std::cout << "\nFound: " << it->first << " = " << it->second << "\n";
        // Output: Found: key_number_999_with_some_extra_padding = 999
    }

    return 0;
}
// Typical output:
// Normal map (allocating):      ~120 ms
// Heterogeneous map (no alloc): ~40 ms  (3x faster!)
```

Three times faster just by enabling two type aliases and switching the map's template parameters. In a text-heavy workload (a parser, a configuration reader, a cache) that difference shows up directly in profiler traces.

**How it works:**

- When `find()` is called with a `std::string_view` on a normal `unordered_map<std::string, int>`, the compiler must construct a temporary `std::string` (heap allocation) because `find()` only accepts `const std::string&`.
- With heterogeneous lookup enabled, `find()` accepts any type that the hash and equal functors can handle - `string_view` is hashed and compared directly against the stored `std::string` keys without any allocation.
- The performance difference is most pronounced with long keys (where string allocation + copying is expensive) and in tight loops with millions of lookups.

### Q3: Implement a custom transparent hash for a struct with multiple members

The string key case is common, but you may also need heterogeneous lookup for domain types. The pattern generalizes: define a lightweight lookup type that avoids heap allocation, then write hash and equality functors that handle both the full type and the lightweight type:

```cpp
#include <iostream>
#include <unordered_map>
#include <string>
#include <string_view>
#include <functional>

// === Domain type: Employee identified by (department, id) ===
struct Employee {
    std::string department;
    int id;

    bool operator==(const Employee& o) const {
        return department == o.department && id == o.id;
    }
};

// === Lightweight lookup key (no heap allocation) ===
struct EmployeeKey {
    std::string_view department;
    int id;
};

// === Transparent hash ===
struct EmployeeHash {
    using is_transparent = void;

    // Hash both Employee and EmployeeKey the same way
    static size_t combine(std::string_view dept, int id) {
        size_t h1 = std::hash<std::string_view>{}(dept);
        size_t h2 = std::hash<int>{}(id);
        // Boost-style hash combine
        return h1 ^ (h2 * 0x9e3779b97f4a7c15ULL + 0x9e3779b9 + (h1 << 6) + (h1 >> 2));
    }

    size_t operator()(const Employee& e) const noexcept {
        return combine(e.department, e.id);
    }

    size_t operator()(const EmployeeKey& k) const noexcept {
        return combine(k.department, k.id);
    }
};

// === Transparent equality ===
struct EmployeeEqual {
    using is_transparent = void;

    bool operator()(const Employee& a, const Employee& b) const noexcept {
        return a.department == b.department && a.id == b.id;
    }

    bool operator()(const Employee& a, const EmployeeKey& b) const noexcept {
        return a.department == b.department && a.id == b.id;
    }

    bool operator()(const EmployeeKey& a, const Employee& b) const noexcept {
        return a.department == b.department && a.id == b.id;
    }
};

int main() {
    std::unordered_map<Employee, double, EmployeeHash, EmployeeEqual> salaries;

    // Insert with full Employee (std::string members)
    salaries[Employee{"Engineering", 101}] = 95000.0;
    salaries[Employee{"Engineering", 102}] = 88000.0;
    salaries[Employee{"Marketing", 201}]   = 75000.0;
    salaries[Employee{"Marketing", 202}]   = 72000.0;

    // === Lookup with lightweight EmployeeKey (string_view, no allocation!) ===
    EmployeeKey lookup{"Engineering", 101};  // No heap allocation
    auto it = salaries.find(lookup);

    if (it != salaries.end()) {
        std::cout << "Found: " << it->first.department
                  << " #" << it->first.id
                  << " -> $" << it->second << "\n";
    }
    // Output: Found: Engineering #101 -> $95000

    // === Check contains with lightweight key ===
    if (salaries.contains(EmployeeKey{"Marketing", 201})) {
        std::cout << "Marketing #201 exists\n";
    }
    // Output: Marketing #201 exists

    // === Hash consistency verification ===
    EmployeeHash h;
    Employee full{"Engineering", 101};
    EmployeeKey light{"Engineering", 101};
    std::cout << "\nHash(Employee):    " << h(full) << "\n";
    std::cout << "Hash(EmployeeKey): " << h(light) << "\n";
    std::cout << "Match: " << std::boolalpha << (h(full) == h(light)) << "\n";
    // Output: Match: true (CRITICAL: hashes must match for equal objects!)

    return 0;
}
```

The hash consistency check at the end is mandatory to verify. `EmployeeHash::combine` uses the same formula for both types, guaranteeing that `hash(Employee{"Engineering",101}) == hash(EmployeeKey{"Engineering",101})`. If you use a different combine formula for the two overloads, elements will become unfindable - a very subtle bug.

**How it works:**

- `Employee` is the stored key type (owns its `std::string` department). `EmployeeKey` is a lightweight lookup type that uses `std::string_view` instead - no heap allocation.
- Both the hash and equality functors handle both types, and the `is_transparent` tag enables the container to call them with `EmployeeKey` directly.
- The hash function uses a stable combine strategy: for any `Employee` and `EmployeeKey` with the same department/id, the hash values are identical. This is **mandatory** - violating this invariant causes undefined behavior (elements won't be found).
- Equality cross-compares `string` vs `string_view` via the implicit conversion in `operator==`.

---

## Notes

- **Ordered containers got heterogeneous lookup in C++14** (with `std::less<>` as comparator). Unordered containers required C++20 because they need both transparent hash AND transparent equality.
- **`std::hash<std::string>` and `std::hash<std::string_view>` produce the same hash** for the same character sequence on all major implementations, but this is not guaranteed by the standard. Using `std::hash<std::string_view>` for both (as shown) is safe because `std::string` converts to `std::string_view`.
- **C++23 adds** heterogeneous `erase()` and `extract()` for unordered containers.
- **Thread safety:** Heterogeneous lookup is read-only (no modifications), so it's as thread-safe as any other `find()`/`count()` call on a const container.
- **Performance tip:** The biggest wins come in text-heavy workloads (parsers, databases, caches) where keys are looked up by `string_view` slices from a larger buffer.
