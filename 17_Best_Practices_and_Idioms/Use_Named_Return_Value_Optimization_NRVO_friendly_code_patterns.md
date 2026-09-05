# Use Named Return Value Optimization (NRVO) friendly code patterns

**Category:** Best Practices and Idioms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/copy_elision>  

---

## Topic Overview

**NRVO** allows the compiler to construct a named local variable directly in the caller's return slot, eliminating the copy/move entirely. Unlike RVO (prvalue), NRVO is **not guaranteed** by the standard but is performed by all major compilers.

The distinction between RVO and NRVO is worth understanding clearly. RVO applies when you return a temporary directly - `return Widget{};`. C++17 mandates this elision. NRVO applies when you return a *named* local variable - `Widget w; ...; return w;`. Compilers do this in practice, but the standard does not require it, so you need to write your code to make it easy for the compiler to apply.

### RVO vs NRVO

```asm
RVO (guaranteed C++17):       NRVO (optional, but universal):
return Widget{};              Widget w;
                              w.setup();
                              return w;
Compiler MUST elide copy.     Compiler usually elides copy.
```

---

## Self-Assessment

### Q1: Show NRVO in action with a named local variable

The instrumented `Widget` class below prints on every construction, copy, move, and destruction. With NRVO working, you should see only one construction - no copy or move at all.

```cpp
#include <iostream>
#include <string>

class Widget {
    std::string name_;
public:
    Widget(std::string name) : name_(std::move(name)) {
        std::cout << "Construct: " << name_ << '\n';
    }
    Widget(const Widget& other) : name_(other.name_) {
        std::cout << "Copy: " << name_ << '\n';
    }
    Widget(Widget&& other) noexcept : name_(std::move(other.name_)) {
        std::cout << "Move: " << name_ << '\n';
    }
    ~Widget() {
        std::cout << "Destroy: " << name_ << '\n';
    }
};

// NRVO: named local variable returned by value
Widget create_widget() {
    Widget w("NRVO");  // constructed directly in caller's space
    return w;           // NRVO elides the copy/move
}

// RVO: prvalue (guaranteed in C++17)
Widget create_rvo() {
    return Widget("RVO");  // guaranteed: no copy/move
}

int main() {
    std::cout << "--- NRVO ---\n";
    Widget w1 = create_widget();  // only 1 construction, no copy/move
    std::cout << "--- RVO ---\n";
    Widget w2 = create_rvo();     // guaranteed: 1 construction
    std::cout << "--- end ---\n";
}
// Expected output (with NRVO):
// --- NRVO ---
// Construct: NRVO
// --- RVO ---
// Construct: RVO
// --- end ---
// Destroy: RVO
// Destroy: NRVO
```

The object is built directly into `w1`'s storage - `create_widget`'s local variable `w` and the caller's `w1` are the same memory location.

### Q2: Show when multiple return paths prevent NRVO and how to fix it

NRVO requires the compiler to know at compile time *which* named variable will end up in the return slot. When you have multiple return paths returning different variables, that determination is impossible and NRVO is defeated.

```cpp
#include <iostream>
#include <string>

class Heavy {
    std::string data_;
public:
    Heavy(std::string s) : data_(std::move(s)) {
        std::cout << "Construct: " << data_ << '\n';
    }
    Heavy(const Heavy& o) : data_(o.data_) {
        std::cout << "COPY: " << data_ << '\n';  // we don't want this!
    }
    Heavy(Heavy&& o) noexcept : data_(std::move(o.data_)) {
        std::cout << "Move: " << data_ << '\n';
    }
};

// BAD: two different named variables -> NRVO prevented
Heavy create_bad(bool flag) {
    Heavy a("A");
    Heavy b("B");
    if (flag) return a;  // which variable gets the return slot?
    return b;            // compiler can't decide at compile time
    // Result: copy or move (not elided)
}

// GOOD: single named variable -> NRVO works
Heavy create_good(bool flag) {
    Heavy result(flag ? "A" : "B");  // single variable
    return result;  // NRVO applies
}

// ALSO GOOD: construct in branches, return immediately
Heavy create_also_good(bool flag) {
    if (flag) return Heavy("A");  // RVO (prvalue)
    return Heavy("B");            // RVO (prvalue)
}

int main() {
    std::cout << "--- bad (NRVO prevented) ---\n";
    Heavy h1 = create_bad(true);
    std::cout << "--- good (NRVO) ---\n";
    Heavy h2 = create_good(true);
    std::cout << "--- also good (RVO) ---\n";
    Heavy h3 = create_also_good(true);
}
// Typical output:
// --- bad (NRVO prevented) ---
// Construct: A
// Construct: B
// Move: A    <-- extra move!
// --- good (NRVO) ---
// Construct: A  <-- no copy/move
// --- also good (RVO) ---
// Construct: A  <-- no copy/move
```

The `create_also_good` pattern is worth remembering: if you can return prvalues directly from each branch, you get guaranteed RVO (C++17) rather than the optional NRVO.

### Q3: Explain how guaranteed copy elision (C++17) differs from NRVO

The practical difference that really matters: guaranteed RVO works even when the copy and move constructors are deleted. NRVO does not - the compiler needs a valid copy or move constructor as a fallback in case it cannot apply the optimization.

| Feature | Guaranteed copy elision (RVO) | NRVO |
| --- | --- | --- |
| Standard | C++17 mandatory | Optional (all compilers do it) |
| Applies to | Prvalues (`return Widget{}`) | Named variables (`Widget w; return w;`) |
| Can fail? | Never (guaranteed) | Yes (multiple return paths) |
| Requires copy/move ctor? | No! Works with deleted copy/move | Yes (must be available even if elided) |
| Example | `return std::string("hi")` | `std::string s = "hi"; return s;` |

```cpp
#include <iostream>

class NoCopy {
public:
    NoCopy() { std::cout << "Construct\n"; }
    NoCopy(const NoCopy&) = delete;
    NoCopy(NoCopy&&) = delete;
};

NoCopy create_guaranteed() {
    return NoCopy{};  // OK in C++17! Guaranteed copy elision.
    // No copy/move ctor needed because the prvalue is
    // constructed directly in the destination.
}

// NoCopy create_nrvo() {
//     NoCopy nc;     // named variable
//     return nc;     // ERROR: copy/move ctor is deleted!
//     // NRVO is optional, so the compiler must have
//     // a valid copy/move ctor as fallback.
// }

int main() {
    NoCopy x = create_guaranteed();  // works!
}
// Expected output:
// Construct
```

This is why move-only types like `std::unique_ptr` work fine as return values - they rely on guaranteed RVO (prvalue elision), not NRVO.

---

## Notes

- **NRVO-friendly pattern:** single named variable, single return path.
- Don't `std::move` the return value: `return std::move(w);` **prevents** NRVO!
- `-fno-elide-constructors` disables NRVO for testing (but NOT guaranteed RVO in C++17).
- GCC/Clang's `-Wpessimizing-move` warns when `std::move` prevents elision.
