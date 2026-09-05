# Understand Sequencing Rules and Evaluation Order Since C++17

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++11, C++14, C++17, C++20  
**Reference:** [cppreference - Order of evaluation](https://en.cppreference.com/w/cpp/language/eval_order)  

---

## Topic Overview

Evaluation order and sequencing rules determine **when side effects are visible** during expression evaluation. Violations of these rules are a major source of undefined behavior. The rules evolved significantly: C++03 used "sequence points," C++11 introduced "sequenced-before" as a partial order, and C++17 strengthened guarantees for several common patterns.

Here is how the model evolved across standards:

| Era | Model | Key Change |
| --- | --- | --- |
| **C++03** | Sequence points | Between full expressions, at &&, \|\|, comma, ?: |
| **C++11** | Sequenced-before (partial order) | Formalized as a relation; added for overloaded operators |
| **C++14** | (Same as C++11) | Minor clarifications |
| **C++17** | Strengthened guarantees | Postfix expressions, assignment, `<<`/`>>` chaining |
| **C++20** | (Same as C++17 for evaluation) | Additional `<=>` sequencing |

Before C++17, expressions like `f(a(), b())` where `a()` and `b()` modify shared state were hazardous: the arguments could be evaluated in any order, and interleaving was allowed. C++17 fixed several patterns. The critical distinction is between **unsequenced** (may interleave, UB if modifying same scalar), **indeterminately sequenced** (one completes before the other, but order unspecified), and **sequenced-before** (guaranteed order). Here is a summary of what C++17 nailed down:

```cpp
C++17 Sequencing Guarantees:

1. a.b           -> a is evaluated before b
2. a->b          -> a is evaluated before b
3. a->*b         -> a is evaluated before b
4. a(b1,b2,b3)   -> a is evaluated before b1,b2,b3

                    but b1,b2,b3 are indeterminately sequenced

5. b op= a       -> a is evaluated before b (right-to-left)
6. a[b]          -> a is evaluated before b
7. a << b        -> a is evaluated before b (left-to-right)
8. a >> b        -> a is evaluated before b (left-to-right)

Still UNSPECIFIED (C++17+):

- Order among function arguments: f(a(), b(), c())

  -> a(), b(), c() fully evaluated before each other starts
  -> but we don't know WHICH comes first
```

---

## Self-Assessment

### Q1: Identify which expressions are UB, unspecified, or well-defined

Let's walk through a set of classic examples and see how each one is classified. Some of these changed behavior between C++14 and C++17, which is exactly the kind of thing that bites you on old codebases:

```cpp
#include <cstdio>
#include <iostream>
#include <map>
#include <string>

int global = 0;

int inc() { return ++global; }
int get() { return global; }

void sequencing_examples() {
    int i = 0;

    // --- Example 1: Classic UB (pre-C++17) ---
    // i = i++ + ++i;
    // C++03/11/14: UB - two unsequenced modifications of i
    // C++17: STILL UB - + does not sequence its operands

    // --- Example 2: Well-defined since C++17 ---
    i = 0;
    i = ++i + 1;
    // C++17: assignment sequences right before left.
    // ++i is fully evaluated (i becomes 1), then + 1 = 2,
    // then assigned to i. Result: i == 2. Well-defined.
    std::printf("Ex2: i = %d (expected 2)\n", i);

    // --- Example 3: Chained << is left-to-right since C++17 ---
    global = 0;
    std::cout << inc() << inc() << inc() << "\n";
    // C++17: guaranteed left-to-right: prints "1 2 3"
    // Pre-C++17: unspecified order, could print any permutation

    // --- Example 4: Function arguments - indeterminately sequenced ---
    global = 0;
    auto f = [](int a, int b, int c) {
        std::printf("f(%d, %d, %d)\n", a, b, c);
    };
    f(inc(), inc(), inc());
    // C++17: each inc() completes before the next starts,
    // but the ORDER is unspecified. Could be (1,2,3) or (3,2,1) etc.
    // NOT UB - just unspecified.

    // --- Example 5: Map insertion - well-defined since C++17 ---
    std::map<int, int> m;
    i = 0;
    m[++i] = ++i;
    // C++17: right side (++i) evaluated before left side (m[++i])
    //        because op= sequences right before left.
    //        So: ++i -> i=1 (value 1), then m[++i] -> m[2] = 1
    // Pre-C++17: UB (two unsequenced modifications of i)
    std::printf("Ex5: m[2] = %d (expected 1)\n", m[2]);
}

int main() {
    sequencing_examples();
}
```

Here is the bottom line for each expression:

| Expression | Pre-C++17 | C++17+ |
| --- | --- | --- |
| `i = i++ + ++i` | UB | UB (+ is unsequenced) |
| `i = ++i + 1` | UB | **Well-defined** (= sequences R before L) |
| `cout << inc() << inc()` | Unspecified order | **Left-to-right** |
| `f(inc(), inc(), inc())` | May interleave (UB possible) | **Indeterminately sequenced** (no UB) |
| `m[++i] = ++i` | UB | **Well-defined** (= sequences R before L) |

---

### Q2: Show how C++17 fixed the "make_unique swap" idiom

This is a real-world case where sequencing rules had practical safety consequences. Before C++17, passing raw `new` expressions as function arguments could lead to memory leaks if an exception was thrown mid-evaluation. C++17's "indeterminately sequenced" guarantee - where each argument evaluates fully before the next starts - closed that gap:

```cpp
#include <cstdio>
#include <memory>
#include <stdexcept>
#include <string>

struct Logger {
    std::string name;
    Logger(std::string n) : name(std::move(n)) {
        std::printf("Logger(%s) constructed\n", name.c_str());
    }
    ~Logger() {
        std::printf("Logger(%s) destroyed\n", name.c_str());
    }
};

// Pre-C++17, this could leak if evaluation was interleaved:
void register_loggers_old(Logger* a, Logger* b) {
    std::printf("Registered: %s, %s\n", a->name.c_str(), b->name.c_str());
}

// f(new A, new B) pre-C++17:
// Possible evaluation: new A -> new B (throws) -> A leaks!
// Because arguments could be: alloc A, alloc B, construct A, construct B
// If construct B throws after alloc A, A leaks.

// C++17 fix: each argument is fully evaluated (alloc + construct)
// before the next argument's evaluation begins.
// So: (alloc A + construct A), then (alloc B + construct B)
// If construct B throws, A was already fully constructed and...
// ...still leaks! Because raw new doesn't have RAII.

// The REAL fix is to use make_unique:
void register_loggers_safe(std::unique_ptr<Logger> a,
                           std::unique_ptr<Logger> b) {
    std::printf("Registered: %s, %s\n",
                a->name.c_str(), b->name.c_str());
}

void demonstrate_c17_safety() {
    // C++17: arguments are indeterminately sequenced.
    // Each make_unique call fully completes before the next starts.
    // If the second throws, the first unique_ptr is properly destroyed.
    try {
        register_loggers_safe(
            std::make_unique<Logger>("Alpha"),
            std::make_unique<Logger>("Beta")
        );
    } catch (const std::exception& e) {
        std::printf("Exception: %s\n", e.what());
    }
}

// Chained method calls: guaranteed left-to-right in C++17
struct Builder {
    std::string result;

    Builder& add(const char* s) {
        result += s;
        return *this;
    }

    std::string build() { return result; }
};

void builder_demo() {
    int step = 0;
    auto next = [&step]() -> std::string {
        return std::to_string(++step);
    };

    Builder b;
    // C++17: b.add() is evaluated left-to-right
    // Each add() call evaluates b (left of .) before its argument
    std::string result = b.add(next().c_str())
                          .add(next().c_str())
                          .add(next().c_str())
                          .build();
    // Guaranteed: "123" in C++17
    std::printf("Builder result: %s\n", result.c_str());
}

int main() {
    demonstrate_c17_safety();
    builder_demo();
}
```

C++17 guarantees that function arguments are **indeterminately sequenced** (each fully evaluated before the next), eliminating the interleaving that caused resource leaks with `new`. Combined with `std::make_unique`, this provides exception-safe argument evaluation. Chained method calls like `a.b().c()` are guaranteed left-to-right.

---

### Q3: Remaining UB and unspecified behavior in C++20/23

Even after C++17's improvements, there is still plenty of room to go wrong. This example catalogues what is still unspecified or UB, alongside the operators that have always had well-defined sequencing:

```cpp
#include <cstdio>
#include <functional>
#include <iostream>

int counter = 0;
int next() { return ++counter; }

// Even in C++23, some evaluation orders remain unspecified or UB.

void still_unspecified() {
    // 1. Order of function argument evaluation
    counter = 0;
    auto f = [](int a, int b, int c) {
        std::printf("f(%d, %d, %d)\n", a, b, c);
    };
    f(next(), next(), next());
    // Indeterminately sequenced - any permutation of (1,2,3)

    // 2. Order of evaluation in aggregate initialization
    struct S { int a, b, c; };
    counter = 0;
    S s{next(), next(), next()};
    // C++17+: guaranteed left-to-right in braced-init-list!
    // s.a=1, s.b=2, s.c=3
    std::printf("Aggregate: {%d, %d, %d}\n", s.a, s.b, s.c);

    // 3. Designated initializers follow declaration order
    counter = 0;
    S s2{.a = next(), .b = next(), .c = next()};
    // Guaranteed left-to-right (follows member declaration order)
    std::printf("Designated: {%d, %d, %d}\n", s2.a, s2.b, s2.c);
}

void still_ub() {
    int i = 0;

    // The + operator does NOT sequence its operands.
    // Modifying the same scalar through both operands is UB.
    // int x = ++i + ++i;  // UB even in C++23

    // Comma operator DOES sequence (always has)
    i = 0;
    int y = (++i, ++i);  // Well-defined: i=1, then i=2, y=2
    std::printf("Comma: y = %d (expected 2)\n", y);

    // Logical operators sequence (always have)
    i = 0;
    bool b = (++i == 1) && (++i == 2);  // Well-defined
    std::printf("Logical: b = %d (expected 1), i = %d (expected 2)\n", b, i);

    // Ternary sequences the condition before the chosen branch
    i = 0;
    int z = (++i == 1) ? ++i : ++i;  // Well-defined: i=1 (true), i=2
    std::printf("Ternary: z = %d (expected 2)\n", z);
}

/*
Summary Table: Sequencing in C++17/20/23

| Expression          | Sequencing              | Notes                  |
| --- | --- | --- |
| a + b               | Unsequenced             | UB if both modify same |
| a = b               | R sequenced before L    | C++17                  |
| a << b              | L sequenced before R    | C++17                  |
| f(a, b, c)          | Indeterminately seq.    | Not interleaved        |
| {a, b, c}           | Left to right           | Braced-init-list       |
| a && b              | L sequenced before R    | Short-circuit          |
| a || b              | L sequenced before R    | Short-circuit          |
| a, b                | L sequenced before R    | Comma operator         |
| a ? b : c           | a before chosen branch  | Ternary                |
*/

int main() {
    still_unspecified();
    still_ub();
}
```

Even in C++23, the order of function argument evaluation is unspecified, and modifying the same scalar in two unsequenced operands of arithmetic operators is still UB. Braced-init-lists are left-to-right since C++17. Logical operators, comma, and ternary have always sequenced their operands.

---

## Notes

- **C++17 was the watershed:** it fixed `<<`/`>>` chaining, assignment order, postfix expression evaluation, and function argument interleaving.
- The `i = ++i + 1` idiom went from UB to well-defined in C++17 - but relying on this is poor style. Use separate statements.
- Function argument evaluation order is **intentionally** left unspecified to allow compiler optimization (register allocation, instruction scheduling).
- `-Wsequence-point` (GCC) and `-Wunsequenced` (Clang) warn about modifications to the same variable in unsequenced operations.
- When in doubt, extract expressions with side effects into named temporaries. This eliminates all sequencing concerns and improves readability.
- Overloaded operators follow function-call sequencing rules, not built-in operator rules. `a + b` with overloaded `+` sequences `a` and `b` as function arguments (indeterminately sequenced).
