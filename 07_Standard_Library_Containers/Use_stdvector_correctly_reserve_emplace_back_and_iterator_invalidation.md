# Use std::vector correctly: reserve, emplace_back, and iterator invalidation

**Category:** Standard Library Containers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/container/vector>  

---

## Topic Overview

`std::vector` is the most used C++ container - a dynamic array backed by contiguous memory. Most of the time it "just works," but there are three specific areas where a shallow understanding leads to real bugs and hidden performance costs: knowing when to `reserve`, choosing `emplace_back` over `push_back`, and respecting iterator invalidation. Let's look at each one.

### Memory Model

Before diving into the operations, it helps to have the right mental picture of what a vector actually is. There are two numbers that matter - `size` (how many elements exist) and `capacity` (how many elements fit before the vector has to move). They live separately:

```cpp
vector v = {1, 2, 3}

Stack:               Heap:
┌────────────┐       ┌───┬───┬───┬───┬───┐
│ data ──────────────>│ 1 │ 2 │ 3 │   │   │
│ size = 3   │       └───┴───┴───┴───┴───┘
│ capacity = 5│       <-- size -->
└────────────┘       <---- capacity ----->

push_back(4): fits in capacity -> just writes to slot 3. Size becomes 4.
push_back(6) after 5: exceeds capacity -> allocate new larger buffer, copy all elements, free old buffer.
```

The reallocation step is the expensive one - and it silently invalidates every iterator, pointer, and reference you held into the old buffer.

### Key Operations

Here's a quick reference for the operations you'll use most often. The complexity column is the piece worth remembering.

| Operation | Effect | Complexity |
| --- | --- | --- |
| `push_back(x)` | Copy/move x into vector | Amortized O(1) |
| `emplace_back(args...)` | Construct in-place | Amortized O(1) |
| `reserve(n)` | Ensure capacity >= n (no size change) | O(n) if reallocation |
| `resize(n)` | Change size to n (default-constructs new elements) | O(n) |
| `shrink_to_fit()` | Request reduce capacity to size | Implementation-defined |
| `insert(pos, x)` | Insert at position | O(n) - shifts elements |
| `erase(pos)` | Remove at position | O(n) - shifts elements |

### Iterator Invalidation Rules

This table is the one that trips people up. The key insight is that any operation that might trigger a reallocation will invalidate *all* iterators - not just the ones near the affected position.

| Operation | Invalidated iterators |
| --- | --- |
| `push_back` / `emplace_back` | All, **if** reallocation occurs. Otherwise none. |
| `insert` | All if reallocation. Past-the-insertion-point otherwise. |
| `erase` | Past-the-erasure-point and `end()` |
| `reserve` | All, **if** reallocation occurs |
| `resize` (growing) | All if reallocation |
| `clear` | All |
| `swap` | All (they refer to swapped-from vector) |
| Read-only (`operator[]`, `at`, iteration) | None |

---

## Self-Assessment

### Q1: Explain when iterators are invalidated in a vector and write a bug that demonstrates this

There are two distinct invalidation scenarios worth seeing side by side. The first is the reallocation bug - an iterator saved before a `push_back` that causes a resize. The second is the erase-during-iteration bug, which is one of the most common sources of undefined behavior in production C++ code.

```cpp
#include <iostream>
#include <vector>

int main() {
    // === BUG: iterator invalidation during push_back ===
    std::vector<int> v = {1, 2, 3, 4, 5};

    std::cout << "Before: size=" << v.size() << " capacity=" << v.capacity() << "\n";

    // Save an iterator to the third element
    auto it = v.begin() + 2;
    std::cout << "Iterator points to: " << *it << "\n";  // 3

    // Force reallocation by exceeding capacity
    while (v.size() < v.capacity()) v.push_back(99);  // fill to capacity
    std::cout << "Filled to capacity: size=" << v.size() << "\n";

    // This push_back WILL reallocate
    v.push_back(100);
    std::cout << "After reallocation: size=" << v.size()
              << " capacity=" << v.capacity() << "\n";

    // UNDEFINED BEHAVIOR: `it` now points to freed memory!
    // std::cout << "*it = " << *it << "\n";  // CRASH or garbage

    // === BUG: erasing during iteration ===
    std::vector<int> nums = {1, 2, 3, 4, 5, 6};

    // WRONG: erase invalidates iterators past the erasure point
    // for (auto it = nums.begin(); it != nums.end(); ++it) {
    //     if (*it % 2 == 0) nums.erase(it);  // UNDEFINED BEHAVIOR: it is now invalid
    // }

    // CORRECT: erase returns the next valid iterator
    for (auto it = nums.begin(); it != nums.end(); ) {
        if (*it % 2 == 0)
            it = nums.erase(it);   // erase returns iterator to next element
        else
            ++it;
    }

    std::cout << "After removing evens: ";
    for (int n : nums) std::cout << n << " ";
    std::cout << "\n";  // 1 3 5

    // EVEN BETTER (C++20):
    std::vector<int> nums2 = {1, 2, 3, 4, 5, 6};
    std::erase_if(nums2, [](int x) { return x % 2 == 0; });
    std::cout << "C++20 erase_if:     ";
    for (int n : nums2) std::cout << n << " ";
    std::cout << "\n";  // 1 3 5

    return 0;
}
```

The reason the erase-during-iteration bug is so common is that the incorrect loop *looks* right at a glance. The fix is subtle: `erase()` returns an iterator to the element that now occupies the erased position, and you need to use that return value instead of incrementing the iterator yourself. The C++20 `std::erase_if` eliminates the pattern entirely and is almost always the right choice when you have access to it.

- **Reallocation invalidation:** When `push_back` exceeds capacity, the vector allocates a new buffer, copies/moves all elements, and frees the old one. Any iterator, pointer, or reference into the old buffer is now dangling.
- **Erase invalidation:** `erase(pos)` shifts elements forward to fill the gap. The iterator at `pos` now refers to what was the next element (or `end()`). Iterators past `pos` are invalidated.
- **Safe patterns:** Use index-based loops when modifying the vector, use the iterator returned by `erase()`, or use C++20's `std::erase`/`std::erase_if`.

### Q2: Show that emplace_back avoids an extra move constructor call compared to push_back with a temporary

The difference between `push_back` and `emplace_back` is small but real. The example below uses an instrumented type so you can see exactly which constructors fire for each approach.

```cpp
#include <iostream>
#include <vector>
#include <string>

struct Widget {
    std::string name;
    int value;

    Widget(std::string n, int v) : name(std::move(n)), value(v) {
        std::cout << "  Constructor: Widget(" << name << ", " << value << ")\n";
    }
    Widget(const Widget& other) : name(other.name), value(other.value) {
        std::cout << "  Copy ctor: Widget(" << name << ")\n";
    }
    Widget(Widget&& other) noexcept : name(std::move(other.name)), value(other.value) {
        std::cout << "  Move ctor: Widget(" << name << ")\n";
    }
};

int main() {
    std::vector<Widget> v;
    v.reserve(4);  // prevent reallocation to isolate the effect

    // === push_back with temporary ===
    std::cout << "push_back(Widget{\"A\", 1}):\n";
    v.push_back(Widget{"A", 1});
    // Output:
    //   Constructor: Widget(A, 1)     <- temporary constructed
    //   Move ctor: Widget(A)          <- moved into vector storage

    // === emplace_back with constructor args ===
    std::cout << "\nemplace_back(\"B\", 2):\n";
    v.emplace_back("B", 2);
    // Output:
    //   Constructor: Widget(B, 2)     <- constructed directly in vector storage
    // No move/copy! One fewer constructor call.

    // === push_back with existing object ===
    std::cout << "\npush_back(existing):\n";
    Widget w{"C", 3};
    v.push_back(std::move(w));
    // Output:
    //   Constructor: Widget(C, 3)     <- w constructed
    //   Move ctor: Widget(C)          <- moved into vector

    // === Summary ===
    std::cout << "\nAll widgets:\n";
    for (const auto& w : v)
        std::cout << "  " << w.name << " = " << w.value << "\n";
    // A = 1
    // B = 2
    // C = 3

    // === When does it matter? ===
    // emplace_back saves 1 move when constructing from args.
    // For cheap-to-move types (string, vector), the difference is tiny.
    // For expensive types (no move ctor, large objects), it matters more.
    // emplace_back can also construct from implicit conversion:
    std::vector<std::string> sv;
    sv.reserve(2);
    sv.push_back("hello");      // constructs temp string, then moves
    sv.emplace_back("world");   // constructs string directly in-place

    return 0;
}
```

Notice the `push_back` case triggers two constructor calls (one to build the temporary, one to move it in), while `emplace_back` triggers exactly one (the object is built directly inside the vector's storage). The saving is exactly one move per insertion. For cheap-to-move types like `std::string`, that difference is negligible. For types with heavy copy/move semantics - or no move constructor at all - it genuinely matters.

- `push_back(Widget{"A", 1})`: Constructs a temporary, then move-constructs it into vector storage. Two constructor calls.
- `emplace_back("B", 2)`: Forwards the arguments to Widget's constructor and builds the object in-place. One constructor call.
- Prefer `emplace_back` when you're constructing from multiple arguments. For inserting an already-existing object, `push_back` and `emplace_back` are equivalent.

### Q3: Demonstrate using reserve before a loop to avoid repeated reallocations

Without `reserve`, every time a vector runs out of capacity it allocates a new buffer (typically 2x bigger), copies all existing elements over, and frees the old one. For N elements that's O(log N) reallocations and O(N log N) total element moves. One `reserve` call collapses all of that to a single allocation.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <string>
#include <cstdlib>

// Track allocations
static int alloc_count = 0;
void* operator new(std::size_t sz) {
    alloc_count++;
    return std::malloc(sz);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

int main() {
    constexpr int N = 100'000;

    // === WITHOUT reserve ===
    alloc_count = 0;
    {
        std::vector<int> v;
        for (int i = 0; i < N; ++i)
            v.push_back(i);
        std::cout << "Without reserve: " << alloc_count << " allocations\n";
        std::cout << "  Final capacity: " << v.capacity() << "\n";
    }
    // Typical output: ~17 allocations (capacity doubles: 1,2,4,8,16,...,131072)
    // Each reallocation copies ALL existing elements!

    // === WITH reserve ===
    alloc_count = 0;
    {
        std::vector<int> v;
        v.reserve(N);  // ONE allocation for the full size
        for (int i = 0; i < N; ++i)
            v.push_back(i);
        std::cout << "\nWith reserve:    " << alloc_count << " allocations\n";
        std::cout << "  Final capacity: " << v.capacity() << "\n";
    }
    // Output: 1 allocation (exactly what we needed)

    // === Performance comparison ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        std::vector<int> v;
        for (int i = 0; i < N; ++i) v.push_back(i);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "\nWithout reserve: " << us << " us\n";
    }
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        std::vector<int> v;
        v.reserve(N);
        for (int i = 0; i < N; ++i) v.push_back(i);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "With reserve:    " << us << " us\n";
    }

    // === reserve vs resize ===
    std::vector<int> r1, r2;
    r1.reserve(10);   // capacity=10, size=0  (no elements created)
    r2.resize(10);    // capacity=10, size=10 (10 zero-initialized ints)

    std::cout << "\nreserve(10): size=" << r1.size() << " cap=" << r1.capacity() << "\n";
    std::cout << "resize(10):  size=" << r2.size() << " cap=" << r2.capacity() << "\n";

    // reserve: use push_back/emplace_back to add elements
    // resize:  elements already exist, access with operator[]

    return 0;
}
```

The allocation counter is the clearest way to see what's happening. Without `reserve` you'll typically see around 17 allocations for 100,000 elements (1, 2, 4, 8, 16, ... doubling up to 131,072). Each of those copies everything accumulated so far. With `reserve` you get exactly one.

The `reserve` vs `resize` distinction is also worth keeping straight. They both make room for N elements, but `resize` actually creates those elements (default-constructing them and setting size to N), while `reserve` only allocates the memory and leaves size at zero. Use `reserve` when you're going to fill the vector with `push_back`; use `resize` when you need immediate random access via `operator[]`.

- **Without reserve:** O(log N) reallocations, O(N log N) total element copies.
- **With reserve:** One allocation, no copies. O(N) total.
- As a rule of thumb: if you know (or can estimate) the final size before a fill loop, always call `reserve` first.

---

## Notes

- **Growth factor:** Most implementations use 2x (GCC, Clang) or 1.5x (MSVC). The standard doesn't mandate a specific factor.
- **`shrink_to_fit()`** is a non-binding request - the implementation may ignore it.
- **`emplace_back` pitfall:** `v.emplace_back(v[0])` can be undefined behavior if the emplace triggers reallocation - the reference to `v[0]` dangles after the old buffer is freed. Copy the value first, or use `push_back`.
- **Prefer `emplace_back`** when constructing from multiple arguments. For single values, `push_back` is equally efficient (and slightly clearer in intent).
- **C++20 `std::erase` / `std::erase_if`:** Replace the erase-remove idiom. `std::erase(v, 3)` removes all elements equal to 3.
- **Contiguous memory guarantee:** `&v[0]` is a valid pointer to a C-style array of `v.size()` elements. You can pass it directly to C APIs.
