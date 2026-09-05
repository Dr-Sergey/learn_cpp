# Use std::views::as_const (C++23) to create const views of mutable ranges

**Category:** Standard Library Utilities  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/as_const_view>  

---

## Topic Overview

`std::views::as_const` (C++23) is a range adaptor that wraps a range so its elements are accessed as `const`. It prevents modification through the view without making the underlying range itself `const`. This is useful for passing mutable containers to functions that should only read - enforced at compile time.

The reason this matters more than just taking `const vector<int>&` is that `views::as_const` works generically with any range type, composes naturally in pipelines, and documents intent at the exact point where read-only access begins. You can pass the mutable container in and freeze access to it midway through a pipeline.

### How It Works

```cpp
Before:  vector<int>&  -> elements are int&       (mutable)
After:   vector<int>&  --| views::as_const -> elements are const int& (read-only)
                          -> underlying vector still mutable
```

### Comparison of Approaches

| Technique | Prevents modification? | Documents intent? | Works with ranges? |
| --- | --- | --- | --- |
| `const std::vector<int>&` | Yes | Somewhat | No (must know container type) |
| `std::span<const int>` | Yes | Yes | Only contiguous ranges |
| `views::as_const` | Yes | Yes | Any range |
| `std::as_const(vec)` | Yes | Yes | Not a view, not lazy |

### Core Example

Watch how the `const_view` prevents modification at compile time while `v.push_back(6)` on the underlying vector is still perfectly valid - the view just sees the added element.

```cpp
#include <ranges>
#include <vector>
#include <iostream>

void print_elements(auto&& range) {
    for (const auto& elem : range)
        std::cout << elem << " ";
    std::cout << "\n";
}

void try_modify(auto&& range) {
    for (auto& elem : range) {
        // elem = 0; // ERROR if range is as_const view
        (void)elem;
    }
}

int main() {
    std::vector<int> v{1, 2, 3, 4, 5};

    // Create a const view - elements are const int&
    auto const_view = v | std::views::as_const;

    print_elements(const_view); // Works fine: 1 2 3 4 5

    // try_modify(const_view); // COMPILE ERROR: can't assign to const int&

    // The underlying vector is still mutable:
    v.push_back(6);
    print_elements(const_view); // 1 2 3 4 5 6 - view sees the change!
}
```

---

## Self-Assessment

### Q1: Apply views::as_const to prevent modification through a range view passed to a function

**Answer:**

Here the goal is to make accidental modification of the data impossible even if someone edits the `analyze` function later and writes a loop that tries to assign.

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>

// A function that should ONLY read, not modify
void analyze(std::ranges::range auto&& data) {
    int count = 0;
    int sum = 0;
    for (const auto& val : data) {
        sum += val;
        ++count;
    }
    std::cout << "Count: " << count << ", Sum: " << sum
              << ", Avg: " << (count ? sum / count : 0) << "\n";

    // If someone accidentally tries to modify:
    // for (auto& val : data) val = 0; // COMPILE ERROR with as_const!
}

// A pipeline that filters and views as const
void process_scores(std::vector<int>& scores) {
    // Pass only passing scores, as read-only
    auto passing = scores
        | std::views::filter([](int s) { return s >= 60; })
        | std::views::as_const;  // prevent modification through the view

    analyze(passing);

    // Without as_const, a bug in analyze() could corrupt the data:
    // for (auto& s : passing) s = 100; // Would compile without as_const!
}

int main() {
    std::vector<int> scores{95, 42, 78, 55, 88, 60, 33};

    process_scores(scores);
    // Output: Count: 4, Sum: 321, Avg: 80

    // Verify scores are unchanged:
    for (int s : scores) std::cout << s << " ";
    std::cout << "\n";
    // Output: 95 42 78 55 88 60 33
}
```

By appending `| std::views::as_const` to the pipeline, the elements become `const int&` through the view. Any function receiving this view cannot modify elements - enforced at compile time. The underlying vector remains mutable for the owner.

### Q2: Show that views::as_const is more expressive than const_cast and documents intent

**Answer:**

The naming is important here. `const_cast` in C++ is semantically associated with removing const - that's its primary purpose. Using it to add const is technically legal but reads as backwards. `views::as_const` states the intent directly.

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <iostream>
#include <type_traits>

// BAD: Using const_cast to add const
void read_only_bad(const std::vector<int>& v) {
    // Problem 1: Requires knowing the exact container type
    // Problem 2: const_cast is usually for REMOVING const - confusing
    // Problem 3: Doesn't work with range pipelines
    for (auto x : v) std::cout << x << " ";
}

// GOOD: Using views::as_const
void read_only_good(std::ranges::range auto&& r) {
    // Works with ANY range type - generic
    // Intent is crystal clear: "I want const access"
    for (const auto& x : r) std::cout << x << " ";
}

int main() {
    std::vector<int> v{10, 20, 30};

    // --- Approach 1: const ref (works but inflexible) ---
    read_only_bad(v);
    std::cout << "\n";

    // --- Approach 2: views::as_const (expressive, generic) ---
    auto cv = v | std::views::as_const;
    read_only_good(cv);
    std::cout << "\n";

    // Views::as_const works in pipelines:
    auto pipeline = v
        | std::views::transform([](int x) { return x * 2; })
        | std::views::as_const;  // const-ify the transformed results

    // Prove elements are const:
    using ElemType = std::ranges::range_reference_t<decltype(cv)>;
    static_assert(std::is_const_v<std::remove_reference_t<ElemType>>,
                  "Elements must be const");
    std::cout << "Elements are const: true\n";

    // views::as_const documents intent in the code:
    //   v | views::as_const    <- "I want read-only access"
    // vs:
    //   const_cast<const std::vector<int>&>(v)  <- "What? Why cast?"
    //   std::as_const(v)      <- Only works on the whole container, not views
}
```

`views::as_const` is self-documenting - it states exactly what it does ("view this range as const"). It works generically with any range, composes in pipelines, and is the idiomatic C++23 way to enforce read-only access. `const_cast` is semantically associated with removing const, and `std::as_const` doesn't create a lazy view.

### Q3: Verify that views::as_const propagates constness to the elements, not just the view

**Answer:**

This is an important subtlety: the view object itself is not const (you can copy it, advance its iterators), but dereferencing an iterator through it gives you a `const T&`. Only the element access is frozen.

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <iostream>
#include <type_traits>

int main() {
    std::vector<std::string> names{"Alice", "Bob", "Charlie"};

    // Create an as_const view
    auto cv = names | std::views::as_const;

    // === Verify: element references are const ===
    using ViewRef = std::ranges::range_reference_t<decltype(cv)>;
    // ViewRef is "const std::string&"
    static_assert(std::is_same_v<ViewRef, const std::string&>,
                  "Elements should be const string&");

    // === The VIEW object itself is not const ===
    // (you can copy/move/reassign the view)
    auto cv2 = cv; // copy the view - fine
    (void)cv2;

    // === Elements are truly const - modification fails ===
    for (auto& elem : cv) {
        // elem is "const std::string&"
        std::cout << elem << "\n";
        // elem = "Modified"; // COMPILE ERROR: assignment of read-only reference
        // elem.clear();      // COMPILE ERROR: passing 'const string' to non-const
    }
    // Output:
    // Alice
    // Bob
    // Charlie

    // === Contrast: without as_const, elements are mutable ===
    for (auto& elem : names) {
        // elem is "std::string&" - mutable!
        elem += "!"; // This compiles and modifies the original
    }
    // names is now: {"Alice!", "Bob!", "Charlie!"}

    // === Key insight: as_const wraps the ITERATOR ===
    // It changes iterator::operator* to return const T& instead of T&
    // The view itself, the container, and the iterators are still mutable
    // objects - only the dereferenced element type gains const.

    auto it = cv.begin();
    ++it; // iterator is mutable (can advance)
    // *it = "X"; // but dereferencing gives const ref - can't assign
    std::cout << "Second: " << *it << "\n"; // "Bob!"
}
```

`views::as_const` modifies the iterator's `operator*` to return `const T&` instead of `T&`. The view object itself is not const (you can copy it, iterate it, advance iterators). The underlying container is not const either. Only the **dereference result** gains const, which is exactly what you want - read-only access to elements while keeping everything else functional.

---

## Notes

- **Idempotent:** Applying `views::as_const` to an already-const range is a no-op (returns the range unchanged).
- **Composability:** `as_const` works anywhere in a pipeline: `v | filter(...) | as_const | transform(...)`. Elements are const from that point onward.
- **`std::as_const` (C++17)** is different: it returns a `const T&` to the whole object. It's not a view, not lazy, and doesn't compose with ranges.
- **Use cases:** Passing ranges to APIs that should never modify data, protecting shared data in concurrent pipelines, making intent clear in code review.
- Compile with `-std=c++23 -Wall -Wextra`.
