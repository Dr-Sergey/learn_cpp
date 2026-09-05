# Know static lambdas (C++23) and when they enable optimizations

**Category:** Lambda and Functional  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

### What Are Static Lambdas

C++23 allows the `static` specifier on a lambda's call operator. A **static lambda** is a lambda with no captures whose `operator()` is declared `static`. Here's the simplest form:

```cpp
auto add = [](int a, int b) static { return a + b; };
```

This tells both the compiler and the reader something important: *this lambda captures nothing and its call operator has no implicit `this` parameter*. It's a deliberate, enforced statement of intent.

### Why Does It Matter

Without `static`, a captureless lambda's `operator()` is a **non-static member function** of the closure type. Even though there's no state, the compiler still passes a `this` pointer (pointing to the empty closure object). Most compilers optimize this away in simple cases, but there are real situations where it matters:

1. **ABI boundaries** - At shared-library boundaries, the unused `this` parameter can prevent tail calls or occupy a register unnecessarily.
2. **Function pointer conversion** - A captureless lambda can convert to a function pointer. With `static`, the call operator itself is already a free function - no thunk needed.
3. **constexpr / consteval contexts** - Marking `static` makes intent explicit and can simplify constant evaluation.

The reason this trips people up is that "captureless" and "stateless" feel equivalent - and in practice they usually are - but the ABI difference becomes visible at boundaries the compiler can't see through, like C callbacks or shared library interfaces.

### Syntax

The placement of `static` is right after the parameter list, before any trailing return type. It can be combined with `constexpr`:

```cpp
// C++23 static lambda
auto square = [](int x) static -> int { return x * x; };

// Also works with constexpr
auto cube = [](int x) static constexpr -> int { return x * x * x; };

// ERROR: cannot capture and be static
// auto bad = [&val](int x) static { return x + val; };  // ill-formed
```

If you try to add a capture, the compiler will reject it - which is exactly the point. The `static` keyword turns a good habit into an enforced contract.

### When It Enables Real Optimizations

In hot loops where an algorithm takes a callable parameter, `static` ensures the compiler sees a true free function with no hidden indirection. Notice the comment in this example:

```cpp
#include <algorithm>
#include <vector>

void sort_descending(std::vector<int>& v) {
    // The static lambda guarantees no hidden state.
    // The compiler can inline this directly as a function pointer
    // without a closure-object indirection.
    std::sort(v.begin(), v.end(),
              [](int a, int b) static { return a > b; });
}
```

The more compelling case is when you need to pass a lambda to a C API that demands a raw function pointer. Without `static`, the compiler has to generate a tiny adapter thunk. With `static`, the call operator *is* the function pointer - zero extra machinery:

```cpp
extern "C" {
    using Comparator = int(*)(const void*, const void*);
    void qsort(void* base, size_t nmemb, size_t size, Comparator comp);
}

void c_sort(int* arr, size_t n) {
    // static lambda converts directly to function pointer - zero overhead
    qsort(arr, n, sizeof(int),
          [](const void* a, const void* b) static -> int {
              return *static_cast<const int*>(a) - *static_cast<const int*>(b);
          });
}
```

### Comparison with Non-Static Captureless Lambdas

If the table feels like a lot, the key row is "Hidden `this` param" - that's the only real behavioral difference. Everything else flows from it.

| Feature | Non-static captureless | `static` lambda (C++23) |
| --- | --- | --- |
| `operator()` | Non-static member function | Static member function |
| Hidden `this` param | Yes (optimized away usually) | No |
| Converts to fn pointer | Yes (via conversion operator) | Yes (direct) |
| Captures allowed | No captures (for conversion) | No captures (enforced) |
| Intent clarity | Implicit | Explicit |

---

## Self-Assessment

### Q1: Write a static lambda and show it converts to a raw function pointer without a thunk

The key thing to observe here is that the conversion to a function pointer is direct - no intermediate step. Try this with a regular lambda and you'll get the same behavior in most compilers, but the mechanism is different under the hood.

```cpp
#include <iostream>

int main() {
    // Static lambda - operator() is a static member function
    auto add = [](int a, int b) static -> int { return a + b; };

    // Direct conversion to function pointer
    int (*fn_ptr)(int, int) = add;

    std::cout << fn_ptr(3, 4) << "\n";  // 7

    // Verify it's the same address as the static operator()
    // (implementation-defined, but typically identical)
    std::cout << "fn_ptr address: " << reinterpret_cast<void*>(fn_ptr) << "\n";
    return 0;
}
```

### Q2: Show a compile error when trying to capture in a static lambda

This example is really about understanding the constraint. The compiler error message is the feature - it tells you exactly why your design is inconsistent.

```cpp
int main() {
    int factor = 10;

    // ERROR: a static lambda cannot have captures
    // auto bad = [factor](int x) static { return x * factor; };
    //            ^~~~~~~ error: 'static' lambda cannot have captures

    // Correct: use a regular lambda if you need captures
    auto good = [factor](int x) { return x * factor; };

    // Or pass factor as a parameter to keep the lambda static
    auto also_good = [](int x, int f) static { return x * f; };

    return 0;
}
```

### Q3: Demonstrate a performance-relevant use case where static lambdas help

Here you can see the before/after story in one function. The commented-out version shows what the compiler had to do previously; the `static` version eliminates that work entirely.

```cpp
#include <cstdlib>
#include <iostream>

// C API that takes a function pointer callback
extern "C" using CmpFn = int(*)(const void*, const void*);

void demo_qsort(int* data, size_t n) {
    // Before C++23: captureless lambda + implicit conversion
    // The compiler generates a thunk: closure::operator() -> function pointer adapter
    // auto old_cmp = [](const void* a, const void* b) -> int { ... };

    // C++23: static lambda - the operator() IS the function pointer, no adapter
    qsort(data, n, sizeof(int),
          [](const void* a, const void* b) static -> int {
              int ia = *static_cast<const int*>(a);
              int ib = *static_cast<const int*>(b);
              return (ia > ib) - (ia < ib);
          });
}

int main() {
    int arr[] = {5, 2, 8, 1, 9, 3};
    demo_qsort(arr, 6);
    for (int x : arr) std::cout << x << " ";
    // Output: 1 2 3 5 8 9
    return 0;
}
```

---

## Notes

- `static` on a lambda is a **C++23** feature - check compiler support (`-std=c++23`).
- It's a compile-time guarantee of no captures - self-documenting and enforced by the compiler.
- The main practical benefit is at ABI/FFI boundaries and in generic code that stores function pointers.
- In purely inlined code, modern compilers already optimize away the `this` parameter for captureless lambdas, so the difference shows up mostly at visibility boundaries.
