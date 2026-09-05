# Use static analysis tools: clang-tidy, cppcheck, and sanitizers

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/extra/clang-tidy/>  

---

## Topic Overview

Static analysis and sanitizers catch bugs that compilers miss. Think of them as a second set of eyes - the compiler checks if your code is well-formed, but these tools look for patterns that compile just fine yet are almost certainly wrong (uninitialized reads, use-after-free, signed overflow, data races). They should be part of every CI pipeline.

### Tool Comparison

| Tool | Type | Catches | When |
| --- | --- | --- | --- |
| **clang-tidy** | Static analyzer | Style, bugs, modernization | Compile time |
| **cppcheck** | Static analyzer | Leaks, null deref, UB patterns | Compile time |
| **AddressSanitizer** (ASan) | Runtime sanitizer | Use-after-free, buffer overflow, leaks | Runtime |
| **UBSan** | Runtime sanitizer | Signed overflow, null deref, alignment | Runtime |
| **ThreadSanitizer** (TSan) | Runtime sanitizer | Data races, deadlocks | Runtime |
| **MemorySanitizer** (MSan) | Runtime sanitizer | Uninitialized reads | Runtime |

---

## Self-Assessment

### Q1: Run clang-tidy and fix modernize/readability warnings

To appreciate what clang-tidy does for you, look at the kind of legacy code it flags. The issues here are real and common: no `explicit` on single-argument constructors, raw `new` without RAII, `NULL` instead of `nullptr`, `typedef` instead of `using`.

**Before clang-tidy (legacy code):**

```cpp
#include <vector>
#include <iostream>
using namespace std;

class Widget {
    int* data;
public:
    Widget(int n) { data = new int[n]; }    // missing explicit
    ~Widget() { delete[] data; }            // missing Rule of 5
    void process() {
        for (int i = 0; i < 10; i++) {      // magic number
            int* p = NULL;                   // use nullptr
            auto v = vector<int>();          // use initializer
            typedef int MyInt;               // use using
        }
    }
};
```

These are the clang-tidy commands to analyze and auto-fix the file:

```bash
# Run with modernize checks
clang-tidy source.cpp -checks='modernize-*,readability-*' --

# Common modernize checks:
# modernize-use-nullptr          : NULL -> nullptr
# modernize-use-override         : add override keyword
# modernize-use-auto             : explicit types -> auto
# modernize-use-using            : typedef -> using
# modernize-avoid-c-arrays       : int[] -> std::array
# modernize-loop-convert         : index loop -> range for
# modernize-make-unique          : new -> make_unique
# modernize-pass-by-value        : const T& + copy -> T by value + move

# Fix automatically:
clang-tidy source.cpp -checks='modernize-*' -fix --
```

After running with `-fix`, the same class looks like this - notice that `unique_ptr` eliminates the Rule-of-Five problem entirely:

**After fix (modern C++):**

```cpp
#include <array>
#include <memory>
#include <vector>
#include <iostream>

class Widget {
    std::unique_ptr<int[]> data;
public:
    explicit Widget(int n) : data(std::make_unique<int[]>(n)) {}
    // Rule of 5: automatically handled by unique_ptr

    void process() {
        constexpr int iterations = 10;
        for (int i = 0; i < iterations; ++i) {
            int* p = nullptr;
            auto v = std::vector<int>();
            using MyInt = int;
        }
    }
};
```

Each mechanical problem is gone, and the resulting code is both safer and more readable.

### Q2: Enable AddressSanitizer and detect use-after-free

AddressSanitizer (ASan) instruments your binary at compile time to track every heap allocation and deallocation. When you access memory that has already been freed - or go past the end of an array - it prints an exact stack trace pointing to the bad access and to where the memory was originally allocated and freed.

```cpp
#include <iostream>
#include <vector>

// This code has a use-after-free bug!
void buggy_code() {
    int* p = new int(42);
    delete p;
    // BUG: accessing freed memory
    std::cout << *p << '\n';  // ASan catches this!
}

void dangling_reference() {
    std::vector<int> v = {1, 2, 3};
    int& ref = v[0];
    v.push_back(4);  // may reallocate!
    // BUG: ref may now be dangling
    std::cout << ref << '\n';  // ASan may catch this
}

void buffer_overflow() {
    int arr[5] = {1, 2, 3, 4, 5};
    // BUG: out-of-bounds access
    std::cout << arr[10] << '\n';  // ASan catches this!
}

int main() {
    std::cout << "Testing sanitizers...\n";
    // Uncomment ONE to test:
    // buggy_code();        // heap-use-after-free
    // dangling_reference(); // sometimes caught
    // buffer_overflow();    // stack-buffer-overflow
    std::cout << "No bugs triggered.\n";
}
// Expected output (without bugs):
// Testing sanitizers...
// No bugs triggered.
```

Build with the `-fsanitize=address` flag - the `-g` and `-O1` combination keeps frames readable without losing too much information:

```bash
# Compile with ASan
g++ -std=c++20 -fsanitize=address -fno-omit-frame-pointer -g -O1 test.cpp -o test

# ASan output for use-after-free:
# ==12345==ERROR: AddressSanitizer: heap-use-after-free
# READ of size 4 at 0x602000000010
#     #0 buggy_code() test.cpp:7
#     #1 main test.cpp:20
# allocated by thread T0 here:
#     #0 operator new test.cpp:5
# freed by thread T0 here:
#     #0 operator delete test.cpp:6
```

That diagnostic tells you exactly which line did the bad read, where the memory came from, and where it was freed. Finding this manually in a large codebase would take hours; ASan catches it in seconds.

### Q3: Use UndefinedBehaviorSanitizer to catch signed integer overflow

Signed integer overflow is undefined behavior in C++. The compiler is allowed to assume it never happens, and it uses that assumption to perform optimizations. That means overflow can silently produce wrong results, or the compiler can eliminate the check that was supposed to prevent it. UBSan instruments the binary to trap at the exact instruction where overflow would occur.

```cpp
#include <iostream>
#include <climits>

// Signed overflow is UNDEFINED BEHAVIOR in C++!
int dangerous_add(int a, int b) {
    return a + b;  // UBSan catches overflow here
}

int dangerous_shift(int x) {
    return x << 31;  // UB if x is negative or result overflows
}

int dangerous_divide(int a, int b) {
    return a / b;  // UB if b == 0 or a == INT_MIN && b == -1
}

int main() {
    // These would trigger UBSan:
    std::cout << "INT_MAX = " << INT_MAX << '\n';

    // Overflow detection:
    // int result = dangerous_add(INT_MAX, 1);  // UBSan: signed overflow
    // UBSan output: "runtime error: signed integer overflow:
    //                2147483647 + 1 cannot be represented in type 'int'"

    // Safe version:
    long long safe = static_cast<long long>(INT_MAX) + 1;
    std::cout << "Safe: " << safe << '\n';

    // Shift detection:
    // int shifted = dangerous_shift(-1);  // UBSan: left shift of negative value

    // Division detection:
    // int div = dangerous_divide(INT_MIN, -1);  // UBSan: division overflow
}
// Expected output:
// INT_MAX = 2147483647
// Safe: 2147483648
```

Enabling UBSan is just one flag change, and it catches a whole class of silent bugs:

```bash
# Compile with UBSan
g++ -std=c++20 -fsanitize=undefined -g -O1 test.cpp -o test

# Combine multiple sanitizers:
g++ -std=c++20 -fsanitize=address,undefined -fno-omit-frame-pointer -g test.cpp -o test

# UBSan checks include:
# -fsanitize=signed-integer-overflow
# -fsanitize=null
# -fsanitize=alignment
# -fsanitize=bool
# -fsanitize=enum
# -fsanitize=float-divide-by-zero
# -fsanitize=shift
```

You can combine ASan and UBSan in one build since they use compatible instrumentation. Note that TSan and ASan conflict with each other, so those need separate test builds.

---

## Notes

- Run sanitizers in CI with every PR - they find bugs unit tests miss.
- ASan has ~2x slowdown; UBSan has <10% slowdown.
- ASan and TSan cannot be used together (conflicting instrumentation).
- cppcheck can be run without building: `cppcheck --enable=all src/`.
- `.clang-tidy` file in project root configures checks per-project.
