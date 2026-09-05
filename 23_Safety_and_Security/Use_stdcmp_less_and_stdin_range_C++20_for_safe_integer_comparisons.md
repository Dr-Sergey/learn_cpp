# Use std::cmp_less and std::in_range (C++20) for safe integer comparisons

**Category:** Safety and Security  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/intcmp>  

---

## Topic Overview

C++20 introduced `std::cmp_less`, `std::cmp_greater`, `std::cmp_equal`, etc., and `std::in_range` in `<utility>` to safely compare integers of different signedness. Before C++20, comparing `signed` and `unsigned` integers gave mathematically wrong results - `-1 < 1u` evaluates to `false` because `-1` is implicitly converted to a huge unsigned number.

The reason this trips people up is that the compiler warning for mixed-signedness comparisons is easy to silence with a cast - but the cast is often wrong too. `std::cmp_less` is the correct fix: no cast, no warning, and the right mathematical result.

### The Problem

The core issue is C++'s implicit conversion rules. When you mix signed and unsigned in a comparison, the signed value is converted to unsigned before the comparison happens. Negative numbers become enormous positive numbers:

```cpp
int x = -1;
unsigned y = 1;

x < y  -> false!   (x is converted to unsigned: -1 -> 4294967295, 4294967295 < 1 -> false)
x > y  -> true!    (4294967295 > 1 -> true, but -1 is NOT greater than 1!)
x == y -> false    (correct by luck, but for different values could be wrong)
```

### C++20 Safe Comparison Functions

All six comparison functions live in `<utility>` and work with any integer types, including mixing signed and unsigned freely. `std::in_range` is the companion function for checking whether a value fits in a target type before casting:

| Function | Equivalent | Safe? |
| --- | --- | --- |
| `std::cmp_less(a, b)` | `a < b` (mathematically) | Yes |
| `std::cmp_greater(a, b)` | `a > b` (mathematically) | Yes |
| `std::cmp_less_equal(a, b)` | `a <= b` | Yes |
| `std::cmp_greater_equal(a, b)` | `a >= b` | Yes |
| `std::cmp_equal(a, b)` | `a == b` | Yes |
| `std::cmp_not_equal(a, b)` | `a != b` | Yes |
| `std::in_range<T>(value)` | `value in [T_min, T_max]` | Yes |

### Core Example

Watch what happens to the same comparison done two ways - raw `operator<` vs `std::cmp_less`. Same values, opposite results:

```cpp
#include <utility>
#include <iostream>
#include <cstdint>

int main() {
    int signed_val = -1;
    unsigned unsigned_val = 1;

    // Broken: implicit conversion makes -1 into 4294967295
    std::cout << "(-1 < 1u) with operator<: " << (signed_val < unsigned_val) << "\n";
    // Output: 0 (false!) - WRONG

    // Fixed: std::cmp_less handles mixed signedness correctly
    std::cout << "(-1 < 1u) with cmp_less:  " << std::cmp_less(signed_val, unsigned_val) << "\n";
    // Output: 1 (true) - CORRECT
}
```

---

## Self-Assessment

### Q1: Replace a signed/unsigned comparison warning with std::cmp_less

**Answer:**

The classic example is checking a signed index against a container's `size()`, which returns `size_t`. With a negative index, the raw comparison silently fails to catch the invalid index - `std::cmp_less` + `std::cmp_greater_equal` catches it correctly:

```cpp
#include <utility>
#include <iostream>
#include <vector>
#include <cstdint>

// Before: compiler warning
//
// void find_index_buggy(const std::vector<int>& vec, int target_index) {
//     // WARNING: comparison of integer expressions of different signedness:
//     //          'int' and 'std::vector<int>::size_type' [-Wsign-compare]
//     if (target_index < vec.size()) {   // BUG if target_index is negative!
//         std::cout << vec[target_index] << "\n";
//     }
//     // When target_index = -1:
//     //   -1 < vec.size() -> false (unsigned conversion)
//     //   -> validation BYPASSED -> no access (safe by accident here)
//     //   BUT: if used as index elsewhere -> UB
// }

// After: std::cmp_less

void find_index_safe(const std::vector<int>& vec, int target_index) {
    // std::cmp_less handles: int vs size_t safely
    if (std::cmp_less(target_index, vec.size()) &&
        std::cmp_greater_equal(target_index, 0)) {
        std::cout << "vec[" << target_index << "] = "
                  << vec[static_cast<size_t>(target_index)] << "\n";
    } else {
        std::cout << "Index " << target_index << " is out of range\n";
    }
}

// More examples

void compare_packet_length(int32_t user_length, uint32_t max_length) {
    // BEFORE (warning + potential bug):
    // if (user_length > max_length) { ... }
    // When user_length = -1: -1 > max_length -> true (unsigned conversion)

    // AFTER:
    if (std::cmp_greater(user_length, max_length)) {
        std::cout << "Length exceeds maximum\n";
    } else {
        std::cout << "Length OK\n";
    }
}

void bounds_check(int64_t offset, uint32_t buffer_size) {
    // Safe combined check: 0 <= offset < buffer_size
    if (std::cmp_greater_equal(offset, 0) &&
        std::cmp_less(offset, buffer_size)) {
        std::cout << "Offset " << offset << " is valid\n";
    } else {
        std::cout << "Offset " << offset << " is INVALID\n";
    }
}

int main() {
    std::vector<int> vec{10, 20, 30, 40, 50};

    find_index_safe(vec, 2);     // vec[2] = 30
    find_index_safe(vec, -1);    // Index -1 is out of range
    find_index_safe(vec, 100);   // Index 100 is out of range

    compare_packet_length(-1, 1024);  // Length OK (correctly: -1 < 1024)
    compare_packet_length(2000, 1024); // Length exceeds maximum

    bounds_check(-5, 100);    // Offset -5 is INVALID
    bounds_check(50, 100);    // Offset 50 is valid
    bounds_check(200, 100);   // Offset 200 is INVALID

    // Output:
    // vec[2] = 30
    // Index -1 is out of range
    // Index 100 is out of range
    // Length OK
    // Length exceeds maximum
    // Offset -5 is INVALID
    // Offset 50 is valid
    // Offset 200 is INVALID
}
```

**Explanation:** `std::cmp_less(a, b)` performs a mathematically correct comparison regardless of signedness differences. When comparing `int` `-1` with `size_t` `5`, it correctly determines `-1 < 5` is true. The raw operator `<` would convert `-1` to `4294967295` and report `false`. No warnings, no implicit conversions, mathematically correct results.

### Q2: Use `std::in_range<int>(value)` to check if a value fits before a narrowing cast

**Answer:**

`std::in_range<T>` is the correct guard before any narrowing `static_cast`. The important thing it adds over a manual range check is that it handles signedness correctly - `std::in_range<uint8_t>(-1)` returns `false`, whereas a naive `value >= 0 && value <= 255` would first convert `-1` to unsigned and get the wrong answer:

```cpp
#include <utility>
#include <iostream>
#include <cstdint>
#include <optional>

// Without std::in_range (dangerous)

uint8_t unsafe_narrow(int64_t value) {
    return static_cast<uint8_t>(value);  // silently truncates!
    // 256 -> 0, -1 -> 255, 100000 -> 160
}

// With std::in_range (safe)

std::optional<uint8_t> safe_narrow(int64_t value) {
    if (std::in_range<uint8_t>(value)) {
        // Value is guaranteed to fit: 0 <= value <= 255
        return static_cast<uint8_t>(value);
    }
    return std::nullopt;  // value doesn't fit
}

// Real-world use cases

// Network protocol: convert parsed length to uint16_t
bool validate_packet_length(int32_t parsed_length) {
    if (!std::in_range<uint16_t>(parsed_length)) {
        std::cerr << "Invalid length: " << parsed_length
                  << " (must be 0-65535)\n";
        return false;
    }
    uint16_t length = static_cast<uint16_t>(parsed_length);
    std::cout << "Valid packet length: " << length << "\n";
    return true;
}

// Safe array index from user input
void safe_index_access(const int* arr, size_t arr_size, int64_t user_index) {
    if (!std::in_range<size_t>(user_index)) {
        std::cerr << "Index out of size_t range\n";
        return;
    }
    auto index = static_cast<size_t>(user_index);
    if (index >= arr_size) {
        std::cerr << "Index " << index << " >= array size " << arr_size << "\n";
        return;
    }
    std::cout << "arr[" << index << "] = " << arr[index] << "\n";
}

int main() {
    // std::in_range examples
    std::cout << std::boolalpha;
    std::cout << "in_range<uint8_t>(255):    " << std::in_range<uint8_t>(255) << "\n";    // true
    std::cout << "in_range<uint8_t>(256):    " << std::in_range<uint8_t>(256) << "\n";    // false
    std::cout << "in_range<uint8_t>(-1):     " << std::in_range<uint8_t>(-1) << "\n";     // false
    std::cout << "in_range<int8_t>(127):     " << std::in_range<int8_t>(127) << "\n";     // true
    std::cout << "in_range<int8_t>(128):     " << std::in_range<int8_t>(128) << "\n";     // false
    std::cout << "in_range<size_t>(-1LL):    " << std::in_range<size_t>(-1LL) << "\n";    // false

    std::cout << "\n";

    // Safe narrowing
    auto result = safe_narrow(200);
    if (result) std::cout << "Narrowed to: " << static_cast<int>(*result) << "\n";

    auto bad = safe_narrow(300);
    if (!bad) std::cout << "300 doesn't fit in uint8_t\n";

    auto neg = safe_narrow(-5);
    if (!neg) std::cout << "-5 doesn't fit in uint8_t\n";

    // Output:
    // in_range<uint8_t>(255):    true
    // in_range<uint8_t>(256):    false
    // in_range<uint8_t>(-1):     false
    // in_range<int8_t>(127):     true
    // in_range<int8_t>(128):     false
    // in_range<size_t>(-1LL):    false
    //
    // Narrowed to: 200
    // 300 doesn't fit in uint8_t
    // -5 doesn't fit in uint8_t
}
```

**Explanation:** `std::in_range<T>(value)` returns `true` if `value` can be stored in type `T` without changing its mathematical value. It handles signed/unsigned cross-checks correctly - `std::in_range<uint8_t>(-1)` returns `false` even though `static_cast<uint8_t>(-1)` would silently produce 255. Always check with `in_range` before narrowing casts to prevent silent truncation bugs.

### Q3: Show a silent truncation bug caught by std::in_range that a cast would have silenced

**Answer:**

The allocator example below illustrates a real class of heap overflow vulnerability. When the user requests 65537 bytes but only 1 byte gets allocated because of silent truncation, the resulting heap buffer is far too small for whatever the caller writes into it:

```cpp
#include <utility>
#include <iostream>
#include <cstdint>
#include <vector>
#include <cassert>
#include <algorithm>

// The Bug: Silent truncation in memory allocation

struct BuggyAllocator {
    // User requests allocation with a 64-bit size
    uint8_t* allocate(uint64_t requested_size) {
        // BUG: truncate to uint16_t for "small buffer optimization"
        uint16_t actual_size = static_cast<uint16_t>(requested_size);
        // If requested_size = 0x10001 (65537):
        //   actual_size = 1 (truncated!)
        //   Allocates 1 byte, user writes 65537 bytes -> HEAP OVERFLOW

        auto* buf = new uint8_t[actual_size];
        std::cout << "[BUGGY] Requested: " << requested_size
                  << ", allocated: " << actual_size << "\n";
        return buf;
    }
};

// The Fix: std::in_range catches the truncation

struct SafeAllocator {
    uint8_t* allocate(uint64_t requested_size) {
        if (!std::in_range<uint16_t>(requested_size)) {
            std::cerr << "[SAFE] Rejected: " << requested_size
                      << " exceeds uint16_t max (65535)\n";
            return nullptr;  // or throw, or use full 64-bit allocation
        }

        uint16_t actual_size = static_cast<uint16_t>(requested_size);
        auto* buf = new uint8_t[actual_size];
        std::cout << "[SAFE] Allocated: " << actual_size << " bytes\n";
        return buf;
    }
};

// More silent truncation bugs

void truncation_in_loop_counter() {
    uint32_t total_items = 100'000;

    // BUG: loop counter is uint16_t (max 65535)
    // for (uint16_t i = 0; i < total_items; ++i) { ... }
    // i wraps around at 65536 -> infinite loop!

    // Detection with std::in_range:
    if (!std::in_range<uint16_t>(total_items)) {
        std::cerr << "Loop counter would overflow: "
                  << total_items << " > 65535\n";
    }
}

void truncation_in_protocol() {
    // Network protocol: server sends file size as int64_t
    int64_t file_size = 5'000'000'000LL;  // 5 GB

    // Client stores as int32_t (max ~2.1 GB)
    // int32_t local_size = static_cast<int32_t>(file_size);
    // local_size = 705032704 (truncated!) -> wrong number of bytes downloaded

    // Detection:
    if (!std::in_range<int32_t>(file_size)) {
        std::cerr << "File size " << file_size
                  << " exceeds int32_t range\n";
    }
}

int main() {
    // Demonstrate the bug
    BuggyAllocator buggy;
    auto* p1 = buggy.allocate(65537);  // allocates only 1 byte!
    delete[] p1;

    auto* p2 = buggy.allocate(200);    // allocates 200 - correct
    delete[] p2;

    std::cout << "\n";

    // Demonstrate the fix
    SafeAllocator safe;
    auto* p3 = safe.allocate(65537);   // rejected!
    // p3 is nullptr

    auto* p4 = safe.allocate(200);     // allocated: 200 bytes
    delete[] p4;

    std::cout << "\n";

    truncation_in_loop_counter();
    truncation_in_protocol();

    // Output:
    // [BUGGY] Requested: 65537, allocated: 1
    // [BUGGY] Requested: 200, allocated: 200
    //
    // [SAFE] Rejected: 65537 exceeds uint16_t max (65535)
    // [SAFE] Allocated: 200 bytes
    //
    // Loop counter would overflow: 100000 > 65535
    // File size 5000000000 exceeds int32_t range
}
```

**Explanation:** `static_cast<uint16_t>(65537)` silently produces `1` (truncation). The compiler doesn't warn because you explicitly asked for the cast. `std::in_range<uint16_t>(65537)` returns `false`, telling you the value doesn't fit. This catches real security bugs: allocating too-small buffers (heap overflow), loop counter wraps (infinite loops or missed iterations), and protocol field truncation (wrong file sizes).

---

## Notes

- **`std::cmp_*` functions** work only with integer types (not floating-point). They are in `<utility>`.
- **No runtime overhead:** `std::cmp_less` and `std::in_range` compile to a few compare/branch instructions - the same work the programmer would write manually with correct casts.
- **`-Wsign-compare`** detects the problem; `std::cmp_less` fixes it without suppressing the warning.
- **GSL alternative:** `gsl::narrow<T>(value)` throws `gsl::narrowing_error` on truncation. `std::in_range` + `static_cast` achieves the same without GSL.
- **`std::cmp_less` is NOT a template class** - it's a function template: `std::cmp_less(a, b)`, not `std::cmp_less<int,size_t>(a, b)`.
- Compile with `-std=c++20 -Wall -Wextra -Wsign-compare`.
