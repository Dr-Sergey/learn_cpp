# Use Compiler Explorer with multiple compilers to check portability

**Category:** Tooling and Debugging  
**Standard:** C++20  
**Reference:** <https://godbolt.org>  

---

## Topic Overview

One of the most common portability mistakes is writing code on one compiler and assuming it works everywhere. GCC, Clang, and MSVC each have their own extension behavior, warning sets, and C++ standard support timelines. Compiler Explorer (godbolt.org) lets you test code against all three simultaneously - before you write a single line of CI configuration.

The basic workflow is to open multiple compiler panes from the same shared source and compare the output side by side:

```cpp
Godbolt layout for portability testing:
┌────────────┬────────────┬────────────┬────────────┐
│  Source    │ GCC 14     │ Clang 18   │ MSVC 19.38 │
│  (shared)  │ -std=c++20 │ -std=c++20 │ /std:c++20 │
│            │ OK         │ OK         │ error      │
└────────────┴────────────┴────────────┴────────────┘
```

When one pane shows an error and the others do not, you have a portability bug to fix before it reaches a colleague on a different platform.

---

## Self-Assessment

### Q1: Find compiler-specific warnings by comparing GCC and Clang

GCC and Clang agree on most things but diverge on several warning categories. Running the same code through both often reveals issues that you would only discover in production when someone builds with the other compiler.

```cpp
#include <iostream>
#include <cstring>

// Code that triggers different warnings on different compilers:

void example() {
    // 1. Narrowing in list initialization:
    int x = 1000;
    char c = x;  // GCC: -Wnarrowing (sometimes), Clang: -Wconstant-conversion

    // 2. Implicit fallthrough:
    switch (x) {
    case 1:
        std::cout << "one";
        // GCC: -Wimplicit-fallthrough (C++17 with [[fallthrough]])
        // Clang: -Wimplicit-fallthrough
    case 2:
        std::cout << "two";
        break;
    }

    // 3. Comparison of different signedness:
    unsigned u = 5;
    int i = -1;
    if (i < u) {}  // Both warn with -Wsign-compare
                   // But Clang also warns with -Wsign-conversion

    // 4. GCC-only: -Wstrict-aliasing
    float f = 3.14f;
    int* pi = reinterpret_cast<int*>(&f);  // GCC warns; UB in both

    // 5. Clang-only: -Wself-assign
    int val = 42;
    val = val;  // Clang: -Wself-assign, GCC: silent

    (void)c; (void)pi; (void)val;
}

int main() {
    example();
    std::cout << "Compiled successfully\n";
}
```

The practical lesson here is that enabling both `-Wall -Wextra` on GCC and the equivalent on Clang gives you a significantly larger net than either alone. Running both in Godbolt takes about thirty seconds and can surface issues that would otherwise slip through.

### Q2: Use the conformance view to check feature support

Godbolt's "Add new..." - "Conformance" view tests your code against every compiler version at once. This is the fastest way to answer "what is the oldest compiler that supports this feature?"

```cpp
// Feature: C++23 std::expected
#include <expected>
#include <string>

std::expected<int, std::string> parse(const char* s) {
    if (s[0] >= '0' && s[0] <= '9')
        return s[0] - '0';
    return std::unexpected("not a digit");
}

int main() {
    auto r = parse("5");
    return r.value_or(-1);
}
```

Paste this into the conformance view and you immediately see which compiler versions support it:

```text
GCC 12:        No (no <expected> yet)
GCC 13:        Yes
Clang 16:      No (partial)
Clang 17:      Yes
MSVC 19.36:    Yes
MSVC 19.34:    No
```

You can do the same for any C++20 feature with uneven support:

```cpp
// C++20 features with variable support:
auto f(auto x) { return x; }         // abbreviated template (GCC10+, Clang12+, MSVC19.28+)
auto [a, b] = std::pair{1, 2};        // structured bindings (C++17, widely supported)
consteval int sq(int n) { return n*n; }  // consteval (GCC10+, Clang11+, MSVC19.29+)
```

### Q3: Share a permlink for a compiler bug report

When you find a compiler bug, the most useful thing you can do is provide a minimal reproduction in Godbolt along with a permlink. Permlinks are stable forever and let the compiler team reproduce your issue with a single click.

First, minimize the code as much as possible. Then share it:

```cpp
// Potential GCC bug: internal compiler error on valid code
// Minimize the reproduction first!

template<typename T>
concept Addable = requires(T a, T b) { a + b; };

template<Addable T>
auto add(T a, T b) { return a + b; }

// If this ICEs on GCC but compiles on Clang:
int main() {
    return add(1, 2);
}
```

**Steps to report:**

1. Paste code in Godbolt with the failing compiler version.
2. Click **Share** -> **Short Link** (e.g., `https://godbolt.org/z/abc123`).
3. File bug at gcc.gnu.org/bugzilla or github.com/llvm/llvm-project/issues.
4. Include:
   - Permlink to Godbolt
   - Compiler version (e.g., "GCC 14.1 x86-64")
   - Expected behavior ("should compile") vs actual ("ICE")
   - Note which compilers succeed ("Clang 18 and MSVC 19.38 accept this")

A good bug report template looks like this:

```cpp
Bug report template:
  Title: [C++20] ICE on valid concept-constrained function template
  Compiler: GCC 14.1 (x86-64)
  Flags: -std=c++20 -O2
  Godbolt: https://godbolt.org/z/abc123
  Expected: Compiles successfully
  Actual: internal compiler error: in ...
  Note: Clang 18 and MSVC 19.38 accept this code.
```

---

## Notes

- Add multiple compiler panes for side-by-side comparison.
- Use "Conformance" view for quick feature-support checks across all compilers.
- Permlinks are permanent and stable - safe for bug reports and documentation.
- Test with both GCC and Clang since they have different C++20/23 support timelines.
- MSVC uses `/std:c++20` and `/std:c++latest` (not `-std=c++20`).
