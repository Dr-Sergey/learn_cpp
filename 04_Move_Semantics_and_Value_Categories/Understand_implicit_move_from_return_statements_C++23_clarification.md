# Understand Implicit Move from Return Statements (C++23 Clarification)

**Category:** Move Semantics and Value Categories  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/return>  

---

## Topic Overview

### The Evolution of Implicit Move

The rule for when `return x;` performs a move instead of a copy has been tightened up with every standard since C++11. Each version fixed a category of cases that the previous version handled suboptimally. C++23 finishes the job with a cleaner, unified rule.

| Standard | Implicit move applies to | Example |
| --- | --- | --- |
| **C++11** | Named local variables of same type | `return local;` |
| **C++14** | Same as C++11 | - |
| **C++17** | + catch clause parameters | `catch(Exc e) { ... return e; }` |
| **C++20** | + implicit conversion via move ctor | `return local;` where conversion needed |
| **C++23** | **Any id-expression that is move-eligible** | Parameters, structured bindings, more |

### C++23 Rule (P2266)

In C++23, `return x;` performs an **implicit move** if `x` is a **move-eligible** id-expression. An expression is move-eligible if:

1. It names an implicitly-movable entity: a non-volatile object or rvalue reference
2. The entity is declared in the body of the innermost enclosing function/lambda
3. It's not a function parameter declared with the `register` storage class (rare)

The key change: **rvalue references to types different from the return type** are now move-eligible.

The reason this matters in practice: before C++23, a named rvalue reference parameter like `std::unique_ptr<int>&& p` was treated as an lvalue inside the function body (because it has a name). So `return p;` would try to copy it - which fails for move-only types. You had to write `return std::move(p);`. C++23 fixes this inconsistency: if the parameter is an rvalue reference, it is move-eligible in a return statement, and the compiler treats it as an rvalue automatically.

---

## Self-Assessment

### Q1: Explain how C++23 requires implicit move from any rvalue-eligible expression in a return statement

This example walks through each case that C++23 handles - from the basic local-variable move (C++11) through the rvalue-reference parameter fix (C++23). The `from_rvalue_ref` function is the one that actually required a code change across standards:

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <utility>

struct Widget {
    std::string data;
    Widget(std::string d) : data(std::move(d)) {
        std::cout << "  Construct: \"" << data << "\"\n";
    }
    Widget(const Widget& o) : data(o.data) {
        std::cout << "  COPY: \"" << data << "\"\n";
    }
    Widget(Widget&& o) noexcept : data(std::move(o.data)) {
        std::cout << "  MOVE: \"" << data << "\"\n";
    }
};

// C++11: Local variable -> implicit move
Widget from_local() {
    Widget w("local");
    return w;  // Implicit move (or NRVO)
}

// C++17: Catch parameter -> implicit move
Widget from_catch() {
    try {
        throw Widget("caught");
    } catch (Widget w) {
        return w;  // C++17: implicit move from catch parameter
    }
    return Widget("fallback");
}

// C++23 NEW: Function parameter -> always move-eligible
// (C++11/14/17 already allowed this for same-type, but C++23 formalizes the rule)
Widget from_param(Widget w) {
    return w;  // C++23: param is move-eligible -> implicit move
}

// C++23 NEW: rvalue reference parameter -> move-eligible
// Previously this was a pitfall -- the named rvalue ref was treated as lvalue
std::unique_ptr<int> from_rvalue_ref(std::unique_ptr<int>&& p) {
    return p;  // C++23: p is rvalue-ref -> move-eligible -> implicit move!
    // Pre-C++23: This would FAIL -- p is lvalue (it's named) -> tried to copy unique_ptr
    // Pre-C++23 fix: return std::move(p);
}

int main() {
    std::cout << "=== C++23 Implicit Move ===\n\n";

    std::cout << "1. From local variable:\n";
    auto w1 = from_local();

    std::cout << "\n2. From catch parameter (C++17+):\n";
    auto w2 = from_catch();

    std::cout << "\n3. From function parameter:\n";
    auto w3 = from_param(Widget("param"));

    std::cout << "\n4. From rvalue reference parameter (C++23 fix):\n";
    auto p = std::make_unique<int>(42);
    auto p2 = from_rvalue_ref(std::move(p));
    std::cout << "  Got value: " << *p2 << "\n";

    return 0;
}
```

### Q2: Show a C++20 case where implicit move failed and C++23 fixes it

The two most practically important C++23 fixes are: returning a derived type as a base type (conversion case), and returning a named rvalue reference parameter (the move-only type case). Here both are shown side by side so you can see what the pre-C++23 problem looked like:

```cpp
#include <iostream>
#include <memory>
#include <string>

// CASE 1: Returning a move-only type through conversion
// In C++20, returning via conversion constructor didn't always get implicit move

struct Base {
    std::string name;
    Base(std::string n) : name(std::move(n)) {
        std::cout << "  Base(\"" << name << "\")\n";
    }
    Base(Base&& o) noexcept : name(std::move(o.name)) {
        std::cout << "  Base MOVE(\"" << name << "\")\n";
    }
    Base(const Base& o) : name(o.name) {
        std::cout << "  Base COPY(\"" << name << "\")\n";
    }
};

struct Derived : Base {
    Derived(std::string n) : Base(std::move(n)) {}
    Derived(Derived&& o) noexcept : Base(std::move(o)) {
        std::cout << "  Derived MOVE\n";
    }
};

// C++20 BUG: Returning Derived as Base with implicit conversion
// The two-phase overload resolution in C++20 could fail:
// Phase 1: Try move -- but Derived&& -> Base requires conversion, may fail
// Phase 2: Fall back to copy -- copies instead of moves!
Base return_derived_as_base() {
    Derived d("hello");
    return d;
    // C++20: Might COPY (conversion from Derived to Base uses copy)
    // C++23: Always MOVES (d is move-eligible, treated as rvalue in overload resolution)
}

// CASE 2: Rvalue reference parameter (the biggest C++23 fix)
std::unique_ptr<int> relay(std::unique_ptr<int>&& p) {
    // C++20: This would NOT compile without std::move(p)!
    //        p is a named parameter -> lvalue -> can't copy unique_ptr
    // C++23: p is move-eligible -> implicit move -> compiles!
    return p;
    // Pre-C++23 workaround: return std::move(p);
}

// CASE 3: Structured binding (C++23 potential future direction)
// Not yet standardized but under discussion

int main() {
    std::cout << "=== C++20 -> C++23 Implicit Move Fixes ===\n\n";

    std::cout << "Case 1: Derived -> Base implicit move:\n";
    Base b = return_derived_as_base();
    std::cout << "  Got: \"" << b.name << "\"\n";

    std::cout << "\nCase 2: rvalue reference parameter:\n";
    auto p = std::make_unique<int>(42);
    auto p2 = relay(std::move(p));
    std::cout << "  Got: " << *p2 << "\n";

    std::cout << "\n=== Summary ===\n";
    std::cout << "  C++20 pitfalls fixed in C++23:\n";
    std::cout << "  1. Named rvalue ref params: now move-eligible\n";
    std::cout << "  2. Conversion returns: move applied before fallback to copy\n";
    std::cout << "  3. Simpler mental model: 'return x;' moves if x is local/param\n";

    return 0;
}
```

The `relay` function is the clearest example of the improvement: before C++23, `return p;` on a `unique_ptr&&` parameter would not compile because `p` was treated as an lvalue. You had to remember to write `std::move(p)`. In C++23, the compiler figures it out for you.

### Q3: List the conditions under which NRVO is allowed but not guaranteed

NRVO is the compiler's ability to construct a named local variable directly in the caller's return slot, skipping the copy or move entirely. The standard permits it under a specific set of conditions - and when those conditions do not hold, the fallback is implicit move (not copy):

```cpp
#include <iostream>
#include <string>

struct Obj {
    std::string s;
    Obj(std::string v) : s(std::move(v)) {
        std::cout << "  Ctor: \"" << s << "\"\n";
    }
    Obj(const Obj& o) : s(o.s) { std::cout << "  COPY\n"; }
    Obj(Obj&& o) noexcept : s(std::move(o.s)) { std::cout << "  MOVE\n"; }
};

// NRVO allowed: single named local, same type as return
Obj case1() { Obj o("case1"); return o; }

// NRVO allowed: same variable on all paths
Obj case2(bool flag) {
    Obj o("case2");
    if (flag) return o;
    return o;  // Same variable
}

// NRVO HARDER: different variables on different paths
Obj case3(bool flag) {
    Obj a("a"), b("b");
    return flag ? a : b;  // Compiler may not apply NRVO
}

// NRVO NOT ALLOWED: function parameter
Obj case4(Obj param) {
    return param;  // Not eligible for NRVO (but gets implicit move)
}

int main() {
    std::cout << "=== NRVO Conditions ===\n\n";

    std::cout << "NRVO is ALLOWED (not guaranteed) when ALL hold:\n\n";
    std::cout << "  1. Return type is non-volatile class type\n";
    std::cout << "  2. Expression is name of non-volatile automatic-storage object\n";
    std::cout << "  3. Object type is same as (or cv-compatible with) return type\n";
    std::cout << "  4. Object is NOT a function parameter\n";
    std::cout << "  5. Object is NOT a catch-clause parameter\n";
    std::cout << "  6. Object is NOT part of a structured binding\n\n";

    std::cout << "NRVO is TYPICALLY APPLIED when (implementation-specific):\n\n";
    std::cout << "  7. Single variable is returned on all paths (compiler can place\n";
    std::cout << "     the variable directly in the return slot at construction time)\n";
    std::cout << "  8. No address-of operation that requires specific placement\n\n";

    std::cout << "--- Testing ---\n\n";

    std::cout << "Case 1 (simple local):\n";
    auto o1 = case1();   // Likely NRVO

    std::cout << "\nCase 2 (same var, multi-path):\n";
    auto o2 = case2(true);  // Likely NRVO

    std::cout << "\nCase 3 (different vars):\n";
    auto o3 = case3(true);  // NRVO harder, may see MOVE

    std::cout << "\nCase 4 (parameter):\n";
    auto o4 = case4(Obj("param"));  // No NRVO, implicit move

    std::cout << "\nFallback chain: NRVO -> implicit move -> copy\n";

    return 0;
}
```

The fallback chain at the bottom is the key takeaway: the compiler tries NRVO first, then implicit move, then copy as a last resort. Writing `return x;` (without `std::move`) always gives the compiler its best shot at NRVO. Adding `std::move` blocks NRVO, trading the optimization opportunity for a guaranteed move.

---

## Notes

- C++23 simplifies the rule: any named local/parameter is **move-eligible** in a `return` statement.
- Before C++23, `return rvalue_ref_param;` for move-only types required explicit `std::move()`.
- NRVO is always at least as good as implicit move - write `return x;` and let the compiler optimize.
- The `-Wpessimizing-move` and `-Wredundant-move` compiler warnings help catch unnecessary `std::move` in returns.
- `co_return` in coroutines follows the same implicit move rules as `return`.
