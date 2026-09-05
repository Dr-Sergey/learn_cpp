# Use std::ssize (C++20) to get a signed size from a container

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/iterator/size>  

---

## Topic Overview

`std::ssize()` (C++20, header `<iterator>`) returns the size of a container or range as a **signed** integer type. It solves a pervasive and easy-to-miss problem: `container.size()` returns `size_t`, which is unsigned, and mixing unsigned arithmetic with signed integers leads to subtle bugs and compiler warnings that many people just silence instead of fixing.

### The Problem

The reason this trips people up is that unsigned arithmetic wraps around. A subtraction that would naturally go negative instead becomes a huge positive number:

```cpp
std::vector<int> v = {1, 2, 3};

// WARNING: comparison between signed and unsigned integer
for (int i = 0; i < v.size(); ++i) { }  // -Wsign-compare

// BUG: unsigned underflow when subtracting from size()
if (v.size() - 1 >= 0) { }  // Always true! size_t can't be negative
// When v is empty: v.size() - 1 = (size_t)0 - 1 = 18446744073709551615
```

That `>= 0` check looks reasonable, but it's comparing an unsigned value to zero - it can never be false. This is the kind of bug that only appears with empty containers and can sit in code for a long time unnoticed.

### The Solution

```cpp
#include <iterator> // std::ssize

std::vector<int> v = {1, 2, 3};

// No warning: both sides are signed
for (int i = 0; i < std::ssize(v); ++i) { }  // clean

// Safe subtraction: signed result can be negative
auto last_idx = std::ssize(v) - 1;  // ptrdiff_t, can be -1 for empty
```

### Return Type

```cpp
// std::ssize(c) returns:
// std::common_type_t<std::ptrdiff_t, std::make_signed_t<decltype(c.size())>>
//
// In practice: ptrdiff_t (typically int64_t on 64-bit systems)
// This is large enough to represent any valid container size as signed
```

### Core Examples

Here's `ssize` in action across several container types. Notice that the empty-container case is handled safely - `ssize` returns -1 for `ssize(empty) - 1`, which immediately fails the `>= 0` check.

```cpp
#include <iterator>
#include <vector>
#include <array>
#include <iostream>

int main() {
    std::vector<int> v = {10, 20, 30, 40, 50};

    // Basic usage — signed loop index, no warning
    for (auto i = 0; i < std::ssize(v); ++i)
        std::cout << v[i] << " "; // 10 20 30 40 50
    std::cout << "\n";

    // Works with C arrays
    int arr[] = {1, 2, 3};
    std::cout << "array ssize: " << std::ssize(arr) << "\n"; // 3

    // Works with std::array
    std::array<double, 5> a{};
    std::cout << "std::array ssize: " << std::ssize(a) << "\n"; // 5

    // Safe arithmetic
    std::vector<int> empty;
    auto last = std::ssize(empty) - 1; // -1, not 18446744073709551615
    std::cout << "empty last index: " << last << "\n"; // -1
    if (last >= 0) {
        std::cout << "has elements\n";
    } else {
        std::cout << "empty\n"; // prints "empty"
    }
}
```

---

## Self-Assessment

### Q1: Show a signed/unsigned comparison warning fixed by using `std::ssize(vec)` instead of `vec.size()`

**Answer:**

```cpp
#include <vector>
#include <iostream>
#include <iterator>

int main() {
    std::vector<int> data = {10, 20, 30, 40, 50};

    // WARNING with vec.size():
    // "comparison of integers of different signs: 'int' and 'size_t'"
    int target_index = 3;
    // if (target_index < data.size()) { }  // -Wsign-compare warning
    // The compiler warns because 'int' (signed) is compared with 'size_t' (unsigned)
    // If target_index were negative (-1), it would be converted to a huge unsigned
    // value, making the comparison give wrong results!

    // FIXED with std::ssize():
    if (target_index < std::ssize(data)) {
        std::cout << "data[" << target_index << "] = " << data[target_index] << "\n";
    }
    // No warning — both sides are signed

    // The bug scenario without ssize:
    int bad_index = -1;
    // if (bad_index < data.size()) -> -1 converted to unsigned -> 18446... -> false!
    // The element at "index -1" would be accessed -> UB

    if (bad_index < std::ssize(data)) {
        std::cout << "Valid\n";
    } else {
        std::cout << "Invalid index\n";
    }
    // Output: Valid (correct! -1 < 5 is true in signed arithmetic)

    // Loop without warnings:
    for (int i = 0; i < std::ssize(data); ++i) {
        std::cout << data[i] << " ";
    }
    std::cout << "\n";
    // Output: 10 20 30 40 50
}
```

`data.size()` returns `size_t` (unsigned). Comparing `int` with `size_t` triggers `-Wsign-compare` because negative `int` values silently convert to large unsigned values, causing logic bugs. `std::ssize(data)` returns a signed type (`ptrdiff_t`), making the comparison type-safe and warning-free.

### Q2: Use ssize in a loop that counts down from the last index without underflow bugs

**Answer:**

```cpp
#include <vector>
#include <iostream>
#include <iterator>

int main() {
    std::vector<int> data = {10, 20, 30, 40, 50};

    // BUG with size(): unsigned underflow on empty container
    // for (size_t i = data.size() - 1; i >= 0; --i) { }
    // Problem 1: When data is empty, data.size() - 1 = (size_t)-1 = HUGE
    // Problem 2: i >= 0 is ALWAYS TRUE for unsigned i — infinite loop!

    // CORRECT with ssize(): signed arithmetic, handles empty safely
    for (auto i = std::ssize(data) - 1; i >= 0; --i) {
        std::cout << data[i] << " ";
    }
    std::cout << "\n";
    // Output: 50 40 30 20 10

    // Empty container — loop body never executes
    std::vector<int> empty;
    for (auto i = std::ssize(empty) - 1; i >= 0; --i) {
        std::cout << "BUG! This should never print\n";
    }
    std::cout << "Empty handled correctly\n";
    // Output: Empty handled correctly
    // ssize(empty) - 1 = -1, which is < 0, so loop doesn't execute

    // Practical: reverse search
    std::vector<std::string> names = {"Alice", "Bob", "Charlie", "Bob", "Diana"};
    std::string target = "Bob";

    // Find LAST occurrence by iterating backwards
    for (auto i = std::ssize(names) - 1; i >= 0; --i) {
        if (names[i] == target) {
            std::cout << "Last '" << target << "' at index " << i << "\n";
            break;
        }
    }
    // Output: Last 'Bob' at index 3
}
```

The countdown loop `for (auto i = ssize(v)-1; i >= 0; --i)` works correctly because `ssize` returns a signed type. When the container is empty, `ssize(v) - 1 == -1`, which fails the `i >= 0` condition immediately. With unsigned `size()`, the subtraction would wrap to `SIZE_MAX`, creating an infinite loop.

### Q3: Explain why the standard library added ssize instead of changing size() to return signed

The committee chose to add `std::ssize()` as a free function rather than changing `container::size()` to return a signed type for several reasons:

**1. ABI compatibility:**
Changing `size()` return type from `size_t` to `ptrdiff_t` would break every binary compiled against the old ABI. Existing shared libraries, system libraries, and third-party code would all need recompilation.

**2. API compatibility:**
Millions of lines of code depend on `size()` returning `size_t`. Changing it would break code that:

- Assigns `size()` to a `size_t` variable
- Passes `size()` to functions expecting `size_t`
- Uses `size()` in templates deducing `size_t`

**3. Allocator interface:**
`std::allocator::size_type` is `size_t`. The entire memory model uses unsigned sizes - `new`, `malloc`, `memcpy` all take unsigned sizes. Container `size()` returning unsigned is consistent with this.

**4. Range considerations:**
Containers can theoretically hold up to `SIZE_MAX` elements on platforms where `sizeof(ptrdiff_t) == sizeof(size_t)`. A signed return would halve the theoretical maximum size.

**5. Free function is non-breaking:**
`std::ssize()` gives developers the signed-size option without changing any existing types or breaking any code. It's an additive, backward-compatible solution.

```cpp
#include <iterator>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v(10);

    // Both exist and serve different purposes:
    auto unsigned_size = v.size();    // size_t — for allocation, indexing
    auto signed_size = std::ssize(v); // ptrdiff_t — for arithmetic, comparison

    static_assert(std::is_unsigned_v<decltype(unsigned_size)>);
    static_assert(std::is_signed_v<decltype(signed_size)>);

    std::cout << "size_t: " << sizeof(unsigned_size) << " bytes, unsigned\n";
    std::cout << "ssize:  " << sizeof(signed_size) << " bytes, signed\n";
    // Both are 8 bytes on 64-bit (size_t and ptrdiff_t)
}
```

---

## Notes

- **Header:** `std::ssize` is in `<iterator>`, not `<cstddef>`. Many standard headers transitively include it.
- **C-style arrays:** `std::ssize` works on built-in arrays: `int arr[5]; std::ssize(arr) == 5`.
- **`ranges::ssize`:** C++20 also provides `std::ranges::ssize(r)` which works with any range.
- **Cost:** Zero runtime cost - `ssize` is a `constexpr` function that simply casts `size()`.
- **Guideline (C++ Core Guidelines ES.107):** "Don't use unsigned for subscripts, prefer `gsl::index`" - `ssize()` aligns with this recommendation.
- **Not for sizes > `PTRDIFF_MAX`:** If a container somehow has more than `PTRDIFF_MAX` elements (>9.2 quintillion on 64-bit), `ssize()` has UB. This is not a realistic concern.
- Compile with `-std=c++20 -Wall -Wextra`.
