# Use std::cmp_less, std::in_range, and safe integer comparison (C++20)

**Category:** Safety and Security  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/intcmp>  

---

## Topic Overview

This topic focuses on **practical gotchas** in integer comparison that bite even experienced developers - especially the `vec.size() - 1` underflow trap and the dangers of casting `size_t` to `int`. While companion files cover `std::cmp_less` and `std::in_range` basics, this file emphasizes the real-world bugs these functions prevent.

The reason these bugs are so insidious is that they look completely correct at first glance. `vec.size() - 1` seems like a perfectly reasonable way to get the last index. The fact that it wraps to an astronomically large number on an empty vector is not at all obvious from reading the code. These are the bugs that pass code review and show up in production.

### The size_t Subtraction Trap

This is one of the most common unsigned integer bugs in C++. The result is not a small negative number - it is the largest possible value for a 64-bit unsigned integer:

```cpp
std::vector<int> vec;  // empty: vec.size() == 0

vec.size() - 1  ->  0uz - 1  ->  18446744073709551615  (SIZE_MAX!)

if (vec.size() - 1 > idx)  ->  always true for any idx < SIZE_MAX
->  Off-by-one logic -> potential buffer overread
```

### Common Dangerous Patterns

Each pattern in this table represents a class of bug. The `(int)vec.size()` cast looks harmless because containers rarely have billions of elements - but for byte buffers they can, and on those platforms the cast produces a negative count:

| Pattern | Bug | Safe alternative |
| --- | --- | --- |
| `(int)vec.size()` | Truncation if size > INT_MAX | `std::cmp_less(idx, vec.size())` |
| `vec.size() - 1` | Underflow on empty vector | `vec.empty() ? 0 : vec.size() - 1` or `std::cmp_*` |
| `int i = vec.size()` | Implicit narrowing | `std::in_range<int>(vec.size())` check first |
| `for (int i = vec.size()-1; i >= 0; --i)` | UB if size > INT_MAX | `for (auto i = vec.size(); i-- > 0;)` |

### Core Example

This shows the underflow in action. The output is not a small wrong number - it is `SIZE_MAX`, which is why any subsequent comparison using this value will be wrong:

```cpp
#include <utility>
#include <iostream>
#include <vector>

int main() {
    std::vector<int> empty_vec;
    size_t last_index = empty_vec.size() - 1;
    // last_index = 18446744073709551615 (SIZE_MAX!)

    std::cout << "empty_vec.size() - 1 = " << last_index << "\n";
    // Output: empty_vec.size() - 1 = 18446744073709551615

    // Safe check:
    if (!empty_vec.empty()) {
        size_t safe_last = empty_vec.size() - 1;
        std::cout << "Last index: " << safe_last << "\n";
    }
}
```

---

## Self-Assessment

### Q1: Replace `(int)size < n` with `std::cmp_less(size, n)` to eliminate signed/unsigned comparison issues

**Answer:**

The dangerous cast `(int)data.size()` is the kind of thing that works fine in tests with small data sets and fails silently in production when the container grows large. `std::cmp_less` avoids the cast entirely:

```cpp
#include <utility>
#include <iostream>
#include <vector>
#include <cstdint>
#include <string>

// The Dangerous Cast

void search_buggy(const std::vector<std::string>& data, int max_results) {
    // BUGGY: casting size_t to int
    int count = (int)data.size();
    // If data.size() > INT_MAX (2^31 - 1 = 2,147,483,647):
    //   count becomes negative (undefined behavior, or wraps to negative)
    //   All subsequent comparisons with count are wrong

    if (count < max_results) {
        std::cout << "All " << count << " results returned\n";
    }
}

// The Safe Version

void search_safe(const std::vector<std::string>& data, int max_results) {
    // No cast! Compare directly with std::cmp_less
    if (std::cmp_less(data.size(), max_results)) {
        std::cout << "All " << data.size() << " results returned\n";
    } else if (std::cmp_greater(max_results, 0)) {
        std::cout << "Returning first " << max_results << " of "
                  << data.size() << " results\n";
    }
}

// More examples

// API function: buffer size comes as int (legacy C API)
void legacy_api_wrapper(const char* data, size_t data_len, int buffer_size) {
    // BUGGY:
    // if ((int)data_len > buffer_size) { ... }  // truncation if data_len > INT_MAX

    // SAFE:
    if (std::cmp_greater(data_len, buffer_size)) {
        std::cerr << "Data too large for buffer\n";
        return;
    }
    // Now safe to use buffer_size as limit
    std::cout << "Processing " << data_len << " bytes into "
              << buffer_size << "-byte buffer\n";
}

// Loop with external signed count
void process_items(const std::vector<int>& items, int32_t requested_count) {
    // BUGGY: for (int i = 0; i < items.size(); ++i)
    // Warning + wrong if items.size() doesn't fit in int

    // SAFE option 1: std::cmp_less in condition
    for (int32_t i = 0; std::cmp_less(i, items.size()) &&
                          std::cmp_less(i, requested_count); ++i) {
        std::cout << items[static_cast<size_t>(i)] << " ";
    }
    std::cout << "\n";

    // SAFE option 2: compute limit first using std::min with matching types
    // size_t limit = std::in_range<size_t>(requested_count) && requested_count >= 0
    //              ? std::min(static_cast<size_t>(requested_count), items.size())
    //              : 0;
}

int main() {
    std::vector<std::string> data{"a", "b", "c", "d", "e"};

    search_safe(data, 3);    // Returning first 3 of 5 results
    search_safe(data, 10);   // All 5 results returned

    legacy_api_wrapper("hello", 5, 1024);  // Processing 5 bytes
    legacy_api_wrapper("hello", 5, 3);     // Processing 5 bytes (bug here - should check!)

    std::vector<int> nums{10, 20, 30, 40, 50};
    process_items(nums, 3);   // 10 20 30
    process_items(nums, -1);  // (nothing - -1 < 0)

    // Output:
    // Returning first 3 of 5 results
    // All 5 results returned
    // Processing 5 bytes into 1024-byte buffer
    // Processing 5 bytes into 3-byte buffer
    // 10 20 30
}
```

**Explanation:** Casting `size_t` to `int` with `(int)data.size()` silently truncates if the container has more than 2^31 elements (rare but possible for byte buffers). `std::cmp_less(data.size(), max_results)` compares them correctly without any cast - if `max_results` is negative, it correctly determines it's less than any `size_t` value.

### Q2: Use `std::in_range<int>(value)` to check if a value fits in int before narrowing

**Answer:**

C API boundaries are the most common place where you receive a `size_t` but need to pass an `int`. The `safe_send` function below shows the right pattern - check with `std::in_range<int>` first, then cast only if it fits:

```cpp
#include <utility>
#include <iostream>
#include <cstdint>

// Scenario: Receiving size_t from STL, need int for C API

// Many C APIs expect int for sizes (POSIX read, write, recv, send)
// #include <unistd.h>
// ssize_t read(int fd, void *buf, size_t count);
// -> count is size_t, but return value is ssize_t (signed)

int safe_send(const void* data, size_t length) {
    // C API expects int for length on some platforms
    if (!std::in_range<int>(length)) {
        std::cerr << "Length " << length << " exceeds int range, "
                  << "splitting into chunks\n";
        // Handle by splitting into INT_MAX-sized chunks
        return -1;
    }
    int safe_length = static_cast<int>(length);
    std::cout << "Sending " << safe_length << " bytes\n";
    return safe_length;  // simulated
}

// Scenario: JSON parsing returns int64_t, need int32_t

struct JsonValue {
    int64_t integer_value;
};

bool get_int32(const JsonValue& json, int32_t& out) {
    if (!std::in_range<int32_t>(json.integer_value)) {
        std::cerr << "JSON value " << json.integer_value
                  << " doesn't fit in int32_t\n";
        return false;
    }
    out = static_cast<int32_t>(json.integer_value);
    return true;
}

int main() {
    // Test safe_send
    safe_send("hello", 5);                          // OK: 5 fits in int
    safe_send("data", size_t(3'000'000'000ULL));    // Error: > INT_MAX

    // Test JSON narrowing
    JsonValue jv1{.integer_value = 42};
    JsonValue jv2{.integer_value = 5'000'000'000LL};  // > INT32_MAX
    JsonValue jv3{.integer_value = -3'000'000'000LL}; // < INT32_MIN

    int32_t result;
    get_int32(jv1, result);  // OK
    get_int32(jv2, result);  // Error: doesn't fit
    get_int32(jv3, result);  // Error: doesn't fit

    // Output:
    // Sending 5 bytes
    // Length 3000000000 exceeds int range, splitting into chunks
    // JSON value 5000000000 doesn't fit in int32_t
    // JSON value -3000000000 doesn't fit in int32_t
}
```

**Explanation:** `std::in_range<int>(value)` checks if `value` (of any integer type) fits in `int` without changing its mathematical value. This is essential at API boundaries where modern C++ uses `size_t` but legacy C APIs or serialization formats use `int`/`int32_t`. Without the check, `static_cast<int>(3'000'000'000ULL)` silently produces a wrong (negative) value.

### Q3: Show the silent bug: `if (vec.size() - 1 > idx)` wraps on empty vectors and how `std::cmp_*` fixes it

**Answer:**

There are three different fixes shown here, each with a different tradeoff. Fix v2 (check `!vec.empty()` first) is the most readable for the common case. Fix v3 with `std::cmp_less` is the right choice when `idx` comes in as a signed value from external code. The reverse-loop trap at the end is a separate but equally important pitfall to know:

```cpp
#include <utility>
#include <iostream>
#include <vector>
#include <string>

// The Bug

bool has_more_elements_buggy(const std::vector<int>& vec, size_t current_idx) {
    // Intent: check if there are elements after current_idx
    // Bug: vec.size() - 1 underflows to SIZE_MAX when vec is empty!

    return vec.size() - 1 > current_idx;
    // When vec.size() == 0:
    //   0 - 1 = 18446744073709551615 (SIZE_MAX)
    //   SIZE_MAX > current_idx -> true for any reasonable current_idx
    //   -> WRONG: empty vector "has more elements"!
}

// Fix 1: Rearrange to avoid subtraction

bool has_more_elements_v1(const std::vector<int>& vec, size_t current_idx) {
    // Move -1 to other side: size - 1 > idx  ->  size > idx + 1
    // But this has its own overflow risk if idx == SIZE_MAX!
    // Better: check directly
    return current_idx + 1 < vec.size();
    // Still risky if current_idx == SIZE_MAX (wraps to 0)
}

// Fix 2: Check empty first

bool has_more_elements_v2(const std::vector<int>& vec, size_t current_idx) {
    return !vec.empty() && current_idx < vec.size() - 1;
    // Short-circuit: if empty, returns false immediately
    // If not empty: size >= 1, so size-1 >= 0 (no underflow)
}

// Fix 3: std::cmp_less (when idx is signed)

bool has_more_elements_v3(const std::vector<int>& vec, int current_idx) {
    // If idx comes from user/API as signed int:
    // std::cmp_less handles both negative idx AND empty vector
    return std::cmp_less(current_idx + 1, vec.size());
    // current_idx = -1: -1+1 = 0, cmp_less(0, 0) -> false (correct for empty)
    // current_idx = -1: cmp_less(0, 5) -> true (still has elements)
    // current_idx = 4, size = 5: cmp_less(5, 5) -> false (last element)
}

// The reverse loop trap

void iterate_reverse_buggy(const std::vector<int>& vec) {
    // BUGGY: size_t is unsigned, i >= 0 is always true -> infinite loop
    // for (size_t i = vec.size() - 1; i >= 0; --i) {
    //     std::cout << vec[i] << " ";
    // }
    // When i reaches 0 and decrements -> SIZE_MAX -> continues!

    // ALSO BUGGY on empty vector:
    // vec.size() - 1 = SIZE_MAX -> starts at invalid index
}

void iterate_reverse_safe(const std::vector<int>& vec) {
    // Safe pattern: "goes-to" operator
    for (auto i = vec.size(); i-- > 0;) {
        std::cout << vec[i] << " ";
    }
    // When vec is empty: i=0, 0-- > 0 -> false -> loop doesn't execute
    // When vec has elements: starts at size-1, ends at 0 (inclusive)
    std::cout << "\n";
}

int main() {
    std::vector<int> empty;
    std::vector<int> data{10, 20, 30, 40, 50};

    // Show the bug
    std::cout << "Buggy check on empty vec: "
              << has_more_elements_buggy(empty, 0) << "\n";
    // Output: 1 (true!) - WRONG: empty vector has no elements

    std::cout << "Buggy check on data, idx=4: "
              << has_more_elements_buggy(data, 4) << "\n";
    // Output: 0 (false) - correct: idx 4 is last element (size 5)

    // Show the fixes
    std::cout << "\nFix v2 on empty: "
              << has_more_elements_v2(empty, 0) << "\n";
    // Output: 0 (false) - correct!

    std::cout << "Fix v3 on empty (signed): "
              << has_more_elements_v3(empty, 0) << "\n";
    // Output: 0 (false) - correct!

    std::cout << "Fix v3 on data, idx=2: "
              << has_more_elements_v3(data, 2) << "\n";
    // Output: 1 (true) - correct

    // Safe reverse iteration
    std::cout << "\nReverse: ";
    iterate_reverse_safe(data);
    // Output: 50 40 30 20 10

    std::cout << "Reverse empty: ";
    iterate_reverse_safe(empty);
    // Output: (nothing - loop doesn't execute)

    // Output:
    // Buggy check on empty vec: 1
    // Buggy check on data, idx=4: 0
    //
    // Fix v2 on empty: 0
    // Fix v3 on empty (signed): 0
    // Fix v3 on data, idx=2: 1
    //
    // Reverse: 50 40 30 20 10
    // Reverse empty:
}
```

**Explanation:** `vec.size() - 1` when `vec.size() == 0` underflows to `SIZE_MAX` (2^64 - 1 on 64-bit). This makes `SIZE_MAX > idx` true for any practical index, so the "has more elements" check incorrectly returns true for empty vectors. Fixes: (1) check `!vec.empty()` before subtracting, (2) rearrange the math to avoid underflow, or (3) use `std::cmp_less` with signed indices which handles the comparison correctly.

---

## Notes

- **The `size() - 1` pattern** is one of the most common unsigned integer bugs in C++. Never subtract from `size_t` without guarding against zero.
- **The "goes-to" operator** (`i-- > 0`) is an idiomatic safe pattern for reverse iteration with unsigned indices.
- **`std::ssize()`** (C++20) returns `ptrdiff_t` (signed), which avoids unsigned arithmetic traps for container sizes.
- **Loop alternatives:** Use `for (auto& elem : vec | std::views::reverse)` (C++20) for safe reverse iteration without index arithmetic.
- **Unsigned arithmetic is modular** (well-defined wrap), but the wrap is almost never what the programmer intended. The result is a logic bug, not UB.
- Compile with `-std=c++20 -Wall -Wextra -Wsign-compare`.
