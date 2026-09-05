# Use std::cmp_less and std::in_range (C++20) for safe numeric comparisons

**Category:** Safety and Security  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/intcmp>  

---

## Topic Overview

This topic extends the safe integer comparison discussion to focus on **systematically replacing all signed/unsigned comparison warnings** in a codebase and understanding the internal mechanics of how `std::cmp_less` achieves correctness. Where the companion file covers core usage and `std::in_range`, this file emphasizes codebase-wide migration patterns and the underlying algorithm.

The reason it helps to understand the implementation is that it demystifies what the function actually does. Once you see that it is just "check if the signed one is negative first, then compare as unsigned," you will never second-guess the behavior at edge cases.

### How std::cmp_less Works Internally

This simplified implementation shows the exact logic the standard library uses. The `if constexpr` branches let the compiler select the right path at compile time with zero runtime cost for the branches not taken:

```cpp
// Simplified implementation (actual standard library is similar):

template <class T, class U>
constexpr bool cmp_less(T t, U u) noexcept {
    using UT = std::make_unsigned_t<T>;
    using UU = std::make_unsigned_t<U>;

    if constexpr (std::is_signed_v<T> == std::is_signed_v<U>) {
        // Same signedness: normal comparison is correct
        return t < u;
    } else if constexpr (std::is_signed_v<T>) {
        // T is signed, U is unsigned
        // If t < 0, it's definitely less than any unsigned value
        return t < 0 || UT(t) < u;
    } else {
        // T is unsigned, U is signed
        // If u < 0, t (unsigned) is definitely NOT less than u
        return u >= 0 && t < UU(u);
    }
}
```

Notice the key insight: for a signed value `t` and an unsigned value `u`, you never need an unsafe conversion. You just ask "is `t` negative?" - if yes, it is definitely less than any unsigned value. If not, both values are non-negative and you can compare their unsigned representations safely.

### Migration Pattern Summary

If you are working through an existing codebase to eliminate `-Wsign-compare` warnings, this table is your cheat sheet. Most warnings fall into one of these four patterns:

| Before (warning) | After (safe) |
| --- | --- |
| `if (signed_val < unsigned_val)` | `if (std::cmp_less(signed_val, unsigned_val))` |
| `if (int_val >= container.size())` | `if (std::cmp_greater_equal(int_val, container.size()))` |
| `if (a == b)` (mixed sign) | `if (std::cmp_equal(a, b))` |
| `for (int i = 0; i < vec.size(); ++i)` | `for (size_t i = 0; i < vec.size(); ++i)` or `std::cmp_less` |

### Core Example

The classic bug in a single line - `-1 > 0u` evaluates to `true` with raw operators, because `-1` becomes `4294967295` when converted to unsigned:

```cpp
#include <utility>
#include <iostream>

int main() {
    // The classic bug
    int x = -1;
    unsigned y = 0;

    // Raw comparison: -1 > 0u -> true! (because -1 -> 4294967295)
    std::cout << "operator>: " << (x > y) << "\n";  // 1 (true) - WRONG

    // std::cmp_greater: -1 > 0 -> false (correct)
    std::cout << "cmp_greater: " << std::cmp_greater(x, y) << "\n";  // 0 (false) - CORRECT
}
```

---

## Self-Assessment

### Q1: Use `std::cmp_less(-1, 2u)` to safely compare a signed and unsigned value

**Answer:**

Pay attention to the "How it works internally" comment - that is the exact branch taken for a negative signed value vs any unsigned value. No conversion happens at all; the function returns `true` immediately because a negative number is always less than any unsigned number by definition:

```cpp
#include <utility>
#include <iostream>
#include <cstdint>

int main() {
    // The Problem

    int a = -1;
    unsigned b = 2;

    // Implicit conversion: -1 -> 4294967295 (unsigned)
    bool raw_result = (a < b);
    std::cout << "(-1 < 2u) with operator<: " << raw_result << "\n";
    // Output: 0 (false!) - mathematically WRONG

    // The Fix: std::cmp_less

    bool safe_result = std::cmp_less(a, b);
    std::cout << "(-1 < 2u) with cmp_less:  " << safe_result << "\n";
    // Output: 1 (true) - CORRECT

    // How it works internally:
    // std::cmp_less(-1, 2u):
    //   T = int (signed), U = unsigned int (unsigned)
    //   T is signed -> check: t < 0?  -> -1 < 0 -> true
    //   -> return true immediately (any negative < any unsigned)
    //   No conversion needed!

    // Full comparison family

    std::cout << "\nAll safe comparisons with (-1, 2u):\n";
    std::cout << "  cmp_less:          " << std::cmp_less(a, b) << "\n";          // true
    std::cout << "  cmp_greater:       " << std::cmp_greater(a, b) << "\n";       // false
    std::cout << "  cmp_less_equal:    " << std::cmp_less_equal(a, b) << "\n";    // true
    std::cout << "  cmp_greater_equal: " << std::cmp_greater_equal(a, b) << "\n"; // false
    std::cout << "  cmp_equal:         " << std::cmp_equal(a, b) << "\n";         // false
    std::cout << "  cmp_not_equal:     " << std::cmp_not_equal(a, b) << "\n";     // true

    // More edge cases

    std::cout << "\nEdge cases:\n";
    // INT_MAX vs large unsigned
    std::cout << "  cmp_less(INT32_MAX, UINT32_MAX): "
              << std::cmp_less(INT32_MAX, UINT32_MAX) << "\n";  // true

    // 0 vs 0u (should be equal)
    std::cout << "  cmp_equal(0, 0u): "
              << std::cmp_equal(0, 0u) << "\n";  // true

    // Large negative vs 0u
    std::cout << "  cmp_less(INT32_MIN, 0u): "
              << std::cmp_less(INT32_MIN, 0u) << "\n";  // true

    // NOTE: these are NOT templates you instantiate like cmp_less<int, unsigned>
    // They are function templates with deduced template parameters:
    //   std::cmp_less(a, b)  - types deduced from a and b

    // Output:
    // (-1 < 2u) with operator<: 0
    // (-1 < 2u) with cmp_less:  1
    //
    // All safe comparisons with (-1, 2u):
    //   cmp_less:          1
    //   cmp_greater:       0
    //   cmp_less_equal:    1
    //   cmp_greater_equal: 0
    //   cmp_equal:         0
    //   cmp_not_equal:     1
    //
    // Edge cases:
    //   cmp_less(INT32_MAX, UINT32_MAX): 1
    //   cmp_equal(0, 0u): 1
    //   cmp_less(INT32_MIN, 0u): 1
}
```

**Explanation:** `std::cmp_less(-1, 2u)` detects that the first argument is signed and negative. Since any negative integer is mathematically less than any unsigned integer, it returns `true` immediately without any unsigned conversion. This avoids the classic bug where `-1` becomes `4294967295` and appears larger than `2u`.

### Q2: Use `std::in_range<uint8_t>(value)` to check if a value fits in a type before casting

**Answer:**

Protocol serialization is one of the most common places this matters. You receive data as a larger integer type, you want to store it in a compact wire format, and the cast looks harmless until someone sends a value of `-1` or `300`:

```cpp
#include <utility>
#include <iostream>
#include <cstdint>
#include <vector>
#include <algorithm>

// Network protocol: serialize values to wire format

struct WireProtocol {
    // Protocol uses uint8_t fields for compact encoding
    static bool write_age(int user_age, std::vector<uint8_t>& buffer) {
        if (!std::in_range<uint8_t>(user_age)) {
            std::cerr << "Age " << user_age << " cannot be encoded as uint8_t\n";
            return false;
        }
        buffer.push_back(static_cast<uint8_t>(user_age));
        return true;
    }

    static bool write_port(int32_t port, std::vector<uint8_t>& buffer) {
        if (!std::in_range<uint16_t>(port)) {
            std::cerr << "Port " << port << " out of range\n";
            return false;
        }
        uint16_t p = static_cast<uint16_t>(port);
        buffer.push_back(static_cast<uint8_t>(p >> 8));
        buffer.push_back(static_cast<uint8_t>(p & 0xFF));
        return true;
    }
};

// Database: convert query result to application type

template <typename Target>
bool safe_convert(int64_t db_value, Target& out) {
    if (!std::in_range<Target>(db_value)) {
        std::cerr << "Value " << db_value << " doesn't fit in "
                  << sizeof(Target) << "-byte type\n";
        return false;
    }
    out = static_cast<Target>(db_value);
    return true;
}

int main() {
    std::vector<uint8_t> buffer;

    // Valid values
    WireProtocol::write_age(25, buffer);     // OK: 25 fits in uint8_t
    WireProtocol::write_port(8080, buffer);  // OK: 8080 fits in uint16_t

    // Invalid values caught by std::in_range
    WireProtocol::write_age(-5, buffer);     // Error: negative
    WireProtocol::write_age(300, buffer);    // Error: > 255
    WireProtocol::write_port(-1, buffer);    // Error: negative
    WireProtocol::write_port(70000, buffer); // Error: > 65535

    std::cout << "\nBuffer size: " << buffer.size() << " bytes\n";

    // Database conversion
    int16_t small_val;
    safe_convert<int16_t>(32000, small_val);     // OK
    safe_convert<int16_t>(40000, small_val);     // Error: > 32767
    safe_convert<int16_t>(-40000LL, small_val);  // Error: < -32768

    // Output:
    // Age -5 cannot be encoded as uint8_t
    // Age 300 cannot be encoded as uint8_t
    // Port -1 out of range
    // Port 70000 out of range
    //
    // Buffer size: 3 bytes
    //
    // Value 40000 doesn't fit in 2-byte type
    // Value -40000 doesn't fit in 2-byte type
}
```

**Explanation:** `std::in_range<uint8_t>(value)` is a compile-time or runtime check that verifies `value` fits in `uint8_t` (0-255) regardless of the source type's signedness. It's the correct guard before any narrowing `static_cast`. In protocol code and database interop, this prevents silent truncation that could corrupt data or create security vulnerabilities.

### Q3: Replace all signed/unsigned comparison warnings in a codebase with `std::cmp_*` functions

**Answer:**

The migration strategy below is the systematic way to work through a real codebase. The grep/clang-tidy commands at the end of the code block are what you run first to find everything that needs changing:

```cpp
#include <utility>
#include <iostream>
#include <vector>
#include <string>
#include <cstdint>
#include <algorithm>

// Systematic replacement patterns

// PATTERN 1: Loop with size_t
// Before:
// for (int i = 0; i < vec.size(); ++i) { ... }  // WARNING: signed/unsigned
// Fix option A (prefer): use size_t
// for (size_t i = 0; i < vec.size(); ++i) { ... }
// Fix option B: use range-for
// for (auto& elem : vec) { ... }
// Fix option C: cmp_less (when index comes from external source)
// for (int i = 0; std::cmp_less(i, vec.size()); ++i) { ... }

// PATTERN 2: Bounds check with signed index
struct SafeContainer {
    std::vector<int> data;

    // Before:
    // bool valid(int idx) { return idx >= 0 && idx < data.size(); }  // WARNING

    // After:
    bool valid(int idx) const {
        return std::cmp_greater_equal(idx, 0) &&
               std::cmp_less(idx, data.size());
    }

    int at(int idx) const {
        if (!valid(idx)) throw std::out_of_range("bad index");
        return data[static_cast<size_t>(idx)];
    }
};

// PATTERN 3: Function parameter validation
// Before:
// void process(int count, unsigned max) {
//     if (count > max) return;  // WARNING + BUG when count is negative
// }

void process(int count, unsigned max) {
    if (std::cmp_greater(count, max)) return;
    std::cout << "Processing " << count << " items (max: " << max << ")\n";
}

// PATTERN 4: std::find with signed offset
void find_element(const std::vector<int>& vec, int target, int start_offset) {
    if (!std::cmp_less(start_offset, vec.size()) ||
        std::cmp_less(start_offset, 0)) {
        std::cerr << "Invalid start offset\n";
        return;
    }
    auto it = std::find(vec.begin() + start_offset, vec.end(), target);
    if (it != vec.end()) {
        std::cout << "Found at index " << (it - vec.begin()) << "\n";
    }
}

// grep patterns to find warnings
//
// Find all signed/unsigned comparisons in codebase:
//   g++ -std=c++20 -Wall -Wsign-compare -Werror src/*.cpp 2>&1 | grep sign-compare
//
// Or use clang-tidy:
//   clang-tidy -checks='bugprone-signed-char-misuse,
//               bugprone-narrowing-conversions,
//               misc-misplaced-sign' src/*.cpp
//
// Automated replacement (not fully automatic - needs review):
//   Replace: if (a < b)     ->  if (std::cmp_less(a, b))
//   When:    a and b have different signedness
//   Add:     #include <utility>  at file top

int main() {
    SafeContainer sc{{10, 20, 30, 40, 50}};

    std::cout << "sc.at(2) = " << sc.at(2) << "\n";
    std::cout << "sc.valid(-1) = " << std::boolalpha << sc.valid(-1) << "\n";
    std::cout << "sc.valid(100) = " << sc.valid(100) << "\n";

    process(-5, 100);   // doesn't process (-5 is not > 100, so it proceeds)
    process(200, 100);  // skipped (200 > 100)

    std::vector<int> data{1, 2, 3, 4, 5};
    find_element(data, 3, 0);
    find_element(data, 3, -1);

    // Output:
    // sc.at(2) = 30
    // sc.valid(-1) = false
    // sc.valid(100) = false
    // Processing -5 items (max: 100)
    // Found at index 2
    // Invalid start offset
}
```

**Explanation:** The migration strategy: (1) Compile with `-Wsign-compare -Werror` to find all warnings. (2) For loop indices, prefer `size_t` or range-for. (3) For bounds checks receiving external signed values, use `std::cmp_less`/`std::cmp_greater_equal`. (4) For equality checks across signedness, use `std::cmp_equal`. (5) Add `#include <utility>` where needed. Each replacement is a one-line change with zero runtime cost.

---

## Notes

- **`std::cmp_*` only works with integer types.** Passing `float` or `double` is a compile error.
- **Zero overhead:** The generated assembly is identical to hand-written correct comparisons (early-out on negative check + unsigned compare).
- **Pre-C++20 alternative:** Write a `safe_less(T a, U b)` helper with the same logic, or use `gsl::narrow_cast` / `gsl::narrow`.
- **`std::ssize(container)`** (C++20) returns a signed size (`ptrdiff_t`), which can eliminate some sign-compare warnings when paired with a signed loop variable.
- **`ranges::ssize`** works on any range, not just containers.
- Compile with `-std=c++20 -Wall -Wextra -Wsign-compare -Werror`.
