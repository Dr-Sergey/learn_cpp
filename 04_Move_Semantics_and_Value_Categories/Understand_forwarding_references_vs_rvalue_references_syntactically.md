# Understand Forwarding References vs Rvalue References Syntactically

**Category:** Move Semantics and Value Categories  
**Standard:** C++11 (concepts/requires: C++20)  
**Reference:** <https://en.cppreference.com/w/cpp/language/reference>  

---

## Topic Overview

### The Syntactic Ambiguity

Here is a genuinely confusing thing about C++: `T&&` can mean two completely different things depending on whether `T` is being deduced right now, at the point of this function call. If `T` is deduced, you have a forwarding reference that can bind to anything. If `T` is already known (concrete), you have a plain rvalue reference that only binds to rvalues.

The reason this trips people up is that both look identical in source code. You have to ask: "Is this `T` being deduced by this specific call?" If yes - forwarding reference. If no - rvalue reference.

| Syntax | Context | Type | What it is |
| --- | --- | --- | --- |
| `void f(T&& x)` | T is a deduced template param | Deduced | **Forwarding reference** |
| `void f(Widget&& x)` | Widget is a concrete type | Fixed | **Rvalue reference** |
| `auto&& x = expr` | auto is deduced | Deduced | **Forwarding reference** |
| `void f(vector<T>&& x)` | Even if T is deduced | Fixed (vector part) | **Rvalue reference** |
| `void C::f(T&& x)` | T is a class template param, not deduced by f | Fixed | **Rvalue reference** |

### Quick Rule

**Forwarding reference** = `T&&` where `T` is deduced **in that function call**. Everything else is an rvalue reference.

---

## Self-Assessment

### Q1: Distinguish `void f(T&& x)` (forwarding ref) from `void f(Widget&& x)` (rvalue ref)

This example puts all the cases side by side so you can see exactly which situations give you a forwarding reference and which do not. Pay close attention to `Container::push` - even though `T` looks like a template parameter, it was fixed when you instantiated the class, so it is not deduced by `push` itself:

```cpp
#include <iostream>
#include <string>
#include <type_traits>

struct Widget {
    int val;
    Widget(int v) : val(v) {}
};

// FORWARDING REFERENCE: T is deduced from the argument
template <typename T>
void forwarding(T&& x) {
    if constexpr (std::is_lvalue_reference_v<T>)
        std::cout << "  forwarding(T&&): received LVALUE (T = "
                  << typeid(T).name() << "&)\n";
    else
        std::cout << "  forwarding(T&&): received RVALUE (T = "
                  << typeid(T).name() << ")\n";
}

// RVALUE REFERENCE: Widget is concrete -- only accepts rvalues
void rvalue_only(Widget&& w) {
    std::cout << "  rvalue_only(Widget&&): val = " << w.val << "\n";
}

// Also rvalue reference -- even though T is a template parameter,
// it's already determined at class level, not deduced by this function
template <typename T>
struct Container {
    void push(T&& x) {  // NOT a forwarding reference! T is fixed
        std::cout << "  Container::push(T&&) - rvalue ref only\n";
    }
};

int main() {
    std::cout << "=== Forwarding ref vs Rvalue ref ===\n\n";

    Widget w{42};

    // Forwarding reference -- accepts BOTH lvalues and rvalues
    std::cout << "forwarding(T&&) - accepts everything:\n";
    forwarding(w);              // OK: T = Widget&, T&& = Widget&
    forwarding(Widget{1});      // OK: T = Widget, T&& = Widget&&
    forwarding(std::move(w));   // OK: T = Widget, T&& = Widget&&

    // Rvalue reference -- accepts ONLY rvalues
    std::cout << "\nrvalue_only(Widget&&) - rvalues only:\n";
    // rvalue_only(w);          // ERROR: cannot bind lvalue to Widget&&
    rvalue_only(Widget{2});     // OK: temporary is rvalue
    rvalue_only(std::move(w));  // OK: xvalue is rvalue

    // Container::push -- NOT a forwarding reference
    std::cout << "\nContainer<Widget>::push(T&&) - rvalue ref:\n";
    Container<Widget> c;
    // c.push(w);               // ERROR: T is Widget, not deduced
    c.push(Widget{3});          // OK: rvalue

    // auto&& -- IS a forwarding reference
    std::cout << "\nauto&& - forwarding reference:\n";
    auto&& r1 = w;              // auto = Widget& -> Widget&
    auto&& r2 = Widget{4};     // auto = Widget -> Widget&&
    std::cout << "  r1 is lvalue ref: " << std::is_lvalue_reference_v<decltype(r1)> << "\n";
    std::cout << "  r2 is rvalue ref: " << std::is_rvalue_reference_v<decltype(r2)> << "\n";

    return 0;
}
```

Notice that `r1` binds to a named variable (lvalue), so `auto` deduces to `Widget&` and `auto&&` collapses to `Widget&`. `r2` binds to a temporary (rvalue), so `auto` deduces to `Widget` and `auto&&` becomes `Widget&&`. This is reference collapsing in action.

### Q2: Show a failed attempt to overload on forwarding reference and explain why it fails

The forwarding reference template is "too greedy" - it can produce an exact match for almost any argument type, including types you intended to go elsewhere. Here you can see it stealing the non-const copy case right out from under the proper copy constructor:

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <utility>

class Person {
    std::string name_;
public:
    // Forwarding reference constructor -- TOO GREEDY!
    template <typename T>
    Person(T&& n) : name_(std::forward<T>(n)) {
        std::cout << "  Template ctor: " << name_ << "\n";
    }

    // Copy constructor -- should be called for lvalue Person
    Person(const Person& other) : name_(other.name_) {
        std::cout << "  Copy ctor: " << name_ << "\n";
    }

    const std::string& name() const { return name_; }
};

int main() {
    std::cout << "=== Forwarding Reference Overload Problem ===\n\n";

    // This works fine
    std::cout << "1. String rvalue:\n";
    Person p1(std::string("Alice"));  // T = string -> template ctor

    // This fails -- template ctor is a BETTER match than copy ctor!
    std::cout << "\n2. Copying non-const Person:\n";
    Person p2(p1);  // BUG! Calls template ctor with T = Person&
    // The template ctor deduces T = Person& -> exact match
    // Copy ctor requires const Person& -> requires const conversion
    // Template wins -> tries to construct string from Person -> ERROR or wrong behavior!

    // const Person works because const Person& is exact match for copy ctor
    std::cout << "\n3. Copying const Person:\n";
    const Person p3("Bob");
    Person p4(p3);  // OK: const Person& matches copy ctor exactly

    return 0;
}
// Problem: The forwarding reference template is a BETTER match than
// the copy constructor for non-const lvalues because it deduces an
// exact match (Person&) while copy ctor requires adding const.
//
// Solution: Constrain the template (see Q3)
```

The copy constructor requires a `const Person&`, which means adding `const` - a small implicit conversion. The template deduces `T = Person&` for an exact match. Exact match beats the implicit const-add, so the template wins. Always constrain forwarding reference constructors.

### Q3: Write a function that uses `requires` (C++20) to constrain a forwarding reference

The fix is to exclude the template from the overload set when the argument is already the same type (or cv/ref variation of it). C++20 `requires` and `std::same_as<std::remove_cvref_t<T>, PersonFixed>` is the clean way to do that. Notice the `!` negation - the constraint says "only enable this template when T is NOT a PersonFixed":

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <concepts>

class PersonFixed {
    std::string name_;
public:
    // C++20 SOLUTION: Constrain with requires to exclude Person-like types
    template <typename T>
        requires (!std::same_as<std::remove_cvref_t<T>, PersonFixed>)
              && std::constructible_from<std::string, T>
    PersonFixed(T&& n) : name_(std::forward<T>(n)) {
        std::cout << "  Template ctor: \"" << name_ << "\"\n";
    }

    // Copy constructor -- now wins for PersonFixed arguments
    PersonFixed(const PersonFixed& other) : name_(other.name_) {
        std::cout << "  Copy ctor: \"" << name_ << "\"\n";
    }

    // Move constructor
    PersonFixed(PersonFixed&& other) noexcept : name_(std::move(other.name_)) {
        std::cout << "  Move ctor: \"" << name_ << "\"\n";
    }

    const std::string& name() const { return name_; }
};

// C++17 alternative using enable_if:
class PersonFixed17 {
    std::string name_;
public:
    template <typename T,
              std::enable_if_t<
                  !std::is_same_v<std::decay_t<T>, PersonFixed17> &&
                  std::is_constructible_v<std::string, T>,
              int> = 0>
    PersonFixed17(T&& n) : name_(std::forward<T>(n)) {
        std::cout << "  [C++17] Template ctor: \"" << name_ << "\"\n";
    }

    PersonFixed17(const PersonFixed17& other) : name_(other.name_) {
        std::cout << "  [C++17] Copy ctor: \"" << name_ << "\"\n";
    }
};

int main() {
    std::cout << "=== Constrained Forwarding Reference (C++20) ===\n\n";

    // String rvalue -> template ctor
    std::cout << "1. From string rvalue:\n";
    PersonFixed p1(std::string("Alice"));

    // Non-const copy -> copy ctor (template excluded by constraint!)
    std::cout << "\n2. Non-const copy (now works!):\n";
    PersonFixed p2(p1);  // Copy ctor wins -- template excluded

    // Move -> move ctor
    std::cout << "\n3. Move:\n";
    PersonFixed p3(std::move(p2));

    // const copy -> copy ctor
    std::cout << "\n4. Const copy:\n";
    const PersonFixed p4("Bob");
    PersonFixed p5(p4);

    // C-string -> template ctor (convertible to string)
    std::cout << "\n5. From C-string literal:\n";
    PersonFixed p6("Charlie");

    // C++17 enable_if version
    std::cout << "\n=== C++17 enable_if version ===\n";
    PersonFixed17 q1("Delta");
    PersonFixed17 q2(q1);  // Copy ctor, not template

    return 0;
}
```

The `std::remove_cvref_t<T>` trait strips all references and cv-qualifiers before the comparison, so `PersonFixed&`, `const PersonFixed&`, and `PersonFixed&&` all correctly match the constraint and get excluded. That is why C++20 prefers `remove_cvref_t` here over the older `std::decay_t`.

---

## Notes

- `T&&` is a forwarding reference **only** when `T` is deduced by that specific function call.
- `vector<T>&&`, `Container<T>::f(T&&)` - these are **rvalue references**, not forwarding.
- `auto&&` in range-for, lambdas, and variable declarations is a forwarding reference.
- Forwarding reference constructors are too greedy - always constrain them with `requires` or `enable_if`.
- C++20 `std::remove_cvref_t` is the right trait to strip references and cv-qualifiers for constraints.
