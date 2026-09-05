# Use std::span (C++20) to Pass Non-Owning Views to Contiguous Memory

**Category:** Memory and Ownership  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/container/span>  

---

## Topic Overview

### What Is `std::span`

`std::span` is a lightweight, non-owning view over a contiguous sequence of objects. Think of it as the modern replacement for the classic `(T* data, size_t n)` parameter pair - it bundles the pointer and the size together, and gives you a proper interface on top.

```cpp
#include <span>

void process(std::span<int> data) {     // Non-owning view
    for (int& x : data) x *= 2;        // Range-for works
    data[0] = 42;                        // Subscript access
    // data.size(), data.data(), data.subspan(), etc.
}
```

### Key Properties

| Feature | `std::span<T>` | `T* + size_t` |
| --- | --- | --- |
| Size info | Carried with pointer | Separate parameter |
| Range-for | Yes | No |
| Bounds info | `.size()` available | Must track manually |
| Accepts | vector, array, C-array | Must convert manually |
| Static extent | `span<T, N>` - compile-time size | Not possible |
| Ownership | Non-owning (like `string_view`) | Non-owning |

### `span<T>` vs `span<const T>`

| Type | Meaning |
| --- | --- |
| `span<int>` | Read-write view (can modify elements) |
| `span<const int>` | Read-only view (elements are const) |

---

## Self-Assessment

### Q1: Rewrite a function taking `(int* data, size_t n)` to take `std::span<int>` instead

The old C-style function and the span-based version are placed side by side here so you can see exactly what changes. Notice how the span version accepts all three containers - C array, `std::array`, and `std::vector` - without any overloads or manual conversions.

```cpp
#include <iostream>
#include <span>
#include <vector>
#include <array>
#include <numeric>

// OLD: C-style interface - error-prone
int sum_old(const int* data, size_t n) {
    int total = 0;
    for (size_t i = 0; i < n; ++i) total += data[i];
    return total;
}

// NEW: std::span - safe, flexible
int sum_new(std::span<const int> data) {
    int total = 0;
    for (int x : data) total += x;  // Range-for!
    return total;
    // Or: return std::accumulate(data.begin(), data.end(), 0);
}

// span also works for modification
void double_values(std::span<int> data) {
    for (int& x : data) x *= 2;
}

int main() {
    std::cout << "=== Rewriting ptr+size to span ===\n\n";

    // Works with C arrays
    int c_arr[] = {1, 2, 3, 4, 5};
    std::cout << "C array:    " << sum_new(c_arr) << "\n";

    // Works with std::array
    std::array<int, 4> std_arr = {10, 20, 30, 40};
    std::cout << "std::array: " << sum_new(std_arr) << "\n";

    // Works with std::vector
    std::vector<int> vec = {100, 200, 300};
    std::cout << "vector:     " << sum_new(vec) << "\n";

    // Works with subranges
    std::cout << "subspan:    " << sum_new(std::span(vec).subspan(1)) << "\n";

    // Modification through span
    double_values(vec);
    std::cout << "After 2x:   ";
    for (int x : vec) std::cout << x << " ";
    std::cout << "\n";

    // Old C-style - requires manual conversion
    std::cout << "C-style:    " << sum_old(c_arr, 5) << "\n";
    std::cout << "C-style:    " << sum_old(vec.data(), vec.size()) << "\n";

    return 0;
}
```

The old-style calls make you pass the size separately every time. With span, the size travels with the view - you can't accidentally pass the wrong one.

### Q2: Explain the difference between `std::span<T>` and `std::span<const T>`

`span<T>` and `span<const T>` work the same way as `T&` and `const T&` for references: one lets you modify the elements, the other doesn't. Choosing the right one makes your intent visible in the function signature.

```cpp
#include <iostream>
#include <span>
#include <vector>

// span<T>: Read-Write view - can modify elements through the span
void fill_with(std::span<int> data, int value) {
    for (int& x : data) x = value;  // OK: non-const span
}

// span<const T>: Read-Only view - cannot modify elements
int compute_sum(std::span<const int> data) {
    int total = 0;
    for (int x : data) total += x;
    // data[0] = 99;  // ERROR: data elements are const
    return total;
}

// Function that should NOT modify input - use span<const T>
void print_data(std::span<const int> data) {
    for (size_t i = 0; i < data.size(); ++i)
        std::cout << data[i] << (i + 1 < data.size() ? ", " : "\n");
}

int main() {
    std::cout << "=== span<T> vs span<const T> ===\n\n";

    std::vector<int> v = {1, 2, 3, 4, 5};

    // span<const int> from vector - read only
    print_data(v);
    std::cout << "Sum: " << compute_sum(v) << "\n";

    // span<int> from vector - read/write
    fill_with(v, 42);
    print_data(v);

    // const vector -> can only create span<const int>
    const std::vector<int> cv = {10, 20, 30};
    print_data(cv);              // OK: span<const int>
    // fill_with(cv, 0);         // ERROR: can't get span<int> from const vector

    // Conversion: span<int> -> span<const int> is implicit
    std::span<int> rw(v);
    std::span<const int> ro = rw;  // OK: implicit conversion (add const)
    // std::span<int> rw2 = ro;    // ERROR: can't remove const

    std::cout << "\nKey rules:\n";
    std::cout << "  span<T>       = read/write, like T&\n";
    std::cout << "  span<const T> = read-only, like const T&\n";
    std::cout << "  span<int> -> span<const int>: OK (implicit)\n";
    std::cout << "  span<const int> -> span<int>: ERROR\n";

    return 0;
}
```

A good rule of thumb: if the function only reads elements, take `span<const T>`; if it needs to write, take `span<T>`. This mirrors the `const T&` vs `T&` pattern you already know from references.

### Q3: Show how `std::span` prevents an off-by-one error that a raw pointer interface would not catch

The C-style function below has a classic `<=` vs `<` bug in the loop. With a raw pointer you will not notice at runtime - the read past the end is silent undefined behavior. The span-based version can't have the same bug because range-for and `size()` are always in sync.

```cpp
#include <iostream>
#include <span>
#include <vector>
#include <cassert>

// C-style: off-by-one goes undetected
void process_c_style(int* data, size_t n) {
    // Bug: loop goes one past the end (n should be < n, not <= n)
    for (size_t i = 0; i <= n; ++i) {  // OFF BY ONE: <= instead of <
        data[i] *= 2;                   // UB on last iteration!
    }
    // No runtime error - silent corruption
}

// span-style: bounds checking available
void process_span_safe(std::span<int> data) {
    // span knows its size - can't accidentally use wrong N
    for (size_t i = 0; i < data.size(); ++i) {
        data[i] *= 2;  // size() is always correct
    }

    // Even better: range-for eliminates indexing entirely
    // for (int& x : data) x *= 2;
}

// Subspan prevents buffer overflows
void process_first_n(std::span<int> data, size_t n) {
    // C-style: if n > array_size -> buffer overflow
    // span: subspan checks bounds (in debug, or throws)
    auto slice = data.first(std::min(n, data.size()));
    for (int& x : slice) x *= 2;
}

int main() {
    std::cout << "=== span prevents off-by-one ===\n\n";

    std::vector<int> v = {1, 2, 3, 4, 5};

    // C-style: must pass size separately - easy to get wrong
    // process_c_style(v.data(), v.size());  // BUG: <= in loop

    // span: size is bundled with pointer - always correct
    process_span_safe(v);
    std::cout << "After safe processing: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    // Subspan with bounds protection
    v = {1, 2, 3, 4, 5};
    process_first_n(v, 3);  // Only processes first 3
    std::cout << "After first_n(3): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    // Static extent span: size known at compile time
    std::span<int, 5> fixed(v);  // Exactly 5 elements
    // std::span<int, 3> wrong(v);  // Compile error if sizes don't match

    std::cout << "\n=== Why span is safer ===\n";
    std::cout << "1. size() always matches actual data - no stale count\n";
    std::cout << "2. Range-for eliminates manual indexing\n";
    std::cout << "3. subspan/first/last provide bounds-safe slicing\n";
    std::cout << "4. Static extent catches size mismatches at compile time\n";
    std::cout << "5. Can't accidentally pass wrong size to a function\n";

    return 0;
}
```

The static-extent form `span<int, 5>` goes even further: the compiler checks that the source has exactly 5 elements. That turns a potential runtime buffer overread into a compile-time error.

---

## Notes

- `std::span` is the pointer+size replacement - always use it instead of `(T* data, size_t n)`.
- It's non-owning like `string_view` - the underlying data must outlive the span.
- `span<T, N>` (static extent) stores size at compile time - zero overhead, stronger guarantees.
- `span<T>` (dynamic extent) stores size at runtime - `sizeof(span) == sizeof(T*) + sizeof(size_t)`.
- Accepts any contiguous container: `vector`, `array`, C arrays, `string`, other spans.
