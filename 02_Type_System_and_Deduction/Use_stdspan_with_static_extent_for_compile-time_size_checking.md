# Use `std::span` with Static Extent for Compile-Time Size Checking

**Category:** Type System and Deduction  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/container/span>  

---

## Topic Overview

### What Is `std::span`

`std::span<T>` is a lightweight, non-owning view over a contiguous sequence of elements. Think of it as a safer replacement for the old `(T* ptr, size_t size)` pair - it bundles both into a single, type-safe object without taking ownership of the memory.

### Static vs Dynamic Extent

The second template parameter is where things get interesting. Leave it out and you get a dynamic extent (size tracked at runtime). Provide a number and you get a static extent (size baked into the type itself).

| Declaration | Extent | Size Known At | `sizeof(span)` |
| --- | --- | --- | --- |
| `std::span<int>` | Dynamic (`std::dynamic_extent`) | Runtime | 2 × pointer (ptr + size) |
| `std::span<int, 4>` | Static (4) | Compile time | 1 × pointer (ptr only) |

### Core Syntax

Here's the basic usage showing both extent flavors and how they interact:

```cpp
#include <span>
#include <array>

int arr[4] = {1, 2, 3, 4};
std::array<int, 4> stdarr = {1, 2, 3, 4};

// Static extent — compiler knows the size
std::span<int, 4> s1{arr};        // OK: C-array of 4
std::span<int, 4> s2{stdarr};     // OK: std::array of 4
// std::span<int, 4> s3{arr, 3};  // ERROR: size mismatch at compile time

// Dynamic extent — size determined at runtime
std::span<int> d1{arr};           // OK: deduces size = 4
std::span<int> d2{arr, 3};        // OK: only first 3 elements

// Static -> dynamic: always OK (implicit conversion)
std::span<int> d3 = s1;

// Dynamic -> static: NOT implicit (would need static_cast or subspan)
```

Notice that converting from static to dynamic is always fine - you're just adding a runtime size field. Going the other direction is not implicit because the compiler can't verify the size is correct without your explicit say-so.

### Why Static Extent

Static extent is not just a micro-optimization. It changes what the compiler can check for you:

1. **Compile-time size errors** - pass wrong-size array and the compiler rejects it
2. **Zero overhead** - no need to store the size (it's in the type)
3. **Stronger contracts** - functions that require exactly N elements express this in the type

---

## Self-Assessment

### Q1: Declare `std::span<int, 4>` and show the compile error when a 3-element array is passed

The key thing to watch here is how the static extent becomes part of the function's contract - a mismatch is caught before the program even runs.

```cpp
#include <iostream>
#include <span>
#include <array>
#include <numeric>

// A function that requires exactly 4 elements
void process_quad(std::span<int, 4> quad) {
    std::cout << "Processing quad: ";
    for (int x : quad) std::cout << x << " ";
    std::cout << "(sum=" << std::accumulate(quad.begin(), quad.end(), 0) << ")\n";
}

// A function that requires exactly 3 elements
void process_triple(std::span<const double, 3> triple) {
    std::cout << "Processing triple: ";
    for (double x : triple) std::cout << x << " ";
    std::cout << "\n";
}

int main() {
    // === Valid: matching sizes ===
    int arr4[4] = {1, 2, 3, 4};
    std::array<int, 4> stdarr4 = {10, 20, 30, 40};

    process_quad(arr4);        // OK: C-array of 4
    process_quad(stdarr4);     // OK: std::array of 4

    double d3[3] = {1.1, 2.2, 3.3};
    process_triple(d3);        // OK: C-array of 3

    // === COMPILE ERROR: size mismatch ===
    // int arr3[3] = {1, 2, 3};
    // process_quad(arr3);     // ERROR: cannot convert span<int, 3> to span<int, 4>

    // std::array<int, 5> arr5 = {1, 2, 3, 4, 5};
    // process_quad(arr5);     // ERROR: cannot convert span<int, 5> to span<int, 4>

    // === Static extent captures size in the type ===
    std::span<int, 4> s4{arr4};
    static_assert(s4.extent == 4);
    static_assert(s4.size() == 4);  // constexpr for static extent!

    // Dynamic extent — size() is runtime
    std::span<int> dyn{arr4};
    // static_assert(dyn.size() == 4);  // ERROR: not constexpr for dynamic extent

    std::cout << "\nStatic extent: " << s4.extent << "\n";
    std::cout << "sizeof(span<int,4>): " << sizeof(std::span<int, 4>) << "\n";
    std::cout << "sizeof(span<int>):   " << sizeof(std::span<int>) << "\n";

    return 0;
}
```

Notice the `static_assert` on `s4.size()` - that works only because the size is a compile-time constant for static extents. With a dynamic span, `size()` is a runtime call and cannot be used in a `static_assert`.

**Output:**

```text
Processing quad: 1 2 3 4 (sum=10)
Processing quad: 10 20 30 40 (sum=100)
Processing triple: 1.1 2.2 3.3

Static extent: 4
sizeof(span<int,4>): 8
sizeof(span<int>):   16
```

### Q2: Explain when to use dynamic extent (`std::dynamic_extent`) vs static extent

The table below is a useful cheat sheet, but the mental model is simple: use static extent when the size is part of the contract, dynamic extent when you want flexibility.

| Use Static Extent When... | Use Dynamic Extent When... |
| --- | --- |
| Size is fixed at compile time | Size varies at runtime |
| Function contract requires exact N elements | Function works with any-size range |
| Maximum performance needed (no size stored) | Accepting slices of larger containers |
| Working with fixed-size arrays/structs | Interfacing with C APIs (`ptr + size`) |

Here's a side-by-side showing each in a realistic context:

```cpp
#include <iostream>
#include <span>
#include <vector>
#include <array>
#include <numeric>

// STATIC: Matrix row must be exactly 3 columns
void process_row(std::span<const float, 3> row) {
    std::cout << "  Row: " << row[0] << ", " << row[1] << ", " << row[2] << "\n";
}

// DYNAMIC: Print any number of ints
void print_all(std::span<const int> data) {
    std::cout << "  [" << data.size() << " elements]: ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";
}

// DYNAMIC: Process variable-length buffer from C API
void handle_buffer(std::span<const std::byte> buf) {
    std::cout << "  Buffer: " << buf.size() << " bytes\n";
}

int main() {
    // Static extent: compile-time size enforcement
    float matrix[2][3] = {{1.0f, 2.0f, 3.0f}, {4.0f, 5.0f, 6.0f}};
    std::cout << "Static extent (matrix rows):\n";
    process_row(matrix[0]);
    process_row(matrix[1]);

    // Dynamic extent: works with any container
    std::vector<int> vec = {1, 2, 3, 4, 5};
    std::array<int, 3> arr = {10, 20, 30};
    int c_arr[] = {100, 200};

    std::cout << "\nDynamic extent (any size):\n";
    print_all(vec);
    print_all(arr);
    print_all(c_arr);

    // Slicing with dynamic extent
    print_all(std::span{vec}.subspan(1, 3));  // elements [1,2,3]

    return 0;
}
```

**Output:**

```text
Static extent (matrix rows):
  Row: 1, 2, 3
  Row: 4, 5, 6

Dynamic extent (any size):
  [5 elements]: 1 2 3 4 5
  [3 elements]: 10 20 30
  [2 elements]: 100 200
  [3 elements]: 2 3 4
```

### Q3: Show how static extent enables zero-overhead `subspan` with compile-time offset and count

When you call `subspan` with template (compile-time) arguments, the resulting span also has a static extent. That means no extra size field, no runtime arithmetic - just a pointer offset.

```cpp
#include <iostream>
#include <span>
#include <array>

int main() {
    std::array<int, 8> data = {10, 20, 30, 40, 50, 60, 70, 80};
    std::span<int, 8> full{data};

    // subspan with compile-time offset and count -> static extent result
    auto first3 = full.subspan<0, 3>();   // span<int, 3>
    auto mid2   = full.subspan<3, 2>();   // span<int, 2>
    auto last3  = full.subspan<5, 3>();   // span<int, 3>

    // The result type has STATIC extent — checked at compile time
    static_assert(first3.extent == 3);
    static_assert(mid2.extent == 2);
    static_assert(last3.extent == 3);

    // Zero overhead: subspan just offsets the pointer, no runtime size tracking
    static_assert(sizeof(first3) == sizeof(int*));  // only stores pointer
    static_assert(sizeof(mid2) == sizeof(int*));

    std::cout << "first3: ";
    for (int x : first3) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "mid2:   ";
    for (int x : mid2) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "last3:  ";
    for (int x : last3) std::cout << x << " ";
    std::cout << "\n";

    // Compile-time safety: this would fail
    // auto bad = full.subspan<6, 3>();  // ERROR: 6+3=9 > 8

    // first() and last() also produce static extents
    auto first4 = full.first<4>();   // span<int, 4>
    auto last4  = full.last<4>();    // span<int, 4>
    static_assert(first4.extent == 4);
    static_assert(last4.extent == 4);

    std::cout << "\nfirst<4>: ";
    for (int x : first4) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "last<4>:  ";
    for (int x : last4) std::cout << x << " ";
    std::cout << "\n";

    // Contrast: runtime subspan returns dynamic extent
    size_t offset = 2;
    auto dyn = full.subspan(offset, 3);  // span<int, dynamic_extent>
    static_assert(dyn.extent == std::dynamic_extent);
    std::cout << "\ndynamic subspan: ";
    for (int x : dyn) std::cout << x << " ";
    std::cout << " (sizeof=" << sizeof(dyn) << ")\n";

    // Summary: template subspan is zero-overhead, runtime subspan costs one size_t
    std::cout << "\nsizeof(span<int,3>):       " << sizeof(std::span<int, 3>) << "\n";
    std::cout << "sizeof(span<int>):         " << sizeof(std::span<int>) << "\n";

    return 0;
}
```

The `sizeof` comparison at the end tells the whole story: a static-extent span is just a pointer, while a dynamic-extent span carries an extra `size_t`. The template-based `subspan` preserves that zero-overhead property all the way through.

**Output:**

```text
first3: 10 20 30
mid2:   40 50
last3:  60 70 80

first<4>: 10 20 30 40
last<4>:  50 60 70 80

dynamic subspan: 30 40 50 (sizeof=16)

sizeof(span<int,3>):       8
sizeof(span<int>):         16
```

---

## Notes

- **`std::span` does NOT own memory.** Ensure the underlying data outlives the span. Dangling spans are UB just like dangling pointers.
- Static and dynamic extents interoperate: `span<T, N>` implicitly converts to `span<T>`, but not the reverse.
- `span<const T>` provides read-only access. Prefer `span<const T>` in function parameters for input data.
- `std::span` replaces `gsl::span` from the C++ Core Guidelines. The design is nearly identical.
