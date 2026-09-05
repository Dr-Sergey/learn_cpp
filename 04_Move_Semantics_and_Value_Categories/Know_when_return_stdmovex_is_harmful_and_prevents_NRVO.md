# Know When `return std::move(x)` Is Harmful and Prevents NRVO

**Category:** Move Semantics and Value Categories  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/copy_elision>  

---

## Topic Overview

### The Problem

`return std::move(local)` **prevents NRVO** (Named Return Value Optimization). The reason is subtle but important: NRVO works by having the compiler construct the local variable directly in the memory slot reserved for the return value at the call site. That only works when the return expression is literally the name of a local variable. The moment you wrap it in `std::move`, the expression becomes an xvalue - no longer a named local - and NRVO cannot apply.

| Return statement | Copy elision? | What happens |
| --- | :---: | --- |
| `return Widget{42};` | **Guaranteed** (URVO) | Prvalue -> direct init at call site |
| `return local;` | **NRVO likely** | Compiler constructs local in return slot |
| `return std::move(local);` | **Blocked!** | Forces a move - NRVO cannot apply |

### The Rule

**Never use `return std::move(x)` for local variables.** Plain `return x;` is **always at least as good**:

- If NRVO applies -> zero copies, zero moves (best case)
- If NRVO doesn't apply -> implicit move happens anyway (C++11+)
- `return std::move(x)` -> forces a move even when NRVO would have applied (worst case)

The reason this trips people up is that `return std::move(x)` looks like you're being helpful and efficient, but you're actually preventing the compiler from doing something even better. Trust the compiler here.

---

## Self-Assessment

### Q1: Show that `return std::move(local)` forces a move and prevents NRVO

The `Heavy` type below prints every construction, copy, and move so you can see the difference clearly.

```cpp
#include <iostream>
#include <string>

class Heavy {
    std::string data_;
public:
    Heavy(std::string d) : data_(std::move(d)) {
        std::cout << "  Construct: \"" << data_ << "\"\n";
    }
    Heavy(const Heavy& other) : data_(other.data_) {
        std::cout << "  COPY\n";
    }
    Heavy(Heavy&& other) noexcept : data_(std::move(other.data_)) {
        std::cout << "  MOVE\n";
    }
    const std::string& data() const { return data_; }
};

// GOOD: plain return -> NRVO applies -> ZERO copies/moves
Heavy good_factory() {
    Heavy h("from good_factory");
    return h;  // NRVO: h is constructed directly in caller's slot
}

// BAD: return std::move -> forces a move, kills NRVO
Heavy bad_factory() {
    Heavy h("from bad_factory");
    return std::move(h);  // Blocks NRVO! Forces move constructor
}

// ALSO GOOD: returning prvalue -> guaranteed elision (URVO)
Heavy best_factory() {
    return Heavy("from best_factory");  // Guaranteed: no copy/move
}

int main() {
    std::cout << "=== return std::move(x) vs return x ===\n\n";

    std::cout << "1. GOOD: return local (NRVO):\n";
    Heavy h1 = good_factory();
    std::cout << "  Got: \"" << h1.data() << "\"\n";

    std::cout << "\n2. BAD: return std::move(local) (NRVO prevented!):\n";
    Heavy h2 = bad_factory();
    std::cout << "  Got: \"" << h2.data() << "\"\n";

    std::cout << "\n3. BEST: return temporary (guaranteed elision):\n";
    Heavy h3 = best_factory();
    std::cout << "  Got: \"" << h3.data() << "\"\n";

    return 0;
}
// Expected output (with NRVO enabled, typical compiler):
//   1. GOOD: return local (NRVO):
//     Construct: "from good_factory"
//   2. BAD: return std::move(local) (NRVO prevented!):
//     Construct: "from bad_factory"
//     MOVE                               <- unnecessary move!
//   3. BEST: return temporary (guaranteed elision):
//     Construct: "from best_factory"
```

Look at case 2: you get a construct AND a move. With plain `return h;` in case 1, you get only the construct. The move in case 2 is pure overhead added by trying to be "helpful."

### Q2: Explain why `return local;` (without move) allows NRVO and is always at least as good

The compiler considers three scenarios when it sees `return local;`. From best to worst:

```cpp
#include <iostream>
#include <string>
#include <vector>

class Widget {
    std::vector<int> data_;
public:
    Widget(int n) : data_(n, 42) {
        std::cout << "  Construct (size=" << n << ")\n";
    }
    Widget(const Widget& o) : data_(o.data_) {
        std::cout << "  COPY (size=" << data_.size() << ")\n";
    }
    Widget(Widget&& o) noexcept : data_(std::move(o.data_)) {
        std::cout << "  MOVE (size=" << data_.size() << ")\n";
    }
};

// The three possible outcomes with "return local;":
//
// 1. NRVO applies: local is constructed directly in caller's memory
//    -> ZERO copies, ZERO moves (optimal!)
//
// 2. NRVO doesn't apply BUT implicit move eligible:
//    -> Compiler automatically treats "return local;" as "return std::move(local);"
//    -> ONE move (still good)
//
// 3. Neither applies (shouldn't happen for normal cases):
//    -> Falls back to copy

Widget case_nrvo() {
    Widget w(1000);
    // "return w;" -> NRVO: w constructed directly in caller's space
    return w;
}

Widget case_implicit_move(bool flag) {
    Widget a(100);
    Widget b(200);
    // NRVO may not apply (two possible return variables)
    // But "return a;" still gets IMPLICIT MOVE (C++11)
    if (flag) return a;
    return b;
}

int main() {
    std::cout << "=== Why 'return local;' is always best ===\n\n";

    std::cout << "Case 1: NRVO (single return path):\n";
    Widget w1 = case_nrvo();
    // Expected: just "Construct" - zero copies/moves

    std::cout << "\nCase 2: Implicit move (multiple paths):\n";
    Widget w2 = case_implicit_move(true);
    // Expected: "Construct" + "MOVE" - NRVO may not apply, but move is automatic

    std::cout << "\n=== Decision Flow ===\n";
    std::cout << "  return local;\n";
    std::cout << "    +- Can NRVO apply? -> YES -> zero cost (best)\n";
    std::cout << "    +- NRVO can't apply? -> implicit move (still good)\n";
    std::cout << "\n  return std::move(local);\n";
    std::cout << "    +- ALWAYS forces move -> NRVO impossible (suboptimal)\n";

    return 0;
}
```

The implicit move in case 2 is the fallback. The compiler is already smart enough to move when NRVO fails - you don't need to tell it to.

### Q3: List the exact conditions under which NRVO can be applied by the compiler

NRVO (Named Return Value Optimization) can be applied when ALL of these conditions are met:

```cpp
#include <iostream>
#include <string>

struct Obj {
    Obj() { std::cout << "  Default ctor\n"; }
    Obj(const Obj&) { std::cout << "  COPY\n"; }
    Obj(Obj&&) noexcept { std::cout << "  MOVE\n"; }
};

// CONDITION 1: The return expression is the name of a non-volatile
//    local variable (not a parameter, not a global)
Obj nrvo_ok() {
    Obj o;
    return o;  // named local
}

// VIOLATED: Parameter - not a local variable
Obj nrvo_fail_param(Obj o) {
    return o;  // Can't NRVO a parameter (but implicit move in C++11)
               // Note: C++23 clarifies parameters get implicit move too
}

// VIOLATED: std::move turns it into xvalue
Obj nrvo_fail_move() {
    Obj o;
    return std::move(o);  // xvalue, not a name -> no NRVO
}

// POTENTIALLY VIOLATED: Multiple return paths with different variables
Obj nrvo_maybe(bool flag) {
    Obj a, b;
    if (flag) return a;
    return b;  // Compiler can't place both a and b in return slot
}

// Single variable, multiple return points OK
Obj nrvo_ok_multi(bool flag) {
    Obj o;
    if (flag) return o;
    return o;  // Same variable in all paths
}

// VIOLATED: volatile local
Obj nrvo_fail_volatile() {
    volatile Obj o;
    // return o;  // Would fail - volatile Obj can't bind to Obj&&
    return Obj{};
}

int main() {
    std::cout << "=== NRVO Conditions ===\n\n";

    std::cout << "NRVO conditions summary:\n";
    std::cout << "  1. Return expression is name of non-volatile local object\n";
    std::cout << "  2. Type of local matches function return type (or convertible via\n";
    std::cout << "     move/copy ctor with same cv-qualification)\n";
    std::cout << "  3. Object is not caught by exception handler\n";
    std::cout << "  4. No intervening std::move()\n";
    std::cout << "  5. Compiler can prove single return variable\n";

    std::cout << "\n--- Testing ---\n\n";

    std::cout << "OK (named local, single var):\n";
    Obj o1 = nrvo_ok();

    std::cout << "\nFAIL (parameter -> implicit move):\n";
    Obj o2 = nrvo_fail_param(Obj{});

    std::cout << "\nFAIL (std::move -> forced move):\n";
    Obj o3 = nrvo_fail_move();

    std::cout << "\nMAYBE (multiple variables):\n";
    Obj o4 = nrvo_maybe(true);

    return 0;
}
```

---

## Notes

- **Golden rule:** Write `return x;` for local variables - never `return std::move(x);`.
- `return std::move(x)` is appropriate ONLY for non-local variables (members, global refs) where NRVO can't apply anyway.
- C++11/14/17 progressively expanded implicit move: function parameters, catch-clause parameters.
- C++23 further clarifies that all local variables and parameters eligible for NRVO get implicit move.
- Compilers warn about this: `-Wpessimizing-move` (GCC/Clang) flags `return std::move(local)`.
