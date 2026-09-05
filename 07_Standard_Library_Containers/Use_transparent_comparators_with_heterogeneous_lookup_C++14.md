# Use transparent comparators with heterogeneous lookup (C++14)

**Category:** Standard Library Containers  
**Standard:** C++14  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/less_void>  

---

## Topic Overview

C++14 introduced the **void specialization** of `std::less<>` (and other comparators), which enables **heterogeneous lookup** - searching an ordered container with a type different from the key type, avoiding unnecessary conversions. This is one of the most impactful but underused C++14 features, and it shows up in real performance work whenever you have string-keyed maps or sets in a hot path.

### What Changed in C++14

Before C++14, every lookup on a `set<string>` using a string literal had to pay a detour through a temporary `std::string`. C++14 made it possible to skip that entirely:

```cpp
// Before C++14:
//   std::less<std::string>  ->  operator()(const string&, const string&)
//   find("hello")           ->  const char* -> std::string temporary -> comparison

// C++14:
//   std::less<>             ->  template operator()(const T&, const U&)
//   find("hello")           ->  const char* compared directly via operator< -> NO temporary
```

### The is_transparent Protocol

The reason this trips people up is that it requires a two-step opt-in. The comparator signals its capability by defining `is_transparent`, and the container checks for that signal before offering the heterogeneous overloads. Without the signal, the container only provides `find(const key_type&)` - and your argument gets converted whether you like it or not.

Here's what the standard's implementation of `std::less<void>` looks like under the hood:

```cpp
// From <functional>:
template <>
struct less<void> {
    using is_transparent = void;   // <- enables heterogeneous overloads

    template <typename T, typename U>
    constexpr auto operator()(T&& t, U&& u) const
        -> decltype(std::forward<T>(t) < std::forward<U>(u))
    {
        return std::forward<T>(t) < std::forward<U>(u);
    }
};
```

When `is_transparent` exists in the comparator, the affected lookup functions are:

- `find()`, `count()`, `lower_bound()`, `upper_bound()`, `equal_range()` - all gain template overloads
- These accept any type comparable with the key, not just the key type itself

### Standard Transparent Comparators

All the standard comparison functors have void specializations. If the table feels like a lot, the pattern is simple: wherever you'd write `std::less<T>`, write `std::less<>` instead to get the transparent version.

| Comparator | Transparent version | Operation |
| --- | --- | --- |
| `std::less<T>` | `std::less<>` | `a < b` |
| `std::greater<T>` | `std::greater<>` | `a > b` |
| `std::less_equal<T>` | `std::less_equal<>` | `a <= b` |
| `std::greater_equal<T>` | `std::greater_equal<>` | `a >= b` |
| `std::equal_to<T>` | `std::equal_to<>` | `a == b` |

---

## Self-Assessment

### Q1: Use std::less<> (void specialization) in a std::set<std::string> to enable lookup by string_view

The example below demonstrates all the lookup functions that benefit, not just `find()`. Pay attention to the comment near the end explaining why `std::less<>` actually works - the mechanism is slightly surprising.

```cpp
#include <iostream>
#include <set>
#include <string>
#include <string_view>

int main() {
    // === Non-transparent set (default) ===
    std::set<std::string> normal_set = {"apple", "banana", "cherry"};

    // This compiles but creates a temporary std::string from the literal:
    auto it1 = normal_set.find("banana");  // const char* -> string temporary
    // For string_view, it doesn't compile without implicit conversion:
    // std::string_view sv = "banana";
    // auto it = normal_set.find(sv);  // ERROR in some implementations pre-C++20!

    // === Transparent set ===
    std::set<std::string, std::less<>> trans_set = {"apple", "banana", "cherry"};
    //                     ^^^^^^^^^^
    //                     void specialization = transparent comparator

    // Direct lookup with const char* - no temporary string!
    auto it2 = trans_set.find("banana");
    if (it2 != trans_set.end())
        std::cout << "Found (literal): " << *it2 << "\n";  // Found: banana

    // Direct lookup with string_view - no temporary string!
    std::string_view sv = "cherry";
    auto it3 = trans_set.find(sv);
    if (it3 != trans_set.end())
        std::cout << "Found (sv): " << *it3 << "\n";  // Found: cherry

    // Works with all lookup functions:
    std::cout << "count(\"apple\"): " << trans_set.count("apple") << "\n";  // 1

    // lower_bound / upper_bound with heterogeneous types:
    auto lb = trans_set.lower_bound(std::string_view("b"));
    auto ub = trans_set.lower_bound(std::string_view("c"));
    std::cout << "Between 'b' and 'c': ";
    for (auto it = lb; it != ub; ++it)
        std::cout << *it << " ";
    std::cout << "\n";  // banana

    // === Why std::less<> works ===
    // std::less<>::operator()(const char*, const std::string&) calls:
    //   "banana" < std::string("banana")  ->  uses std::string's operator<
    // which accepts const char* on the left via implicit conversion? NO!
    // Actually: operator<(const char*, const basic_string&) is a free function
    // defined by <string>, which does character comparison without allocation.

    return 0;
}
```

The `lower_bound`/`upper_bound` range query at the end is worth noting specifically. Because the comparator is transparent, you can pass a `string_view("b")` as a bound and it works without constructing a `std::string` - very clean for filtering prefixes or ranges.

- `std::less<>` is the void specialization - a template comparator that accepts any two comparable types.
- It declares `using is_transparent = void;`, which signals the container to provide template overloads of the lookup functions.
- `std::string` has `operator<` overloads accepting `const char*` and `std::string_view` directly - so no conversion is needed.
- The transparent `find()` uses those overloads, bypassing `std::string` construction entirely.

### Q2: Show that lookup with a string literal on a std::set<std::string, std::less<>> avoids constructing a std::string

This example makes the benefit concrete by counting allocations. The strings are intentionally long to defeat SSO (Small String Optimization), which stores short strings inline and would otherwise hide the allocations.

```cpp
#include <iostream>
#include <set>
#include <string>
#include <cstdlib>

// === Global allocation counter ===
static int g_allocs = 0;

void* operator new(std::size_t sz) {
    ++g_allocs;
    return std::malloc(sz);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

int main() {
    // Use long strings to defeat SSO (Small String Optimization)
    // SSO stores short strings inline (~15-22 chars), so they don't allocate.
    const char* key = "a_very_long_string_that_exceeds_sso_buffer_size_definitely";

    // === Build sets (allocations from insert are expected) ===
    std::set<std::string> normal;
    std::set<std::string, std::less<>> transparent;

    normal.insert(key);
    transparent.insert(key);

    // === Non-transparent find ===
    g_allocs = 0;
    for (int i = 0; i < 1000; ++i)
        normal.find(key);  // each find creates temporary std::string
    int normal_allocs = g_allocs;

    // === Transparent find ===
    g_allocs = 0;
    for (int i = 0; i < 1000; ++i)
        transparent.find(key);  // no temporary - direct comparison
    int transparent_allocs = g_allocs;

    std::cout << "1000 finds on non-transparent set: " << normal_allocs << " allocations\n";
    std::cout << "1000 finds on transparent set:     " << transparent_allocs << " allocations\n";
    // Expected output:
    // 1000 finds on non-transparent set: 1000 allocations
    // 1000 finds on transparent set:     0 allocations

    // === The savings are real ===
    // For a hot lookup path, eliminating 1 allocation per find() is significant.
    // Each allocation involves: size calculation, free-list search (or mmap),
    // memory copy, and later deallocation - easily 50-200ns per call.

    return 0;
}
```

A thousand calls, a thousand allocations on one side - zero on the other. That's the clearest possible demonstration that the difference is real and not just theoretical. At 50-200ns per allocation (a realistic estimate for a general-purpose allocator), 1000 unnecessary allocations cost 50-200 microseconds for a task that should take essentially nothing.

- Without transparent comparator: `find(const char*)` must convert to `std::string` to match the non-template `find(const key_type&)` signature. For long strings, that means a heap allocation.
- With transparent comparator: `find(const char*)` uses the template overload that compares `const char*` directly with `std::string` - zero allocations.
- SSO may hide this effect for short strings (typically fewer than 15-22 chars), which is why the example uses a deliberately long key.

### Q3: Implement a custom transparent comparator for a set of custom objects

Here's the full pattern for rolling your own transparent comparator. The example provides two comparators for the same type - one ordering by SKU (an integer), one by name (a string) - so you can see both numeric and string heterogeneous lookup.

```cpp
#include <iostream>
#include <set>
#include <string>
#include <string_view>

// === User-defined type ===
struct Product {
    int sku;
    std::string name;
    double price;

    friend std::ostream& operator<<(std::ostream& os, const Product& p) {
        return os << "{sku=" << p.sku << ", name=" << p.name << ", price=" << p.price << "}";
    }
};

// === Custom transparent comparator ===
// Sorts by SKU, allows lookup by int (SKU number)
struct BySku {
    using is_transparent = void;  // <- MUST define this

    // Product vs Product
    bool operator()(const Product& a, const Product& b) const {
        return a.sku < b.sku;
    }
    // Product vs int (heterogeneous lookup by SKU)
    bool operator()(const Product& p, int sku) const {
        return p.sku < sku;
    }
    bool operator()(int sku, const Product& p) const {
        return sku < p.sku;
    }
};

// === Another comparator: by name, lookup by string_view ===
struct ByProductName {
    using is_transparent = void;

    bool operator()(const Product& a, const Product& b) const {
        return a.name < b.name;
    }
    bool operator()(const Product& p, std::string_view name) const {
        return p.name < name;
    }
    bool operator()(std::string_view name, const Product& p) const {
        return name < p.name;
    }
};

int main() {
    // === Set sorted by SKU ===
    std::set<Product, BySku> by_sku;
    by_sku.insert({1001, "Laptop", 999.99});
    by_sku.insert({1002, "Mouse", 29.99});
    by_sku.insert({1003, "Keyboard", 79.99});
    by_sku.insert({1004, "Monitor", 449.99});

    // Lookup by int - no Product constructed!
    auto it = by_sku.find(1002);
    if (it != by_sku.end())
        std::cout << "Found SKU 1002: " << *it << "\n";
    // Output: Found SKU 1002: {sku=1002, name=Mouse, price=29.99}

    // Range query: SKUs 1002-1003
    auto lo = by_sku.lower_bound(1002);
    auto hi = by_sku.upper_bound(1003);
    std::cout << "SKUs 1002-1003:\n";
    for (auto it = lo; it != hi; ++it)
        std::cout << "  " << *it << "\n";

    // === Set sorted by name ===
    std::set<Product, ByProductName> by_name;
    by_name.insert({1001, "Laptop", 999.99});
    by_name.insert({1002, "Mouse", 29.99});
    by_name.insert({1003, "Keyboard", 79.99});

    // Lookup by string_view - no Product constructed!
    auto it2 = by_name.find(std::string_view("Mouse"));
    if (it2 != by_name.end())
        std::cout << "\nFound by name: " << *it2 << "\n";
    // Output: Found by name: {sku=1002, name=Mouse, price=29.99}

    // Lookup by const char* - also no Product or string constructed!
    auto it3 = by_name.find("Keyboard");
    if (it3 != by_name.end())
        std::cout << "Found by literal: " << *it3 << "\n";

    return 0;
}
```

Notice that `ByProductName` has three overloads: `(Product, Product)` for the normal tree ordering, and `(Product, string_view)` plus `(string_view, Product)` for the two lookup directions. Both directional overloads are needed because during tree traversal the library sometimes compares as `comp(node_key, search_key)` and sometimes as `comp(search_key, node_key)`.

The checklist for a working custom transparent comparator:

- Define `using is_transparent = void;` to opt in.
- Provide `operator()` overloads for `(Key, LookupType)` and `(LookupType, Key)` - both directions.
- The ordering must be **strict weak ordering** and consistent across all overloads: if `comp(a, b)` is true, then `comp(b, a)` must be false.

---

## Notes

- **C++14 origin:** The void specialization and `is_transparent` protocol were introduced in C++14 (N3657). This is one of the most impactful but underused C++14 features.
- **Why not default?** Making `std::less<>` the default could silently break code where `a < b` compiles but gives wrong results for heterogeneous types.
- **`std::map` too:** `std::map<std::string, V, std::less<>>` enables heterogeneous `find()`, `count()`, `lower_bound()`, `upper_bound()`, `equal_range()`, and C++20's `contains()`.
- **Not for `insert`/`emplace`:** Heterogeneous lookup only affects **read** operations. You still need the correct key type for insertion.
- **C++20 extension:** `std::unordered_set`/`std::unordered_map` gained heterogeneous lookup too - but it requires both a transparent hash and a transparent equality comparator, making it more involved than the ordered container case.
- **Performance:** Avoiding one `std::string` construction per lookup saves roughly 50-200ns (depending on string length and allocator). For high-frequency lookups, that adds up significantly.
