# Understand and prevent undefined behavior (UB)

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/ub>  

---

## Topic Overview

**Undefined Behavior (UB)** means the C++ standard imposes no requirements on what happens. The program may crash, produce wrong results, appear to work correctly, or do something completely unrelated to the bug. The "or launch missiles" joke from the early standards discussions exists because the standard literally makes no promise - any outcome is permitted.

### Why Compilers Care About UB

Here is the part that trips people up most. Compilers **assume UB never happens** and use that assumption to optimize. This means UB does not just silently produce the wrong result in one place - it can make *other* code behave unexpectedly, because the compiler is allowed to restructure or delete code on the premise that you wrote a correct program. A UB in one function can corrupt logic in a completely different function after optimization.

### 10 Common Causes of UB

| # | UB Source | Example |
| --- | --- | --- |
| 1 | Signed integer overflow | `INT_MAX + 1` |
| 2 | Null pointer dereference | `int* p = nullptr; *p;` |
| 3 | Out-of-bounds access | `arr[10]` when size is 5 |
| 4 | Data race | Two threads write same variable without sync |
| 5 | Use after free | `delete p; *p;` |
| 6 | Uninitialized variable read | `int x; return x;` |
| 7 | Strict aliasing violation | `*(float*)&int_var` |
| 8 | Double free | `delete p; delete p;` |
| 9 | Shift by negative/excess | `1 << 32` (32-bit int) |
| 10 | Missing return in non-void | `int f() { if(x) return 1; }` |

---

## Self-Assessment

### Q1: List 10 common causes of UB and show examples

Each example below is commented out because running it would actually trigger UB - but the safe alternatives at the bottom show the correct patterns.

```cpp
#include <climits>
#include <iostream>

void ub_examples() {
    // 1. Signed overflow (UB)
    // int a = INT_MAX; a += 1;  // UB! (unsigned wraps, signed is UB)

    // 2. Null dereference
    // int* p = nullptr; int x = *p;  // UB!

    // 3. Out-of-bounds
    int arr[3] = {1, 2, 3};
    // int x = arr[5];  // UB! (no bounds check in C++)

    // 4. Data race (needs threads to demonstrate)
    // Two threads: shared++ without mutex or atomic

    // 5. Use after free
    // int* p = new int(42); delete p; int x = *p;  // UB!

    // 6. Uninitialized read
    // int x; std::cout << x;  // UB! (indeterminate value)

    // 7. Strict aliasing violation
    // int i = 42;
    // float f = *(float*)&i;  // UB! (except through char*/byte*)

    // 8. Double free
    // int* p = new int(5); delete p; delete p;  // UB!

    // 9. Shift overflow
    // int x = 1 << 32;  // UB if int is 32 bits

    // 10. Missing return
    // int foo() { /* no return statement */ }  // UB!
}

// SAFE alternatives:
#include <cstdint>
#include <vector>

int main() {
    // Use unsigned for wrapping arithmetic
    uint32_t a = UINT32_MAX;
    a += 1;  // wraps to 0, well-defined
    std::cout << "Unsigned wrap: " << a << '\n';

    // Use .at() for bounds checking
    std::vector<int> v = {1, 2, 3};
    try {
        int x = v.at(5);  // throws std::out_of_range, not UB
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }

    // Always initialize variables
    int x = 0;  // initialized!
    std::cout << "Initialized: " << x << '\n';
}
// Expected output:
// Unsigned wrap: 0
// Caught: vector::_M_range_check: __n (which is 5) >= this->size() (which is 3)
// Initialized: 0
```

The key safe alternatives: use `uint32_t` when you want wrapping arithmetic, use `.at()` when you want a bounds-checked access that throws instead of corrupting memory, and always initialize your variables.

### Q2: Show how a compiler uses UB assumptions to generate surprising code

This is the most important thing to understand about UB, and the reason it trips experienced programmers. The compiler does not just "do something wrong" - it actively uses the absence of UB as a reasoning tool, which can delete code you thought was protecting you.

```cpp
#include <iostream>

// The compiler ASSUMES signed overflow never happens.
// This lets it "optimize" in surprising ways.

bool check_overflow(int x) {
    // Programmer intends: detect if x+1 overflows
    return x + 1 > x;
}
// Compiler reasoning:
// - Signed overflow is UB
// - I can assume UB never happens
// - Therefore x + 1 > x is ALWAYS true
// - Optimized to: return true;
// Input INT_MAX -> returns true (wrong!)

// Another classic: null check elimination
void process(int* p) {
    int val = *p;         // dereferences p
    if (p == nullptr) {   // compiler sees: p was already dereferenced
        std::cout << "null!\n";  // so p can't be null (that would be UB)
        return;                   // this branch is DEAD CODE
    }
    std::cout << "value: " << val << '\n';
}
// The null check is REMOVED by the compiler because
// dereferencing null is UB, so the compiler assumes p != nullptr.

int main() {
    std::cout << "check_overflow(INT_MAX) = " << std::boolalpha
              << check_overflow(INT_MAX) << '\n';
    // With -O2, likely prints "true" (wrong!)
    // The compiler optimized away the overflow check

    int val = 42;
    process(&val);
}
// Expected output (with -O2):
// check_overflow(INT_MAX) = true
// value: 42
```

The null check elimination example is the one that surprises people most. You write `if (p == nullptr)` after already dereferencing `p`, and the compiler reasons: "if `p` were null, `*p` would be UB, and I'm allowed to assume that does not happen - therefore `p != nullptr`, therefore this whole `if` branch is dead code and I can remove it." Your safety check evaporates.

### Q3: Enable `-fsanitize=undefined` and catch UB

You should run with sanitizers enabled in your development builds and CI. They catch UB at the point it occurs, with a precise error message and a line number - something the optimizer-mangled crash dump from a release build cannot give you.

```cpp
Compilation and usage:

$ cat ub_example.cpp
#include <iostream>
#include <climits>

int main() {
    int x = INT_MAX;
    x += 1;  // signed overflow!
    std::cout << x << '\n';

    int arr[3] = {1, 2, 3};
    std::cout << arr[5] << '\n';  // out of bounds!
}

$ g++ -std=c++20 -fsanitize=undefined -g ub_example.cpp -o ub_example
$ ./ub_example

ub_example.cpp:6:7: runtime error: signed integer overflow:
  2147483647 + 1 cannot be represented in type 'int'
ub_example.cpp:9:18: runtime error: index 5 out of bounds
  for type 'int [3]'
```

Each sanitizer covers a different category of problem. Use them in combination for the most thorough coverage:

**Available sanitizers:**

| Sanitizer | Flag | Detects |
| --- | --- | --- |
| UBSan | `-fsanitize=undefined` | Signed overflow, null deref, shift, alignment |
| ASan | `-fsanitize=address` | Heap/stack buffer overflow, use-after-free |
| TSan | `-fsanitize=thread` | Data races |
| MSan | `-fsanitize=memory` | Uninitialized reads |

---

## Notes

- UB is **not** implementation-defined behavior (which is defined by each compiler).
- Use `-Wall -Wextra -Werror` to catch many UB sources at compile time.
- Sanitizers have 2-5x runtime overhead - use in CI/testing, not production.
- `std::bit_cast` (C++20) is the safe alternative to type-punning through casts.
- The compiler's UB assumptions enable optimizations like: dead code elimination, loop vectorization, constant propagation. Avoiding UB makes your code both correct AND fast.
