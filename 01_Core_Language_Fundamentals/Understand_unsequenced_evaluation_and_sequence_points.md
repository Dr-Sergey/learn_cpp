# Understand unsequenced evaluation and sequence points

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/eval_order>  

---

## Topic Overview

C++ allows the compiler to evaluate subexpressions in any order unless the standard specifies a **sequencing relationship**. Writing expressions where two unsequenced evaluations modify the same object - or one modifies and the other reads - is **undefined behavior (UB)**. This is one of those topics where the examples look almost reasonable but hide real danger.

### Sequencing Terminology

| Relationship | Meaning |
| --- | --- |
| **Sequenced before** | A is fully complete before B starts |
| **Sequenced after** | B happens after A |
| **Indeterminately sequenced** | Either A before B or B before A, but not interleaved |
| **Unsequenced** | No ordering - may overlap. Modifying same scalar = **UB** |

### Classic UB Examples (Pre-C++17)

These expressions look like they should have a clear meaning, but they don't - the compiler is free to evaluate the subexpressions in any order, and modifying a variable twice without sequencing is undefined:

```cpp
int i = 0;
i = i++ + ++i;       // UB: i modified multiple times, unsequenced
f(i++, i++);          // UB before C++17: arguments are unsequenced
a[i] = i++;           // UB: reading i in a[i] and modifying i in i++
std::cout << i++ << i++;  // UB before C++17
```

### C++17 Sequencing Improvements

C++17 tightened the rules significantly, fixing several common cases. If the table feels like a lot, the practical takeaway is: C++17 fixed chained `<<`, assignment, and compound assignment - but function argument order is still unspecified:

| Expression | Before C++17 | C++17+ |
| --- | --- | --- |
| `f(a, b)` | `a`, `b` unsequenced | `a`, `b` **indeterminately sequenced** |
| `a.b(c)` | No ordering guaranteed | `a` sequenced before `c` |
| `a = b` | `b` before `a`, but side effects unspecified | `b` sequenced before `a` |
| `a op= b` | Similar issues | `b` sequenced before `a` |
| `a << b << c` | Unsequenced | Left-to-right (chaining fixed) |
| `a[b]` | Unsequenced | `b` sequenced before subscript |

**Key C++17 change:** Function arguments are **indeterminately sequenced** (not interleaved), so `f(i++, i++)` is still unspecified order but no longer UB. However, it's terrible style.

---

## Self-Assessment

### Q1: Explain precisely why `f(x++, x++)` is UB before C++17

**Answer:**

Before C++17, function arguments are **unsequenced** relative to each other. The standard says:

> If a side effect on a scalar object is unsequenced relative to another side effect on the same scalar object, the behavior is undefined.

Watch what happens step by step - because the two increments have no ordering, the compiler could interleave them at the instruction level, and the result is anyone's guess:

```cpp
#include <iostream>

void f(int a, int b) {
    std::cout << "a=" << a << " b=" << b << "\n";
}

int main() {
    int x = 0;

    // Before C++17: x++ and x++ are UNSEQUENCED
    // Both modify x (side effect) without sequencing -> UB
    // f(x++, x++);  // UB before C++17!

    // In C++17: arguments are INDETERMINATELY sequenced
    // So one x++ completes before the other starts
    // Result is either f(0,1) or f(1,0) - unspecified order, but NOT UB

    // Safe alternative (all standards):
    int a = x++;
    int b = x++;
    f(a, b);  // Always f(0, 1)
}
```

**Analysis:**

| Standard | `f(x++, x++)` | Reason |
| --- | --- | --- |
| C++11/14 | **Undefined behavior** | Arguments unsequenced; two modifications of x |
| C++17+ | **Unspecified order** (valid) | Arguments indeterminately sequenced |

- "Unsequenced" means the two `x++` operations could be interleaved at the instruction level - both could read the old value of `x` before either writes, leading to contradictory state.
- "Indeterminately sequenced" means one fully completes before the other starts - safe but order is compiler-chosen.

### Q2: What specific sequencing rules did C++17 change for function arguments and operands

This example walks through each C++17 improvement in isolation so you can see exactly what changed and why:

```cpp
#include <iostream>
#include <string>
#include <map>

// Demonstrate C++17 sequencing improvements
int main() {
    // 1. Assignment: RHS sequenced before LHS
    int i = 0;
    // Pre-C++17: i = ++i; was UB (modification of i on both sides)
    // C++17: RHS (++i) sequenced before LHS assignment -> well-defined
    i = ++i;  // C++17: i becomes 1, then assigned to i -> i = 1

    // 2. Chained << / >> operators: left-to-right
    // Pre-C++17: std::cout << i++ << i++; was UB
    // C++17: left operand fully evaluated before right
    i = 0;
    std::cout << i++ << " " << i++ << "\n";  // C++17: prints "0 1"

    // 3. Compound assignment: RHS before LHS
    i = 2;
    i += ++i;  // C++17: ++i first (i=3), then i += 3 -> i = 6
    std::cout << "i = " << i << "\n";

    // 4. Subscript: index sequenced before subscript operation
    int arr[] = {10, 20, 30};
    i = 0;
    // Pre-C++17: arr[i] = i++; was UB
    // C++17: still problematic - the subscript i and i++ are
    // indeterminately sequenced (the standard sequences value computation
    // of the RHS before the assignment, but not before the index)

    // 5. Function calls: arguments indeterminately sequenced
    // f(unique_ptr_a.release(), unique_ptr_b.release());
    // C++17: no interleaving, one completes before the other
    // But order is unspecified

    // 6. Member access before arguments
    std::string s = "hello";
    // s.replace(0, 1, s);  // C++17: s evaluated before arguments
}
```

**Summary of C++17 changes:**

1. **Assignment (`a = b`):** `b` is sequenced before `a`.
2. **Compound assignment (`a @= b`):** `b` is sequenced before `a`.
3. **Shift operators in chaining (`a << b << c`):** Left operand before right.
4. **Function arguments:** Indeterminately sequenced (not interleaved), but order unspecified.
5. **Postfix expressions:** In `a.b(c)`, `a` is evaluated before `c`.

### Q3: Refactor code that contains unsequenced modifications to be safe

The fix is always the same: one modification per variable per statement. Here's the unsafe pattern alongside the safe rewrite for each common case:

```cpp
#include <iostream>
#include <vector>

// UNSAFE patterns and their SAFE equivalents

int main() {
    int i, x;

    // ---- Pattern 1: Multiple modifications ----
    // UNSAFE (UB in all standards):
    // i = 0; i = i++ + ++i;

    // SAFE: Break into separate statements
    i = 0;
    int temp1 = i;  // 0
    i++;             // i = 1
    ++i;             // i = 2
    i = temp1 + i;   // i = 0 + 2 = 2
    std::cout << "Pattern 1: " << i << "\n";

    // ---- Pattern 2: Function arguments ----
    // RISKY: f(x++, x++)  - UB pre-C++17, unspecified C++17
    x = 0;
    int a = x++;   // a = 0, x = 1
    int b = x++;   // b = 1, x = 2
    std::cout << "Pattern 2: a=" << a << " b=" << b << "\n";

    // ---- Pattern 3: Array subscript with modification ----
    // UNSAFE pre-C++17: arr[i] = i++;
    int arr[] = {10, 20, 30, 40};
    i = 1;
    int idx = i;   // Save index first
    i++;            // Then modify
    arr[idx] = 99;  // Then use saved index
    std::cout << "Pattern 3: arr[1]=" << arr[1] << "\n";  // 99

    // ---- Pattern 4: Chained output ----
    // UNSAFE pre-C++17: cout << i++ << i++;
    // SAFE: use separate statements
    i = 0;
    std::cout << "Pattern 4: " << i++;
    std::cout << " " << i++ << "\n";

    // ---- Pattern 5: Conditional with side effects ----
    // OK in all standards (short-circuit creates sequence point):
    i = 0;
    if (i++ == 0 && i++ == 1) {
        std::cout << "Pattern 5: both increments sequenced (short-circuit)\n";
    }

    // ---- Pattern 6: Comma operator (always sequenced) ----
    i = 0;
    x = (i++, i++, i++);  // Each comma is a sequence point
    // i=1, then i=2, then i=3; x = 2 (value of last i++)
    std::cout << "Pattern 6: i=" << i << " x=" << x << "\n";
}
```

When in doubt, split it into separate statements. The optimizer will merge them if it's safe to do so - your code loses nothing and gains clarity.

**Refactoring rules:**

| Unsafe Pattern | Safe Alternative |
| --- | --- |
| `f(x++, x++)` | `a=x++; b=x++; f(a,b);` |
| `arr[i] = i++` | `idx=i; i++; arr[idx]=val;` |
| `i = i++ + ++i` | Break into separate statements |
| `cout << i++ << i++` | Separate `cout` statements |
| **General rule** | **One modification per statement per variable** |

---

## Notes

- **Sequence point** is the C++03 term; C++11 replaced it with **sequenced-before/after** relationships for more precision.
- The comma **operator** introduces a sequence point, but the comma in `f(a, b)` is a **separator**, not the comma operator - it does NOT sequence.
- `&&`, `||`, and `?:` always sequence their operands (short-circuit evaluation) - these are safe in all standards.
- Use `-Wsequence-point` (GCC) or `-Wunsequenced` (Clang) to catch common issues.
- **Best practice:** Never modify a variable more than once in a single expression. If in doubt, split into separate statements - the optimizer will combine them if possible.
