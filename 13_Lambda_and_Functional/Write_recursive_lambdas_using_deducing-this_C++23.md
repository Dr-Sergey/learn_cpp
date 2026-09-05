# Write recursive lambdas using deducing-this (C++23)

**Category:** Lambda and Functional  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

Recursive lambdas have always been awkward in C++. The fundamental problem is that a lambda cannot refer to itself by name, because at the point where the lambda body is written, the variable holding the lambda does not exist yet. Before C++23, you had two workarounds: use `std::function` (which works but carries real runtime overhead), or use the "pass-self" trick (which works and is zero-overhead, but requires an ugly double-call syntax at the call site).

C++23's **deducing-this** feature solves this cleanly. You add `this auto& self` as the first explicit parameter, and then use `self(...)` to recurse. The compiler deduces the type of the closure and wires everything up - no `std::function`, no weird calling conventions, and zero overhead.

```cpp
// C++23: deducing-this - clean and zero-overhead!
auto fib = [](this auto& self, int n) -> int {
    return n <= 1 ? n : self(n - 1) + self(n - 2);
};

fib(10);  // 55
```

### Comparison

The table below summarizes what you had before C++23 and why deducing-this is the preferred approach going forward.

| Approach | Overhead | Syntax | Standard |
| --- | --- | --- | --- |
| `std::function` | Heap + virtual dispatch | `std::function<int(int)> fib = [&fib]...` | C++11 |
| Pass-self trick | Zero | `auto fib = [](auto& self, int n) { self(self, n-1); }` | C++14 |
| Y-combinator | Zero | Complex | C++14 |
| **Deducing-this** | **Zero** | `[](this auto& self, int n) { self(n-1); }` | **C++23** |

---

## Self-Assessment

### Q1: Write a recursive Fibonacci lambda using deducing-this without `std::function`

The `this auto& self` parameter is the key. When you write it, you are telling the compiler: "give me a reference to the closure object itself as my first parameter, and deduce its type." From inside the lambda body, calling `self(n - 1)` is just a normal function call on the closure - the compiler handles all the type machinery behind the scenes.

```cpp
#include <iostream>

int main() {
    // C++23: explicit object parameter (deducing-this)
    auto fib = [](this auto& self, int n) -> int {
        if (n <= 1) return n;
        return self(n - 1) + self(n - 2);
    };

    for (int i = 0; i <= 10; ++i) {
        std::cout << "fib(" << i << ") = " << fib(i) << "\n";
    }

    // Recursive factorial:
    auto factorial = [](this auto& self, int n) -> long long {
        return n <= 1 ? 1LL : n * self(n - 1);
    };

    std::cout << "\n12! = " << factorial(12) << "\n";

    // Recursive tree walk:
    struct Node {
        int value;
        Node* left = nullptr;
        Node* right = nullptr;
    };

    Node n4{4}, n5{5}, n2{2, &n4, &n5}, n3{3}, n1{1, &n2, &n3};

    auto inorder = [](this auto& self, const Node* node) -> void {
        if (!node) return;
        self(node->left);
        std::cout << node->value << " ";
        self(node->right);
    };

    std::cout << "Inorder: ";
    inorder(&n1);
    std::cout << "\n";
}
// Expected output:
//   fib(0) = 0
//   fib(1) = 1
//   fib(2) = 1
//   fib(3) = 2
//   fib(4) = 3
//   fib(5) = 5
//   fib(6) = 8
//   fib(7) = 13
//   fib(8) = 21
//   fib(9) = 34
//   fib(10) = 55
//
//   12! = 479001600
//   Inorder: 4 2 5 1 3
```

Notice that `fib(10)` at the call site looks exactly like a normal function call - you do not pass the lambda to itself or do anything special. The `this auto& self` parameter is the explicit object parameter and the compiler supplies it automatically when you invoke the lambda.

---

### Q2: Explain why `std::function` for recursion adds overhead compared to deducing-this

This is worth understanding concretely, not just accepting as "std::function is slow." The overhead comes from **type erasure**: `std::function` stores any callable behind a uniform interface, which requires an indirection (a virtual-call-like dispatch) on every invocation. For a recursive function called millions of times, that indirection adds up. The deducing-this version is a direct call to a concrete closure type, which the optimizer can inline completely.

```cpp
#include <iostream>
#include <functional>
#include <chrono>

int main() {
    // Approach 1: std::function (C++11)
    std::function<long long(int)> fib_func;
    fib_func = [&fib_func](int n) -> long long {
        return n <= 1 ? n : fib_func(n - 1) + fib_func(n - 2);
    };

    // Approach 2: deducing-this (C++23)
    auto fib_self = [](this auto& self, int n) -> long long {
        return n <= 1 ? n : self(n - 1) + self(n - 2);
    };

    // Both produce same result:
    std::cout << "std::function fib(30): " << fib_func(30) << "\n";
    std::cout << "deducing-this fib(30): " << fib_self(30) << "\n";

    // Benchmark:
    auto bench = [](auto fn, int n, const char* name) {
        auto start = std::chrono::steady_clock::now();
        auto result = fn(n);
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << name << ": " << result << " (" << us << " us)\n";
    };

    bench(fib_func, 35, "std::function");
    bench(fib_self, 35, "deducing-this");

    // WHY std::function is slower:
    std::cout << "\nOverhead analysis:\n";
    std::cout << "  sizeof(fib_func): " << sizeof(fib_func) << " bytes (heap alloc)\n";
    std::cout << "  sizeof(fib_self): " << sizeof(fib_self) << " bytes (stack only)\n";
    std::cout << "\n";
    std::cout << "  std::function overhead per call:\n";
    std::cout << "    1. Type-erased virtual dispatch (indirect call)\n";
    std::cout << "    2. Cannot be inlined by optimizer\n";
    std::cout << "    3. Possible heap allocation for captures\n";
    std::cout << "\n";
    std::cout << "  deducing-this overhead per call:\n";
    std::cout << "    - NONE. Direct call, fully inlinable.\n";
}
// Expected output:
//   std::function fib(30): 832040
//   deducing-this fib(30): 832040
//   std::function: 9227465 (~50000 us)   <- SLOW
//   deducing-this: 9227465 (~20000 us)   <- ~2-3x FASTER
//   sizeof(fib_func): 32 bytes (heap alloc)
//   sizeof(fib_self): 1 bytes (stack only)
```

The `sizeof` results tell the whole story. `fib_self` is 1 byte - the minimum size for any C++ object - because the closure has no data members and no captures. `fib_func` is 32 bytes just for the wrapper, and that is before any heap allocation for the captured reference to itself.

---

### Q3: Show the Y-combinator for recursive lambdas in C++14 and compare with C++23

The Y-combinator is a classic computer science concept: a higher-order function that gives a function the ability to recurse without the function needing to know its own name. In C++14, it was the cleanest zero-overhead alternative to `std::function`. Understanding it is useful both as background and because you will see it in codebases written before C++23.

The reason this trips people up is that it feels circular at first. The trick is that `YCombinator::operator()` passes `*this` as the first argument, so the inner function receives the combinator itself and can call it recursively.

```cpp
#include <iostream>

// C++14: Y-combinator
// A fixed-point combinator that enables recursion without naming:
template <typename F>
struct YCombinator {
    F func;

    template <typename... Args>
    auto operator()(Args&&... args) const {
        return func(*this, std::forward<Args>(args)...);
    }
};

// Deduction guide (C++17) or make_y helper (C++14)
template <typename F>
YCombinator<std::decay_t<F>> make_y(F&& f) {
    return {std::forward<F>(f)};
}

int main() {
    // C++14: pass-self trick
    auto fib_14 = [](auto& self, int n) -> int {
        return n <= 1 ? n : self(self, n - 1) + self(self, n - 2);
    };                                  // ^^^^ must pass self twice!
    // Must call: fib_14(fib_14, 10)
    std::cout << "C++14 self-pass: fib(10) = " << fib_14(fib_14, 10) << "\n";

    // C++14: Y-combinator wraps the self-passing
    auto fib_y = make_y([](auto& self, int n) -> int {
        return n <= 1 ? n : self(n - 1) + self(n - 2);
    });                              // ^^^^ clean call syntax
    // Clean call: fib_y(10)
    std::cout << "C++14 Y-comb:    fib(10) = " << fib_y(10) << "\n";

    // C++23: deducing-this (cleanest!)
    auto fib_23 = [](this auto& self, int n) -> int {
        return n <= 1 ? n : self(n - 1) + self(n - 2);
    };
    // Clean call: fib_23(10)
    std::cout << "C++23 deducing:  fib(10) = " << fib_23(10) << "\n";

    // Comparison
    std::cout << "\n";
    std::cout << "Approach         | Syntax at call-site | Overhead\n";
    std::cout << "---------------- | ------------------- | --------\n";
    std::cout << "self-pass (C++14)| fib(fib, 10)        | Zero\n";
    std::cout << "Y-combinator     | fib(10)             | Zero\n";
    std::cout << "std::function    | fib(10)             | HIGH\n";
    std::cout << "deducing-this    | fib(10)             | Zero\n";
}
// Expected output:
//   C++14 self-pass: fib(10) = 55
//   C++14 Y-comb:    fib(10) = 55
//   C++23 deducing:  fib(10) = 55
// ...
```

The Y-combinator and deducing-this achieve the same result with the same overhead, but deducing-this is built into the language so there is no boilerplate struct to maintain. If you are on C++23, use deducing-this. If you are stuck on C++14/17, the Y-combinator gives you the clean call syntax without the `std::function` penalty.

---

## Notes

- **`this auto& self`** is the explicit object parameter syntax. The `this` keyword here denotes the first parameter receives the object itself.
- **Mutable recursive lambda:** Use `this auto self` (by value) if you need mutation. Use `this auto& self` (by reference) for the common case.
- **No `std::function` needed:** Deducing-this completely eliminates the need for `std::function` in recursive lambdas.
- **Compiler support:** GCC 14+, Clang 18+, MSVC 19.37+ support deducing-this.
- **Performance:** In benchmarks, deducing-this recursive lambdas match hand-written recursive functions exactly.
