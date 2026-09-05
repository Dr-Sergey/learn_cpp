# Understand the lookup rules for operator overloads and ADL interaction

**Category:** Core Language Fundamentals  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/adl>  

---

## Topic Overview

When you write `a + b`, the compiler must find the right `operator+`. This involves both **ordinary unqualified lookup** and **Argument-Dependent Lookup (ADL)**. Understanding this interaction is crucial for writing correct operator overloads.

### How Operator Lookup Works

For `a + b`, the compiler rewrites it as `operator+(a, b)` (for non-member) or `a.operator+(b)` (for member) and searches:

1. **Member lookup:** Check if `a`'s class has `operator+` as a member.
2. **Ordinary unqualified lookup:** Search enclosing scopes from the point of use outward.
3. **ADL:** Search the namespaces associated with the types of `a` and `b`.

All candidates from all three sources participate in overload resolution.

### ADL for Operators

Because of ADL, you can define your operator alongside your type in its namespace, and the compiler will find it automatically whenever that type is involved - no `using` directive needed.

```cpp
namespace math {
    struct Vector { double x, y; };

    // This operator+ is found via ADL when used with math::Vector
    Vector operator+(const Vector& a, const Vector& b) {
        return {a.x + b.x, a.y + b.y};
    }
}

int main() {
    math::Vector v1{1, 2}, v2{3, 4};
    auto v3 = v1 + v2;  // ADL finds math::operator+
    // No "using namespace math;" needed!
}
```

### Why This Matters

The same mechanism is what makes `std::cout << w` work for your own types. The `operator<<` lives in your namespace, and ADL brings it into scope because `w` is your type.

```cpp
namespace lib {
    struct Widget {};
    std::ostream& operator<<(std::ostream& os, const Widget& w) {
        return os << "Widget";
    }
}

int main() {
    lib::Widget w;
    std::cout << w;  // ADL finds lib::operator<< via Widget's namespace
}
```

---

## Self-Assessment

### Q1: Show that `a + b` looks up operator+ via ADL in the namespaces of both a and b

ADL considers the associated namespaces of every argument. Here the operator lives in `ns_b` - the namespace of the second argument - and the call still succeeds.

```cpp
#include <iostream>

namespace ns_a {
    struct TypeA { int val; };
}

namespace ns_b {
    struct TypeB { int val; };

    // operator+ defined in ns_b — found via ADL for TypeB
    int operator+(const ns_a::TypeA& a, const TypeB& b) {
        return a.val + b.val;
    }
}

int main() {
    ns_a::TypeA a{10};
    ns_b::TypeB b{20};

    // The compiler looks for operator+(TypeA, TypeB) in:
    // 1. Enclosing scope (main, global) — not found
    // 2. ADL: namespace of TypeA (ns_a) — not found
    // 3. ADL: namespace of TypeB (ns_b) — FOUND!
    int result = a + b;
    std::cout << "a + b = " << result << "\n";  // 30

    // If operator+ were in ns_a instead, it would also be found:
    // ADL searches namespaces of ALL argument types
}
```

ADL casts a wide net by design - it searches all associated namespaces so that operators can live alongside their types rather than at the global scope.

**How this works:**

- When the compiler sees `a + b`, it collects the associated namespaces of both operands.
- For `ns_a::TypeA`, the associated namespace is `ns_a`.
- For `ns_b::TypeB`, the associated namespace is `ns_b`.
- ADL searches both `ns_a` and `ns_b` for `operator+` - it finds it in `ns_b`.

### Q2: Explain why defining operator+ inside a namespace is found by ADL but not by unqualified lookup

**Answer:**

**Unqualified lookup** searches outward from the call site through enclosing scopes: the function body -> enclosing class -> enclosing namespace -> global namespace. It does NOT search unrelated namespaces.

**ADL** adds the namespaces associated with the argument types to the search.

Without ADL, every operator call would need a `using namespace math;` or a fully qualified `math::operator+(a, b)` - clearly impractical for library types.

```cpp
namespace math {
    struct Complex { double re, im; };

    Complex operator+(const Complex& a, const Complex& b) {
        return {a.re + b.re, a.im + b.im};
    }
}

namespace other {
    void test() {
        math::Complex a{1, 2}, b{3, 4};

        // Unqualified lookup from 'other::test':
        //   Searches: other -> global -> NOT math (it's unrelated)
        //   Doesn't find operator+ through ordinary lookup!

        // ADL kicks in:
        //   Arguments are math::Complex -> searches math namespace
        //   Finds math::operator+(Complex, Complex)!

        auto c = a + b;  // Works thanks to ADL

        // Without ADL, you'd need: auto c = math::operator+(a, b);
    }
}

int main() {
    other::test();
}
```

**Key insight:** Without ADL, operators defined in a library's namespace could only be used with fully qualified syntax or `using` declarations - making operator overloading impractical.

### Q3: Show a case where ADL finds an unexpected operator from an unrelated namespace

This example illustrates the subtlety: ADL only searches namespaces that are directly associated with the argument types. Including a type from another namespace does not automatically pull that namespace's operators into scope for unrelated types.

```cpp
#include <iostream>

namespace lib_a {
    struct Token { int id; };
}

namespace lib_b {
    struct Wrapper {
        lib_a::Token tok;
    };

    // Intended: operator<< for Wrapper
    std::ostream& operator<<(std::ostream& os, const Wrapper& w) {
        return os << "Wrapper(" << w.tok.id << ")";
    }

    // UNINTENDED: operator+ for Token defined in lib_b, not lib_a!
    lib_a::Token operator+(const lib_a::Token& a, const lib_a::Token& b) {
        return {a.id + b.id};
    }
}

namespace lib_a {
    // The "correct" operator+ for Token
    Token operator+(const Token& a, const Token& b) {
        return {a.id * 10 + b.id};  // Different behavior!
    }
}

int main() {
    lib_a::Token t1{1}, t2{2};

    // Which operator+ is called?
    // ADL searches: lib_a (for Token) — finds lib_a::operator+
    // But lib_b::operator+ also takes Token — is it found?

    // Actually: lib_b is NOT an associated namespace of Token
    // So ONLY lib_a::operator+ is found via ADL
    auto t3 = t1 + t2;
    std::cout << "t3.id = " << t3.id << "\n";  // 12 (lib_a version)

    // BUT if we use a lib_b::Wrapper:
    lib_b::Wrapper w1{{1}}, w2{{2}};
    // ADL for Wrapper searches lib_b -> finds lib_b::operator+ for Token too!
    // This can cause surprising results when lib_b defines operators for
    // types from lib_a
}
```

The takeaway is that the associated namespace is determined by the static type of the argument, not by which namespaces that type happens to use internally. Keep operators in the same namespace as the type they operate on, and the behavior stays predictable.

**How this works:**

- ADL only searches namespaces associated with the argument types.
- For `lib_a::Token`, only `lib_a` is searched - `lib_b`'s `operator+` is invisible.
- But if argument types involve `lib_b` types, `lib_b` enters the search - potentially finding unexpected overloads.
- This is why operators should be defined in the same namespace as the type they operate on.

---

## Notes

- Best practice: define operators in the same namespace as the types they operate on - this is the namespace ADL will search.
- Hidden friend pattern: `friend operator+` defined inside the class body - found only via ADL, not by ordinary lookup. This limits the scope of the operator.
- `std::swap` customization relies on ADL: `using std::swap; swap(a, b);` - finds your custom swap via ADL.
- C++20 changed how `operator<=>` and comparison operators are looked up, adding rewritten candidates.
- `operator<<` for output streaming should be in the type's namespace for ADL to work with `std::cout`.
