# Use views::as_rvalue (C++23) to move elements from a range

**Category:** Ranges Cpp20  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/as_rvalue_view>  

---

## Topic Overview

`views::as_rvalue` wraps a range so that each element is yielded as an **rvalue reference** (`T&&`). This is the ranges equivalent of `std::make_move_iterator` - it allows move-based algorithms and pipelines without manual iterator wrapping.

The problem it solves is simple: `ranges::copy(src, dest)` copies by default, because dereferencing a normal iterator gives you an lvalue. If you want to move instead of copy, you need the iterator to hand out rvalue references. That's exactly what `views::as_rvalue` does.

### Equivalence

```cpp
views::as_rvalue(r)  is equivalent to  r | views::transform(std::move)
                     is equivalent to  subrange(make_move_iterator(r.begin()),
                                               make_move_iterator(r.end()))
```

But `views::as_rvalue` is more composable and integrates with pipelines cleanly.

### When the Source Is Already Rvalue

If the range's reference type is already an rvalue reference, `views::as_rvalue` is a no-op and returns the range unchanged. There's no cost to applying it defensively.

### Typical Use Cases

| Scenario | Without as_rvalue | With as_rvalue |
| --- | --- | --- |
| Move strings to new vector | Manual loop with `std::move` | `src \| views::as_rvalue \| ranges::to<vector>()` |
| Move-assign into existing container | `ranges::copy` + `make_move_iterator` | `ranges::copy(src \| views::as_rvalue, dest)` |
| Move + filter pipeline | Write a loop | `src \| views::as_rvalue \| views::filter(pred)` |

### Important Warning

After iterating through `views::as_rvalue`, the source elements are in a **moved-from state**. They're valid but unspecified - don't read them. This is the same rule as with `std::move` anywhere else: moving transfers ownership, and you shouldn't rely on what's left behind.

---

## Self-Assessment

### Q1: Use `views::as_rvalue` to move strings from a vector into a new container without a manual loop

The proof that moves are happening is the size of `source[0]` after the operation. A moved-from `std::string` is left empty, so seeing size 0 confirms the strings were moved, not copied.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    std::vector<std::string> source = {
        std::string(100, 'A'),  // long strings to show move benefit
        std::string(100, 'B'),
        std::string(100, 'C'),
    };

    std::cout << "Before move: source[0].size() = " << source[0].size() << '\n';

    // Move all strings into a new vector using views::as_rvalue
    auto dest = source
        | std::views::as_rvalue
        | std::ranges::to<std::vector<std::string>>();

    std::cout << "After move:  source[0].size() = " << source[0].size() << '\n';
    std::cout << "dest[0].size() = " << dest[0].size() << '\n';
    std::cout << "dest[1] starts with: " << dest[1][0] << '\n';
}
// Expected output:
// Before move: source[0].size() = 100
// After move:  source[0].size() = 0
// dest[0].size() = 100
// dest[1] starts with: B
```

Without `views::as_rvalue`, the `ranges::to` call would invoke the copy constructor on each string, leaving the source intact. With it, the move constructor fires instead - much cheaper for large strings, and essential for move-only types.

### Q2: Show that `views::as_rvalue` enables `make_move_iterator` semantics in range algorithms

The commented-out "old way" shows what you'd write in C++20 without `views::as_rvalue`. The new way is shorter and fits naturally into a pipeline. Pay attention to the pipeline example at the end: `filter` receives `const std::string&` from the rvalue reference, so it inspects each string safely without accidentally consuming it.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    std::vector<std::string> src = {"hello", "world", "foo", "bar"};
    std::vector<std::string> dst(src.size());

    // OLD WAY: make_move_iterator
    // std::copy(std::make_move_iterator(src.begin()),
    //           std::make_move_iterator(src.end()),
    //           dst.begin());

    // NEW WAY: views::as_rvalue + ranges::copy
    std::ranges::copy(src | std::views::as_rvalue, dst.begin());

    std::cout << "dst: ";
    for (const auto& s : dst) std::cout << '"' << s << "\" ";
    std::cout << '\n';

    std::cout << "src after move: ";
    for (const auto& s : src) std::cout << '"' << s << "\" ";
    std::cout << '\n';

    // Also works in pipelines:
    std::vector<std::string> words = {"alpha", "bravo", "charlie", "delta"};
    auto long_words = words
        | std::views::as_rvalue
        | std::views::filter([](const std::string& s) { return s.size() > 4; })
        | std::ranges::to<std::vector>();

    std::cout << "Moved long words: ";
    for (const auto& w : long_words) std::cout << w << ' ';
    std::cout << '\n';
}
// Expected output:
// dst: "hello" "world" "foo" "bar"
// src after move: "" "" "" ""
// Moved long words: alpha bravo charlie delta
```

The `src after move` output is all empty strings - confirming that `ranges::copy` performed moves, not copies. The pipeline example shows that composing `as_rvalue` with `filter` works correctly because `filter` takes its predicate argument by `const&`, so it reads the string value without triggering the move.

### Q3: Explain when `views::as_rvalue` avoids copies that `ranges::copy` would otherwise make

**The core issue:** `ranges::copy(src, dest_it)` copies elements by default because it dereferences source iterators to **lvalue references** (`T&`). To get move semantics, you need rvalue references (`T&&`).

| Operation | Iterator dereference | Effect on elements |
| --- | --- | --- |
| `ranges::copy(src, out)` | `*it` -> `T&` (lvalue) | **Copies** each element |
| `ranges::copy(src \| views::as_rvalue, out)` | `*it` -> `T&&` (rvalue) | **Moves** each element |

**When this matters (avoids copies):**

1. **Expensive-to-copy types** - `std::string`, `std::vector<T>`, `std::unique_ptr<T>` (which can't even be copied).
2. **Transfer ownership** - moving elements from one container to another when the source is no longer needed.
3. **Building a new container** from a filtered subset - `filter | as_rvalue | to<vector>` moves only matching elements.

**When it doesn't matter:**

- Trivially-copyable types (`int`, `double`, `char`) - move and copy are identical.
- If the range already yields rvalue references, `views::as_rvalue` is a no-op.

The `unique_ptr` case is the most extreme - it's not just a performance win, it's a correctness requirement. `unique_ptr` cannot be copied at all, so without `as_rvalue` the whole pipeline would fail to compile.

```cpp
#include <iostream>
#include <memory>
#include <ranges>
#include <vector>

int main() {
    // unique_ptr can't be copied - as_rvalue is REQUIRED
    std::vector<std::unique_ptr<int>> ptrs;
    ptrs.push_back(std::make_unique<int>(10));
    ptrs.push_back(std::make_unique<int>(20));
    ptrs.push_back(std::make_unique<int>(30));

    // This would NOT compile:
    // auto copied = ptrs | std::ranges::to<std::vector>();  // ERROR: deleted copy ctor

    // This works - moves each unique_ptr:
    auto moved = ptrs
        | std::views::as_rvalue
        | std::ranges::to<std::vector>();

    std::cout << "moved[0] = " << *moved[0] << '\n';
    std::cout << "moved[2] = " << *moved[2] << '\n';
    std::cout << "ptrs[0] is null? " << (ptrs[0] == nullptr) << '\n';
}
// Expected output:
// moved[0] = 10
// moved[2] = 30
// ptrs[0] is null? 1
```

After the move, `ptrs[0]` is null because ownership transferred to `moved[0]`. This is the expected behavior when working with move-only types.

---

## Notes

- `views::as_rvalue` was added in **C++23** (P2446R2). For C++20, use `std::make_move_iterator` manually.
- It works with **any input range**, not just contiguous containers.
- Composing `as_rvalue` with `filter` is safe - the filter predicate receives `const T&` from the rvalue, so it doesn't accidentally move.
- For `std::unique_ptr` transfers, `views::as_rvalue` is essential since `unique_ptr` is non-copyable.
