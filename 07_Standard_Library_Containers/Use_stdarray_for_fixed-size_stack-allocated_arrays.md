# Use std::array for fixed-size stack-allocated arrays

**Category:** Standard Library Containers  
**Standard:** C++11 (constexpr improvements in C++14/17/20)  
**Reference:** <https://en.cppreference.com/w/cpp/container/array>  

---

## Topic Overview

`std::array<T, N>` is a fixed-size container that wraps a C-style array with a full standard container interface. It lives entirely on the **stack** (no heap allocation), supports bounds checking, and works seamlessly with algorithms, ranges, and structured bindings. If you know the size at compile time, this is the right tool - it gives you everything a C-style array offers, plus all the things it was missing.

### std::array vs C-style array

| Feature | `int arr[5]` | `std::array<int,5>` |
| --- | --- | --- |
| Size known at compile time | Via `sizeof(arr)/sizeof(arr[0])` - fragile | `.size()` - always correct |
| Bounds checking | No | `.at()` throws `out_of_range` |
| Decays to pointer | Yes (dangerous!) | No - explicit `.data()` needed |
| Copyable | No (must memcpy) | Yes - `operator=` works |
| Comparable | No | Yes - `==`, `<`, etc. |
| Works with algorithms | Via raw pointers | Via `.begin()`/`.end()` |
| Structured bindings | No | Yes (C++17) |
| constexpr friendly | Limited | Fully (C++17+) |
| Zero overhead | Baseline | Same - no overhead |

### Memory Layout

`std::array` has zero overhead compared to a raw C-style array - they are bit-for-bit identical in memory. There is no hidden size field, no pointer, nothing extra:

```cpp
std::array<int, 4> a = {10, 20, 30, 40};

Stack:  [10][20][30][40]    <- contiguous, no heap
         ^
         a.data()

sizeof(a) == 4 * sizeof(int) == 16 bytes
// No hidden pointers, no size field - identical layout to int[4]
```

### Core API

Here is a tour of the essential operations. Notice how they all feel like a regular standard container:

```cpp
#include <array>
#include <iostream>
#include <algorithm>

int main() {
    std::array<int, 5> a = {3, 1, 4, 1, 5};

    // Size
    std::cout << "size: " << a.size() << "\n";        // 5 (constexpr)
    std::cout << "empty: " << a.empty() << "\n";      // false

    // Access
    std::cout << "a[0]: " << a[0] << "\n";            // 3 (no bounds check)
    std::cout << "a.at(0): " << a.at(0) << "\n";      // 3 (bounds check)
    std::cout << "front: " << a.front() << "\n";       // 3
    std::cout << "back: " << a.back() << "\n";         // 5

    // Raw pointer
    int* p = a.data();  // Pointer to first element

    // Iterators
    std::sort(a.begin(), a.end());  // {1, 1, 3, 4, 5}

    // Fill
    std::array<int, 5> b;
    b.fill(42);  // All elements = 42

    // Swap (O(N) - swaps element-by-element)
    a.swap(b);

    // Comparison
    std::array<int, 5> c = {1, 1, 3, 4, 5};
    std::cout << "a == c: " << (a == c) << "\n";

    return 0;
}
```

### Important Notes

- `std::array<T, 0>` is valid - an empty array. `data()` may return nullptr, and `begin() == end()`.
- `std::array` is an **aggregate** - there is no constructor, just brace initialization.
- Since C++17, CTAD works: `std::array a = {1, 2, 3};` deduces `std::array<int, 3>`.
- `std::to_array` (C++20) creates an array from a C-array or braced list with deduction.

---

## Self-Assessment

### Q1: Replace a C-style int arr[N] with std::array<int,N> and show range checking with .at()

The problems with C-style arrays are subtle until something goes wrong. This example makes them concrete and shows how `std::array` fixes each one:

```cpp
#include <iostream>
#include <array>
#include <stdexcept>
#include <numeric>

int main() {
    // === C-style array (problems) ===
    {
        int arr[5] = {10, 20, 30, 40, 50};

        // Problem 1: Size is fragile
        size_t size = sizeof(arr) / sizeof(arr[0]);  // Only works locally!
        // If passed to a function, arr decays to int* and size is lost.

        // Problem 2: No bounds checking
        // arr[10] = 99;  // UNDEFINED BEHAVIOR - no error at compile/run time!

        // Problem 3: Can't copy
        int arr2[5];
        // arr2 = arr;  // ERROR: can't assign C-arrays
        std::copy(arr, arr + 5, arr2);  // Manual workaround
    }

    // === std::array replacement ===
    {
        std::array<int, 5> arr = {10, 20, 30, 40, 50};

        // Solution 1: .size() always works, even in functions
        std::cout << "Size: " << arr.size() << "\n";  // 5

        // Solution 2: .at() provides bounds checking
        try {
            std::cout << "arr.at(2): " << arr.at(2) << "\n";   // 30
            std::cout << "arr.at(10): " << arr.at(10) << "\n";  // throws!
        } catch (const std::out_of_range& e) {
            std::cout << "Caught: " << e.what() << "\n";
            // Output: Caught: array::at: __n (which is 10) >= _Nm (which is 5)
        }

        // operator[] is still available (no bounds check, same as C-array)
        std::cout << "arr[0]: " << arr[0] << "\n";  // 10

        // Solution 3: Copyable
        std::array<int, 5> arr2 = arr;  // Just works!
        arr2[0] = 999;
        std::cout << "arr[0]=" << arr[0] << " arr2[0]=" << arr2[0] << "\n";
        // Output: arr[0]=10 arr2[0]=999 (independent copies)

        // Solution 4: Comparable
        std::array<int, 5> arr3 = {10, 20, 30, 40, 50};
        std::cout << "arr == arr3: " << std::boolalpha << (arr == arr3) << "\n";
        // Output: arr == arr3: true

        // Works with algorithms
        int sum = std::accumulate(arr.begin(), arr.end(), 0);
        std::cout << "Sum: " << sum << "\n";  // 150

        // Does NOT decay to pointer
        // void f(int* p);  // C-array would decay
        // f(arr);           // ERROR with std::array - must be explicit
        // f(arr.data());    // OK - explicit conversion
    }

    return 0;
}
```

The `.at()` vs `[]` split is intentional: use `[]` in performance-critical hot paths where you are sure the index is valid, and use `.at()` anywhere you want an exception rather than undefined behavior on a bad index.

### Q2: Show that std::array supports structured bindings, range-for, and algorithm compatibility

One of the nicest things about `std::array` is that it plays well with every modern C++ feature. Here they all are together:

```cpp
#include <iostream>
#include <array>
#include <algorithm>
#include <numeric>
#include <ranges>  // C++20

int main() {
    // === Structured bindings (C++17) ===
    {
        std::array<int, 3> rgb = {255, 128, 0};
        auto [r, g, b] = rgb;
        std::cout << "R=" << r << " G=" << g << " B=" << b << "\n";
        // Output: R=255 G=128 B=0

        // Works with auto& to modify:
        auto& [x, y, z] = rgb;
        x = 0;
        std::cout << "After: " << rgb[0] << "\n";  // 0
    }

    // === Range-for loop ===
    {
        std::array<std::string, 4> names = {"Alice", "Bob", "Carol", "Dave"};

        std::cout << "Names: ";
        for (const auto& name : names) {
            std::cout << name << " ";
        }
        std::cout << "\n";
        // Output: Names: Alice Bob Carol Dave
    }

    // === Algorithm compatibility ===
    {
        std::array<int, 6> a = {5, 2, 8, 1, 9, 3};

        // Sort
        std::sort(a.begin(), a.end());
        // a = {1, 2, 3, 5, 8, 9}

        // Binary search
        bool found = std::binary_search(a.begin(), a.end(), 5);
        std::cout << "Found 5: " << std::boolalpha << found << "\n";  // true

        // Accumulate
        int sum = std::accumulate(a.begin(), a.end(), 0);
        std::cout << "Sum: " << sum << "\n";  // 28

        // Transform
        std::array<int, 6> doubled;
        std::transform(a.begin(), a.end(), doubled.begin(),
                       [](int x) { return x * 2; });
        std::cout << "Doubled: ";
        for (int x : doubled) std::cout << x << " ";
        std::cout << "\n";
        // Output: Doubled: 2 4 6 10 16 18

        // C++20 Ranges
        auto evens = a | std::views::filter([](int x) { return x % 2 == 0; });
        std::cout << "Evens: ";
        for (int x : evens) std::cout << x << " ";
        std::cout << "\n";
        // Output: Evens: 2 8

        // std::to_array (C++20): create from C-array
        int c_arr[] = {10, 20, 30};
        auto arr = std::to_array(c_arr);
        static_assert(arr.size() == 3);
    }

    // === constexpr usage (C++17+) ===
    {
        constexpr std::array<int, 4> a = {1, 2, 3, 4};
        constexpr int first = a[0];      // Compile-time access
        constexpr int sz = a.size();      // Compile-time size
        static_assert(first == 1);
        static_assert(sz == 4);

        // constexpr sort (C++20)
        constexpr auto sorted = []() {
            std::array<int, 4> a = {4, 2, 3, 1};
            std::sort(a.begin(), a.end());
            return a;
        }();
        static_assert(sorted[0] == 1);
    }

    return 0;
}
```

Structured bindings work because `std::array` has `std::tuple_size` and `std::get` specializations. The constexpr sort (C++20) is a particularly satisfying feature - you can produce a compile-time sorted lookup table with zero runtime cost.

### Q3: Write a function template that works with both std::array and std::vector via std::span

The classic problem with array-accepting functions is that you end up writing one version for `vector` and another for `array`. `std::span` (C++20) solves this cleanly - it is a non-owning view over any contiguous memory:

```cpp
#include <iostream>
#include <array>
#include <vector>
#include <span>
#include <numeric>
#include <algorithm>

// === std::span: a non-owning view over contiguous memory ===
// Works with: vector, array, C-array, string, any contiguous range

// Generic function accepting ANY contiguous container
double average(std::span<const int> data) {
    if (data.empty()) return 0.0;
    int sum = std::accumulate(data.begin(), data.end(), 0);
    return static_cast<double>(sum) / data.size();
}

// Function that modifies elements (non-const span)
void double_values(std::span<int> data) {
    for (int& x : data) x *= 2;
}

// Fixed-size span for compile-time safety
void process_rgb(std::span<const uint8_t, 3> rgb) {
    std::cout << "R=" << static_cast<int>(rgb[0])
              << " G=" << static_cast<int>(rgb[1])
              << " B=" << static_cast<int>(rgb[2]) << "\n";
}

// Template function with span
template <typename T>
T find_max(std::span<const T> data) {
    return *std::max_element(data.begin(), data.end());
}

int main() {
    // === Works with std::array ===
    std::array<int, 5> arr = {10, 20, 30, 40, 50};
    std::cout << "array avg: " << average(arr) << "\n";  // 30

    // === Works with std::vector ===
    std::vector<int> vec = {5, 15, 25, 35, 45};
    std::cout << "vector avg: " << average(vec) << "\n";  // 25

    // === Works with C-array ===
    int c_arr[] = {1, 2, 3, 4, 5};
    std::cout << "C-array avg: " << average(c_arr) << "\n";  // 3

    // === Works with subranges ===
    std::cout << "first 3 avg: " << average({arr.data(), 3}) << "\n";  // 20

    // === Modification via span ===
    double_values(arr);
    std::cout << "Doubled array: ";
    for (int x : arr) std::cout << x << " ";
    std::cout << "\n";
    // Output: Doubled array: 20 40 60 80 100

    // === Fixed-size span ===
    std::array<uint8_t, 3> color = {255, 128, 0};
    process_rgb(color);  // OK: array<uint8_t,3> -> span<const uint8_t, 3>
    // process_rgb(vec);  // ERROR: wrong size/type

    // === Template function ===
    std::array<double, 4> da = {1.5, 3.7, 2.1, 4.9};
    std::cout << "Max: " << find_max<double>(da) << "\n";  // 4.9

    // === Key: span doesn't own data ===
    // std::span<int> s = arr;  // s just points to arr's memory
    // If arr goes out of scope, s is DANGLING

    return 0;
}
```

`std::span` is the modern C++ replacement for the old `(T* ptr, size_t len)` function parameter pattern. It stores just a pointer and a size - 16 bytes on 64-bit - and is passed by value. The fixed-extent form `std::span<T, N>` gives you compile-time size checking, which catches type mismatches at compile time rather than producing a runtime crash.

---

## Notes

- **Zero overhead:** `std::array` generates identical machine code to a C-style array. The compiler sees through the abstraction completely.
- **No heap allocation:** Unlike `std::vector`, `std::array` is always stack-allocated (or wherever the enclosing object lives). This makes it ideal for performance-critical code with known sizes.
- **Aggregate initialization:** `std::array<int, 3> a = {1, 2, 3};` - the double braces `{{1,2,3}}` are optional in C++14+.
- **`std::array<T, 0>`:** Valid but unusual. `data()` returns an unspecified pointer, `size()` returns 0. Iterating is a no-op.
- **Use `std::array` over C-arrays** in all new code. The only exception is when interacting with C APIs that require `T[]` - use `.data()` for that.
