# Understand sentinels and the split between begin/end types in ranges

**Category:** Ranges Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges>  

---

## Topic Overview

In classic C++, `begin()` and `end()` return the **same type** (iterator). C++20 ranges allow `end()` to return a **different type** called a **sentinel**. The sentinel is a lightweight object that an iterator can be compared against to detect the end.

This is one of those things that sounds like an obscure language detail until you see why it matters. The motivating example is a null-terminated C string: you know you're done when you hit `'\0'`, not when a pointer reaches some pre-computed endpoint. Computing that endpoint requires a full pass through the string - which is exactly the work you were trying to avoid.

```cpp
Classic (same type):     [begin, end)     both are std::vector<int>::iterator
Ranges (can differ):     [begin, sentinel)  sentinel can be a different type

Example: null-terminated string
  begin = const char*     (pointer to first char)
  end   = NullSentinel     (compares == '\0')
```

The sentinel is minimal on purpose. It only needs to answer one question: "has this iterator reached the end?" It doesn't need to be dereferenceable, incrementable, or anything else an iterator has to be.

### Why Sentinels

| Scenario | Without Sentinel | With Sentinel |
| --- | --- | --- |
| C-string length | `strlen()` first (O(n)) | Check `'\0'` on the fly |
| Unbounded range | Impossible | `std::unreachable_sentinel` |
| Counted range | Separate counter | Sentinel counts down |

---

## Self-Assessment

### Q1: Explain why `std::ranges` allows `begin()` and `end()` to have different types

The comments in this example walk through the three main benefits. Pay attention to `std::unreachable_sentinel` at the end - it's a built-in sentinel that always compares not-equal to any iterator, which lets you express "this algorithm will terminate for a different reason" without needing an end iterator at all.

```cpp
#include <iostream>
#include <ranges>
#include <vector>
#include <concepts>

// A sentinel only needs to be equality-comparable with the iterator
// It does NOT need to be incrementable, dereferenceable, etc.

struct NullSentinel {
    // Only need operator== with const char*
    friend bool operator==(const char* it, NullSentinel) {
        return *it == '\0';
    }
};

int main() {
    // Classic: begin and end are the SAME type
    std::vector<int> v = {1, 2, 3};
    static_assert(std::same_as<decltype(v.begin()), decltype(v.end())>);
    std::cout << "vector: same begin/end type\n";

    // Ranges: begin and end can be DIFFERENT types
    const char* str = "Hello";
    NullSentinel sentinel;

    std::cout << "C-string via sentinel: ";
    for (auto it = str; it != sentinel; ++it) {
        std::cout << *it;
    }
    std::cout << "\n";

    // WHY different types are useful:
    // 1. Efficiency: No need to compute end() upfront
    //    - C-string: no strlen() needed
    //    - Infinite ranges: no end at all
    //    - File streams: end is EOF, not a position
    //
    // 2. Simplicity: Sentinel just answers "are we done?"
    //    - No dereference, increment, or arithmetic needed
    //    - Can encode termination condition in the TYPE
    //
    // 3. Enables new patterns:
    //    - std::unreachable_sentinel (always returns false)
    //    - Custom counted sentinels

    // std::unreachable_sentinel: never equals any iterator
    // Useful when you KNOW the loop will terminate another way
    auto pos = std::ranges::find(v.begin(), std::unreachable_sentinel, 2);
    std::cout << "Found: " << *pos << "\n";
}
// Expected output:
//   vector: same begin/end type
//   C-string via sentinel: Hello
//   Found: 2
```

The reason `std::unreachable_sentinel` is useful is that it tells the optimizer you guarantee termination. Some algorithms can generate tighter code when they don't need to check an end condition on every iteration.

---

### Q2: Implement a null-terminated string range using a custom sentinel

Here we're building a proper range type whose `begin()` and `end()` deliberately return different types. This is the pattern that makes your custom type work natively with range algorithms, `views::take`, and range-based for - all without computing the string length upfront.

```cpp
#include <iostream>
#include <ranges>
#include <concepts>

// Custom sentinel: stops at null terminator
struct NullTermSentinel {
    friend bool operator==(const char* p, NullTermSentinel) {
        return *p == '\0';
    }
};

// A range type that uses our sentinel
struct CStringRange {
    const char* str;

    const char* begin() const { return str; }
    NullTermSentinel end() const { return {}; }
    // begin() returns const char*, end() returns NullTermSentinel
    // - they are DIFFERENT types!
};

int main() {
    CStringRange hello{"Hello, World!"};

    // Works with range-based for (C++20 supports sentinel end):
    std::cout << "Chars: ";
    for (char c : hello) {
        std::cout << c;
    }
    std::cout << "\n";

    // Works with ranges algorithms:
    auto count = std::ranges::count(hello, 'l');
    std::cout << "Count of 'l': " << count << "\n";

    // Works with views:
    std::cout << "Upper 5: ";
    for (char c : hello | std::views::take(5)) {
        std::cout << c;
    }
    std::cout << "\n";

    // Prove begin/end are different types:
    using B = decltype(hello.begin());  // const char*
    using E = decltype(hello.end());    // NullTermSentinel
    static_assert(!std::same_as<B, E>);
    std::cout << "Different begin/end types: confirmed\n";
}
// Expected output:
//   Chars: Hello, World!
//   Count of 'l': 3
//   Upper 5: Hello
//   Different begin/end types: confirmed
```

Notice that `CStringRange` works with `std::views::take` even though its end type is a sentinel, not an iterator. That's the power of the sentinel abstraction - the view adaptor doesn't care what the end type is, as long as the iterator can compare equal to it.

---

### Q3: Show how pointer + sentinel avoids computing string length upfront

This example makes the performance difference concrete. The old approach needs two passes: one to compute the length, and one for the actual search. The sentinel approach does both in one pass and can stop early.

```cpp
#include <iostream>
#include <cstring>
#include <ranges>
#include <algorithm>

struct NullSentinel {
    friend bool operator==(const char* p, NullSentinel) { return *p == '\0'; }
};

int main() {
    const char* long_str = "Find_the_X_in_this_string";

    // OLD: Must compute length first
    size_t len = std::strlen(long_str);  // O(n) traversal #1
    auto it = std::find(long_str, long_str + len, 'X');  // O(n) traversal #2
    std::cout << "Old: found at index " << (it - long_str) << "\n";
    // Total: TWO traversals!

    // NEW: Sentinel stops at null, ONE traversal
    auto it2 = std::ranges::find(long_str, NullSentinel{}, 'X');
    std::cout << "New: found at index " << (it2 - long_str) << "\n";
    // Total: ONE traversal! Stops as soon as 'X' is found.

    // Even better: early termination
    // If 'X' is at position 10 of a 1000-char string:
    //   Old: strlen(1000) + find(10) = 1010 comparisons
    //   New: find(10) = 10 comparisons

    // Counted sentinel example:
    struct CountSentinel {
        int remaining;
        friend bool operator==(const int*, CountSentinel s) { return s.remaining <= 0; }
    };

    int arr[] = {10, 20, 30, 40, 50};
    std::cout << "First 3 elements: ";
    for (auto it = arr; it != CountSentinel{3}; ++it) {
        std::cout << *it << " ";
        // Decrement is handled externally in real implementation
        break;  // simplified demo
    }
    // In practice, use std::counted_iterator + std::default_sentinel
    std::cout << "\n";

    // std::views::counted: built-in counted range
    std::cout << "Counted view: ";
    for (int x : std::views::counted(arr, 3)) {
        std::cout << x << " ";
    }
    std::cout << "\n";
}
// Expected output:
//   Old: found at index 10
//   New: found at index 10
//   First 3 elements: 10
//   Counted view: 10 20 30
```

The `std::views::counted` example at the end shows the standard library's built-in solution for counted ranges - you give it a starting iterator and a count, and it handles the sentinel internally. You don't need to write `CountSentinel` by hand in real code.

---

## Notes

- **`std::sentinel_for<S, I>`** concept: requires `S` to be equality-comparable with iterator `I`.
- **`std::unreachable_sentinel`:** Always compares `!=` to any iterator. Use when you guarantee the loop terminates another way.
- **`std::default_sentinel`:** A built-in empty sentinel type used by `counted_iterator`, `istream_view`, etc.
- **Range-based for loop (C++20):** Supports different begin/end types: `for (auto x : range)` works even if `begin()` and `end()` return different types.
- **`std::ranges::subrange`** can package an iterator-sentinel pair into a range.
