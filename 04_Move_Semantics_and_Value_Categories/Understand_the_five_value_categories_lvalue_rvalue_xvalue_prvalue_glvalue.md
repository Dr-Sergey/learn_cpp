# Understand the Five Value Categories: lvalue, rvalue, xvalue, prvalue, glvalue

**Category:** Move Semantics and Value Categories  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/value_category>  

---

## Topic Overview

### The Value Category Taxonomy

Value categories are one of those topics that feels like taxonomy for its own sake until you realize they're what the compiler uses to decide whether to copy or move. Every C++ expression has exactly one of three primary value categories - lvalue, prvalue, or xvalue - and these three form a two-branch tree:

```cpp
         expression
        /          \
      glvalue     rvalue
     /     \     /     \
  lvalue   xvalue   prvalue
```

If the table feels like a lot, it boils down to two questions: does the expression have a stable memory address (identity), and is it safe to steal from (movable)?

| Category | Has identity? | Can be moved from? | Examples |
| --- | :---: | :---: | --- |
| **lvalue** | Yes | No | `x`, `*p`, `a[n]`, `++i`, `"hello"` |
| **xvalue** | Yes | Yes | `std::move(x)`, `static_cast<T&&>(x)`, `f()` returning `T&&` |
| **prvalue** | No | Yes | `42`, `x + 1`, `f()` returning `T`, `T{args}` |
| **glvalue** | Yes | - | lvalue or xvalue |
| **rvalue** | - | Yes | xvalue or prvalue |

### Key Rules

- **lvalue** = has a name and a persistent address (can appear on the left of `=`)
- **prvalue** = a "pure" temporary that initializes an object (no identity until materialized)
- **xvalue** = "eXpiring" value - has identity but is explicitly marked as movable
- Overload resolution uses value category to choose between copy (`const T&`) and move (`T&&`)

### Why It Matters

The value category of an expression is what drives overload resolution between copy and move. This is why `std::move` works at all - it changes the category, not the bits.

```cpp
void f(const std::string& s);  // Called for lvalues
void f(std::string&& s);       // Called for rvalues (xvalue or prvalue)

std::string name = "hello";
f(name);              // lvalue -> calls f(const string&)
f(name + "!");        // prvalue -> calls f(string&&)
f(std::move(name));   // xvalue -> calls f(string&&)
```

---

## Self-Assessment

### Q1: Classify each of the following as lvalue, prvalue, or xvalue: `x`, `x+1`, `std::move(x)`, `f()` where `f` returns `T`

The trick to classifying an expression is to remember that `decltype((expr))` encodes the answer in the reference type it returns: plain `T` means prvalue, `T&` means lvalue, `T&&` means xvalue. This macro makes that visible:

```cpp
#include <iostream>
#include <string>
#include <type_traits>

// Helper: detect value category using reference-collapsing rules
// decltype(expr) yields:
//   T    if expr is prvalue
//   T&   if expr is lvalue
//   T&&  if expr is xvalue

#define CLASSIFY(expr) \
    do { \
        using Type = decltype((expr)); \
        if constexpr (std::is_lvalue_reference_v<Type>) \
            std::cout << "  " #expr " -> lvalue\n"; \
        else if constexpr (std::is_rvalue_reference_v<Type>) \
            std::cout << "  " #expr " -> xvalue\n"; \
        else \
            std::cout << "  " #expr " -> prvalue\n"; \
    } while(0)

std::string f() { return "temporary"; }            // Returns T (prvalue)
std::string&& g(std::string&& s) { return std::move(s); }  // Returns T&& (xvalue)

int main() {
    std::cout << "=== Value Category Classification ===\n\n";

    int x = 42;
    int* p = &x;
    int arr[3] = {1, 2, 3};

    CLASSIFY(x);               // lvalue - named variable
    CLASSIFY(x + 1);           // prvalue - arithmetic result, no identity
    CLASSIFY(std::move(x));    // xvalue - cast to int&&
    CLASSIFY(f());             // prvalue - function returning T by value
    CLASSIFY(42);              // prvalue - literal
    CLASSIFY(*p);              // lvalue - dereference
    CLASSIFY(arr[0]);          // lvalue - subscript
    CLASSIFY(static_cast<int&&>(x));  // xvalue - explicit cast to T&&

    // String examples
    std::string s = "hello";
    CLASSIFY(s);               // lvalue
    CLASSIFY(s + "!");         // prvalue
    CLASSIFY(std::move(s));    // xvalue

    std::cout << "\n=== Summary Table ===\n";
    std::cout << "Expression              Category   Identity?  Movable?\n";
    std::cout << "-----------------------------------------------------\n";
    std::cout << "x                       lvalue     Yes        No\n";
    std::cout << "x + 1                   prvalue    No         Yes\n";
    std::cout << "std::move(x)            xvalue     Yes        Yes\n";
    std::cout << "f() returning T         prvalue    No         Yes\n";
    std::cout << "g() returning T&&       xvalue     Yes        Yes\n";
    std::cout << "*p                      lvalue     Yes        No\n";
    std::cout << "42                      prvalue    No         Yes\n";
    std::cout << "\"hello\"                 lvalue     Yes        No\n";

    return 0;
}
```

### Q2: Explain what an xvalue is and give two ways to produce one

An **xvalue** (eXpiring value) is an expression that:

- **Has identity** (refers to a real object in memory)
- **Can be moved from** (its resources can be transferred)

The reason xvalues exist as a separate category is that they represent something you've explicitly said is safe to steal from - they have an address, unlike prvalues, but you've consented to the theft, unlike plain lvalues. It's the compiler's signal to prefer move over copy.

**Two ways to produce an xvalue:**

```cpp
#include <iostream>
#include <string>
#include <utility>

struct Widget {
    std::string name;
    Widget(std::string n) : name(std::move(n)) {
        std::cout << "  Constructed: " << name << "\n";
    }
    Widget(Widget&& other) noexcept : name(std::move(other.name)) {
        std::cout << "  Move-constructed from xvalue\n";
    }
    Widget(const Widget& other) : name(other.name) {
        std::cout << "  Copy-constructed from lvalue\n";
    }
};

// Way 2: Function returning T&&
Widget&& xvalue_from_function(Widget&& w) {
    return std::move(w);
}

int main() {
    std::cout << "=== Two Ways to Produce xvalues ===\n\n";

    // WAY 1: std::move(x) - most common
    // std::move casts its argument to T&&, making it an xvalue
    std::cout << "Way 1: std::move(x)\n";
    Widget w1("Alpha");
    Widget w2 = std::move(w1);  // std::move(w1) is an xvalue -> move ctor selected
    std::cout << "  w1.name after move: \"" << w1.name << "\"\n\n";

    // WAY 2: Accessing a member of an rvalue
    // If the object is an rvalue, member access produces an xvalue
    std::cout << "Way 2: Member of rvalue (std::move(pair).first)\n";
    std::pair<Widget, int> p{Widget("Beta"), 42};
    Widget w3 = std::move(p).first;  // std::move(p).first is xvalue
    std::cout << "\n";

    // Additional xvalue producers:
    std::cout << "Other xvalue producers:\n";
    std::cout << "  - static_cast<T&&>(x)\n";
    std::cout << "  - a[n] where a is an rvalue array\n";
    std::cout << "  - a.m where a is rvalue and m is non-static non-reference member\n";
    std::cout << "  - a.*mp where a is rvalue and mp is pointer to data member\n";

    return 0;
}
```

### Q3: Show how value category determines whether copy or move constructor is selected

Watch how the same type gets a different constructor depending solely on the value category of the initializer expression:

```cpp
#include <iostream>
#include <string>

class Heavy {
    std::string data_;
public:
    Heavy(std::string d) : data_(std::move(d)) {}

    // Copy constructor - selected for lvalues
    Heavy(const Heavy& other) : data_(other.data_) {
        std::cout << "  COPY constructor (from lvalue)\n";
    }

    // Move constructor - selected for rvalues (xvalue or prvalue)
    Heavy(Heavy&& other) noexcept : data_(std::move(other.data_)) {
        std::cout << "  MOVE constructor (from rvalue)\n";
    }

    const std::string& data() const { return data_; }
};

Heavy make_heavy() {
    return Heavy("factory-made");  // prvalue -> move (or elided)
}

int main() {
    std::cout << "=== Value Category -> Constructor Selection ===\n\n";

    // 1. lvalue -> COPY
    std::cout << "1. Initialize from lvalue:\n";
    Heavy a("original");
    Heavy b = a;  // 'a' is lvalue -> copy ctor
    std::cout << "\n";

    // 2. xvalue (std::move) -> MOVE
    std::cout << "2. Initialize from xvalue (std::move):\n";
    Heavy c = std::move(a);  // std::move(a) is xvalue -> move ctor
    std::cout << "   a.data() after move: \"" << a.data() << "\"\n\n";

    // 3. prvalue -> MOVE (or guaranteed copy elision in C++17)
    std::cout << "3. Initialize from prvalue (temporary):\n";
    Heavy d = Heavy("temp");  // prvalue -> may be elided entirely
    std::cout << "\n";

    // 4. prvalue from function -> MOVE (or elided)
    std::cout << "4. Initialize from function return (prvalue):\n";
    Heavy e = make_heavy();
    std::cout << "\n";

    // 5. Overload resolution detail
    std::cout << "=== Resolution Rules ===\n";
    std::cout << "  lvalue:  binds to const T& (copy ctor)\n";
    std::cout << "  xvalue:  binds to T&&      (move ctor)\n";
    std::cout << "  prvalue: binds to T&&      (move ctor, or elided)\n";
    std::cout << "\n  If no move ctor exists, rvalue falls back to const T&\n";
    std::cout << "  If move ctor is deleted, rvalue binding fails (compile error)\n";

    return 0;
}
// Expected output (C++17, with copy elision):
//   1. Initialize from lvalue:
//     COPY constructor (from lvalue)
//
//   2. Initialize from xvalue (std::move):
//     MOVE constructor (from rvalue)
//     a.data() after move: ""
//
//   3. Initialize from prvalue (temporary):
//     (nothing - guaranteed copy elision)
//
//   4. Initialize from function return (prvalue):
//     (nothing - guaranteed copy elision)
```

The C++17 prvalue elision case is worth highlighting: a prvalue doesn't even create a temporary object anymore - it directly initializes the destination. That's why cases 3 and 4 print nothing.

---

## Notes

- Value categories are properties of **expressions**, not of types or objects.
- `decltype((expr))` reveals the value category: `T` = prvalue, `T&` = lvalue, `T&&` = xvalue.
- In C++17, prvalues don't create temporaries - they directly initialize the target (guaranteed copy elision).
- `std::move` doesn't move - it casts to `T&&`, changing the value category from lvalue to xvalue.
- The moved-from state must be valid (destructible + assignable) but its value is unspecified.
