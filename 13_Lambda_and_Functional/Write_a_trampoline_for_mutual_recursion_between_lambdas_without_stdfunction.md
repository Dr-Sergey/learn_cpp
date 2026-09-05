# Write a trampoline for mutual recursion between lambdas without std::function

**Category:** Lambda and Functional  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

**Mutual recursion** between lambdas is tricky because each lambda must reference the other, but lambdas have unique, unnamed types - you can't declare one before defining it. The problem is that when you write `is_even`, it tries to capture `is_odd`, but `is_odd` doesn't exist yet. And you can't flip the order because then `is_odd` would be capturing an undefined `is_even`.

The three approaches people use to solve this are:

1. **`std::function`** - type-erased wrapper that separates declaration from definition (works but has overhead)
2. **Trampoline** - pass the other lambda as a parameter instead of capturing it (zero overhead)
3. **Deducing-this (C++23)** - self-referencing without `std::function`

To get a feel for the recursion structure, here's what a call to `is_even(4)` looks like:

```cpp
             is_even(4)
              |
              +- n==0? No -> call is_odd(3)
              |                 |
              |                 +- n==0? No -> call is_even(2)
              |                 |                 |
              |                 |                 +- ... -> is_odd(1) -> is_even(0)
              |                 |                 |                         |
              |                 |                 |                         +- n==0 -> TRUE
```

---

## Self-Assessment

### Q1: Implement mutually recursive `is_even`/`is_odd` lambdas using `std::function`

The trick with `std::function` is that it gives you a named type you can declare before you define the body. So you declare both `is_even` and `is_odd` up front as `std::function<bool(unsigned)>`, and then assign the lambda bodies afterward. At the point of assignment, both names exist in scope and can be captured by reference.

```cpp
#include <iostream>
#include <functional>

int main() {
    // std::function allows forward-referencing because the TYPE is declared
    // before the BODY is defined.
    std::function<bool(unsigned)> is_even;
    std::function<bool(unsigned)> is_odd;

    is_even = [&is_odd](unsigned n) -> bool {
        if (n == 0) return true;
        return is_odd(n - 1);
    };

    is_odd = [&is_even](unsigned n) -> bool {
        if (n == 0) return false;
        return is_even(n - 1);
    };

    std::cout << std::boolalpha;
    std::cout << "is_even(4): " << is_even(4) << "\n";
    std::cout << "is_even(7): " << is_even(7) << "\n";
    std::cout << "is_odd(3):  " << is_odd(3) << "\n";
    std::cout << "is_odd(6):  " << is_odd(6) << "\n";

    // Downside: std::function has overhead
    // - Heap allocation (type erasure)
    // - Virtual dispatch on each call
    // - Cannot be inlined
    std::cout << "\nsizeof(std::function<bool(unsigned)>): "
              << sizeof(std::function<bool(unsigned)>) << " bytes\n";
}
// Expected output:
//   is_even(4): true
//   is_even(7): false
//   is_odd(3): true
//   is_odd(6): false
//
//   sizeof(std::function<bool(unsigned)>): 32 bytes (or 64, platform-dependent)
```

It works, but every call goes through virtual dispatch and the lambdas can't be inlined. For deep recursion in performance-sensitive code, that matters.

---

### Q2: Show the trampoline / deducing-this approach that avoids `std::function` overhead

The trampoline pattern sidesteps the whole problem by not capturing the other lambda at all. Instead, each lambda accepts the other as an explicit parameter. When you call it, you pass both lambdas in - so there's no capture ordering issue and no type erasure needed.

```cpp
#include <iostream>

int main() {
    // Approach 1: Trampoline (pass lambdas as parameters)
    // Each lambda takes the OTHER lambda as its FIRST argument.
    // No std::function needed!

    auto is_even_impl = [](auto& self_even, auto& self_odd, unsigned n) -> bool {
        if (n == 0) return true;
        return self_odd(self_odd, self_even, n - 1);
    };

    auto is_odd_impl = [](auto& self_odd, auto& self_even, unsigned n) -> bool {
        if (n == 0) return false;
        return self_even(self_even, self_odd, n - 1);
    };

    // Convenience wrappers:
    auto is_even = [&](unsigned n) { return is_even_impl(is_even_impl, is_odd_impl, n); };
    auto is_odd  = [&](unsigned n) { return is_odd_impl(is_odd_impl, is_even_impl, n); };

    std::cout << std::boolalpha;
    std::cout << "is_even(10): " << is_even(10) << "\n";
    std::cout << "is_odd(10):  " << is_odd(10) << "\n";
    std::cout << "is_even(7):  " << is_even(7) << "\n";
    std::cout << "is_odd(7):   " << is_odd(7) << "\n";

    // Zero overhead: no heap, no virtual dispatch, fully inlinable!
    // sizeof wrappers = 1 (empty lambda) + pointer to captured lambdas
}
// Expected output:
//   is_even(10): true
//   is_odd(10): false
//   is_even(7): false
//   is_odd(7): true
```

The pattern takes a moment to read, but the key is that `is_even_impl` receives `self_odd` and calls `self_odd(self_odd, self_even, ...)` - it passes both lambdas forward, keeping the trampoline chain alive. The compiler can inline all of this because the types are fully known at compile time.

C++23's deducing-this feature offers another approach. It's most useful for self-recursion (a single lambda calling itself), but for mutual recursion a local struct is the cleanest solution:

```cpp
// Approach 2: Deducing-this (C++23)
// With explicit object parameter, a lambda can name itself:
#include <iostream>

int main() {
    auto is_even = [](this auto& self, unsigned n) -> bool {
        if (n == 0) return true;
        // For mutual recursion, still need reference to the other.
        // Deducing-this is most useful for SELF-recursion.
        // For mutual recursion, combine with a struct:
        return false; // placeholder
    };

    // Best C++23 approach: struct with two deducing-this lambdas
    struct {
        bool is_even(this auto& self, unsigned n) {
            return n == 0 ? true : self.is_odd(n - 1);
        }
        bool is_odd(this auto& self, unsigned n) {
            return n == 0 ? false : self.is_even(n - 1);
        }
    } parity;

    std::cout << std::boolalpha;
    std::cout << "parity.is_even(6): " << parity.is_even(6) << "\n";
    std::cout << "parity.is_odd(6):  " << parity.is_odd(6) << "\n";
}
// Expected output:
//   parity.is_even(6): true
//   parity.is_odd(6): false
```

---

### Q3: Explain why naive mutual lambda recursion without a trampoline fails

This is a useful thing to understand at a conceptual level because the error message you get from the compiler can be confusing. The reason it fails is not about permissions or visibility - it's about the fundamental fact that lambdas have unique, unnamed types that cannot be forward-declared.

```cpp
#include <iostream>

// This DOES NOT COMPILE:
/*
int main() {
    // Problem: is_odd doesn't exist yet when is_even captures it!
    auto is_even = [&is_odd](unsigned n) -> bool {  // ERROR: is_odd not declared
        if (n == 0) return true;
        return is_odd(n - 1);
    };

    auto is_odd = [&is_even](unsigned n) -> bool {
        if (n == 0) return false;
        return is_even(n - 1);
    };
}
*/

// Even if we forward-declare, auto can't be forward-declared:
/*
auto is_odd;  // ERROR: auto requires initializer
*/

int main() {
    std::cout << "=== Why naive mutual recursion fails ===\n\n";

    std::cout << "1. Lambda types are unique, unnamed closure types\n";
    std::cout << "   - Cannot forward-declare them\n";
    std::cout << "   - Cannot write: auto is_odd; (auto needs initializer)\n\n";

    std::cout << "2. Initialization order problem:\n";
    std::cout << "   auto is_even = [&is_odd]... // is_odd doesn't exist yet!\n";
    std::cout << "   auto is_odd  = [&is_even]... // too late\n\n";

    std::cout << "3. Solutions:\n";
    std::cout << "   a) std::function  - type known before body (overhead)\n";
    std::cout << "   b) Trampoline     - pass other lambda as parameter (zero overhead)\n";
    std::cout << "   c) Struct         - members can reference each other (zero overhead)\n";
    std::cout << "   d) Y-combinator   - fixed-point combinator (advanced)\n";
}
// Expected output:
//   === Why naive mutual recursion fails ===
//
//   1. Lambda types are unique, unnamed closure types
//      - Cannot forward-declare them
//      - Cannot write: auto is_odd; (auto needs initializer)
//
//   2. Initialization order problem:
//      auto is_even = [&is_odd]... // is_odd doesn't exist yet!
//      auto is_odd  = [&is_even]... // too late
//
//   3. Solutions:
//      a) std::function  - type known before body (overhead)
//      b) Trampoline     - pass other lambda as parameter (zero overhead)
//      c) Struct         - members can reference each other (zero overhead)
//      d) Y-combinator   - fixed-point combinator (advanced)
```

The reason this trips people up is that regular functions have no such problem - you can forward-declare a regular function and call it before its definition appears. Lambdas don't get that privilege because their type is the closure type, which only exists after the initializer is written.

---

## Notes

- **Performance:** `std::function` adds roughly 5-20 ns per call due to type erasure. The trampoline approach compiles to the same code as regular function calls and can be fully inlined.
- **Y-combinator:** A fixed-point combinator that enables recursion without naming: `auto Y = [](auto f) { return [=](auto... args) { return f(f, args...); }; };`
- **Stack depth:** All mutual recursion approaches are subject to stack overflow for deep recursion. Consider tail-call optimization or an iterative rewrite if the input can be large.
- **Practical use:** Mutual recursion appears naturally in parsers (expression/term/factor grammar rules), state machines, and certain tree traversal algorithms.
