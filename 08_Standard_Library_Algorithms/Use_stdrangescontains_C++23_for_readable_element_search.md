# Use std::ranges::contains (C++23) for readable element search

**Category:** Standard Library Algorithms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges/contains>  

---

## Topic Overview

`std::ranges::contains` (C++23) provides a direct, readable way to check if a range contains a specific value. Before C++23, you had to write the awkward `std::find(b, e, v) != e` idiom - which works, but forces the reader to decode "find-then-compare-to-end" just to understand "does it exist?"

### Before vs After

The improvement is purely about readability. The underlying algorithm is the same O(n) linear scan:

```cpp
// Pre-C++23 - awkward
if (std::find(v.begin(), v.end(), 42) != v.end()) { ... }
if (std::ranges::find(v, 42) != v.end()) { ... }

// C++23 - clean and readable
if (std::ranges::contains(v, 42)) { ... }
```

### Overloads

Both a value-search and a subrange-search variant are available:

```cpp
#include <algorithm>

// Check if range contains a value
bool std::ranges::contains(range, value);
bool std::ranges::contains(range, value, proj);

// Check if range contains a subrange
bool std::ranges::contains_subrange(range, subrange);
bool std::ranges::contains_subrange(range, subrange, pred, proj1, proj2);
```

### Complexity

Use this table to pick the right tool based on what you know about the data:

| Algorithm | Complexity | Returns |
| --- | --- | --- |
| `contains(range, val)` | O(n) | `bool` |
| `find(range, val)` | O(n) | Iterator |
| `count(range, val)` | O(n) | Count |
| `binary_search(range, val)` | O(log n) | `bool` (sorted only) |

---

## Self-Assessment

### Q1: Replace std::find(b,e,v) != e with std::ranges::contains(range, value)

The main message here is expressiveness. `contains` reads like English and works on any range type - strings, sets, lists, whatever:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <set>

int main() {
    std::vector<int> numbers = {10, 20, 30, 40, 50};

    // === Old way (pre-C++23) ===
    if (std::find(numbers.begin(), numbers.end(), 30) != numbers.end()) {
        std::cout << "Old way: found 30\n";
    }

    // === New way (C++23) ===
    if (std::ranges::contains(numbers, 30)) {
        std::cout << "New way: found 30\n";
    }

    if (!std::ranges::contains(numbers, 99)) {
        std::cout << "99 not found\n";
    }

    // Works with any range, including strings
    std::string text = "Hello, World!";
    if (std::ranges::contains(text, 'W')) {
        std::cout << "Found 'W' in text\n";
    }

    // Works with sets, lists, etc.
    std::set<std::string> allowed = {"admin", "user", "guest"};
    std::string role = "user";
    if (std::ranges::contains(allowed, role)) {
        std::cout << "Role '" << role << "' is allowed\n";
    }

    // === contains_subrange - check if a sequence appears ===
    std::vector<int> haystack = {1, 2, 3, 4, 5, 6};
    std::vector<int> needle = {3, 4, 5};

    if (std::ranges::contains_subrange(haystack, needle)) {
        std::cout << "Subrange {3,4,5} found in haystack\n";
    }

    return 0;
}
```

One thing to keep in mind: for `std::set` and `std::map`, prefer the member `.contains()` method (C++20) over the generic algorithm - it's O(log n) instead of O(n).

### Q2: Use contains with a projection to check if a range of structs contains a given field value

Projections make `contains` genuinely powerful for struct searches. Instead of writing a custom lambda, you pass a pointer-to-member and let the algorithm project each element down to the field you care about:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>

struct Product {
    std::string name;
    int id;
    double price;
};

int main() {
    std::vector<Product> catalog = {
        {"Laptop",     101, 999.99},
        {"Mouse",      102,  29.99},
        {"Keyboard",   103,  79.99},
        {"Monitor",    104, 399.99},
        {"Headphones", 105, 149.99}
    };

    // === Search by ID using projection ===
    int search_id = 103;
    if (std::ranges::contains(catalog, search_id, &Product::id)) {
        std::cout << "Product with ID " << search_id << " exists\n";
    }

    // === Search by name ===
    std::string search_name = "Mouse";
    if (std::ranges::contains(catalog, search_name, &Product::name)) {
        std::cout << "'" << search_name << "' is in the catalog\n";
    }

    // === Search with lambda projection ===
    // Check if any product costs less than $50
    // Note: contains checks equality, so we need a different approach for ranges
    // For threshold checks, use ranges::any_of instead
    if (std::ranges::any_of(catalog, [](double p) { return p < 50.0; }, &Product::price)) {
        std::cout << "There's a product under $50\n";
    }

    // === Practical: authorization check ===
    struct Permission {
        std::string user;
        std::string action;
    };

    std::vector<Permission> acl = {
        {"alice", "read"}, {"alice", "write"},
        {"bob", "read"}, {"carol", "admin"}
    };

    // Check if any permission grants "write" action
    if (std::ranges::contains(acl, std::string("write"), &Permission::action)) {
        std::cout << "Write permission exists in ACL\n";
    }

    return 0;
}
```

The projection syntax `&Product::id` is cleaner than a lambda like `[](const Product& p) { return p.id; }`. It's one of the nicest quality-of-life features of the ranges library.

### Q3: Show the difference between contains (existence) and count (frequency) for sorted/unsorted ranges

This example makes a practical point: `contains` short-circuits on the first match, while `count` always scans the entire range. For sorted data you can do even better with `binary_search` and `equal_range`:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> data = {5, 3, 5, 1, 5, 2, 5, 4, 5};

    int target = 5;

    // === contains: just existence (bool) - short-circuits on first match ===
    bool exists = std::ranges::contains(data, target);
    std::cout << "contains(5): " << std::boolalpha << exists << "\n";  // true

    // === count: frequency (size_t) - always scans entire range ===
    auto freq = std::ranges::count(data, target);
    std::cout << "count(5):    " << freq << "\n";  // 5

    // === Performance difference ===
    // contains: stops at first match -> O(1) best case, O(n) worst
    // count:    always O(n) - must scan everything

    // For sorted ranges, use binary_search (O(log n)) for existence:
    std::vector<int> sorted = {1, 2, 3, 4, 5, 5, 5, 6, 7};

    bool found_bs = std::ranges::binary_search(sorted, 5);
    std::cout << "\nbinary_search(5): " << found_bs << "\n";  // true

    // For frequency in sorted ranges, use equal_range (O(log n)):
    auto [lo, hi] = std::ranges::equal_range(sorted, 5);
    std::cout << "equal_range count: " << std::distance(lo, hi) << "\n";  // 3

    // === Summary table ===
    std::cout << "\n--- When to use what ---\n";
    std::cout << "Need          | Unsorted           | Sorted\n";
    std::cout << "Existence     | contains O(n)      | binary_search O(log n)\n";
    std::cout << "Frequency     | count O(n)         | equal_range O(log n)\n";
    std::cout << "Position      | find O(n)          | lower_bound O(log n)\n";

    // === contains vs find: readability ===
    // Use contains when you only need a bool
    // Use find when you also need the iterator to the element

    return 0;
}
```

The rule of thumb in the comments is worth memorizing: reach for `contains` when you only care about a yes/no answer, and `find` when you need to do something with the actual element once you've found it.

---

## Notes

- **`std::ranges::contains` requires C++23.** Compiler support: GCC 13+, Clang 17+, MSVC 19.37+.
- `contains` is essentially syntactic sugar for `find(range, val) != end(range)`, but significantly more readable.
- **`contains_subrange`** checks if one range appears as a contiguous subsequence of another - equivalent to `search(haystack, needle) != end(haystack)`.
- For **associative containers** (`set`, `map`), prefer the member `.contains()` method (C++20) - it's O(log n) vs the O(n) generic algorithm.
- Projections make `contains` extremely powerful for searching structs by field without writing custom predicates.
