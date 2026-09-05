# Understand inline variables (C++17) and their use in headers

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#inline-variables>  

---

## Topic Overview

Before C++17, defining a variable in a header that is included by multiple translation units (TUs) caused **ODR (One Definition Rule) violations** - the linker would see duplicate definitions and report an error. `inline` variables (C++17) solve this by allowing a variable to be defined in multiple TUs with the guarantee that only **one instance** exists.

### The Problem (Pre-C++17)

Without `inline`, a non-const global in a header is a linker error waiting to happen, and even `const` has the subtle downside that each TU gets its own independent copy:

```cpp
// config.h -- included by multiple .cpp files
const int max_size = 1024;       // internal linkage (const) -- OK, but separate copy per TU
extern int counter;              // declaration only -- must define in exactly one .cpp
// int global = 42;              // ERROR: ODR violation if included in multiple TUs
```

### The Solution: inline Variables (C++17)

Mark the variable `inline` and the linker deduplicates - one shared instance, same address everywhere:

```cpp
// config.h
inline int counter = 0;               // one instance across all TUs
inline constexpr int max_size = 1024;  // constexpr is implicitly inline for static members
```

### How It Works

- `inline` tells the linker: "this may appear in multiple TUs - keep only one copy."
- All definitions must be identical (same initializer, same type).
- The variable has **external linkage** - all TUs share the same object.
- Address is guaranteed to be the same everywhere: `&counter` is consistent.

### Key Use Cases

| Use Case | Syntax |
| --- | --- |
| Global constants in headers | `inline constexpr int MAX = 100;` |
| Static class members defined in-header | `static inline int count = 0;` |
| Singleton-like globals | `inline MyConfig config{...};` |

### Static Member Variables in Classes

Before C++17 you needed a separate `.cpp` file just to give the static member storage. Now you can define it right where you declare it:

```cpp
// Before C++17:
// widget.h
struct Widget {
    static int count;   // declaration
};
// widget.cpp
int Widget::count = 0;  // definition -- must be in exactly one .cpp

// C++17:
struct Widget {
    static inline int count = 0;   // definition right in the header!
};
// No .cpp file needed for the definition
```

---

## Self-Assessment

### Q1: Explain why a non-inline `const int` in a header causes ODR violations when included in multiple TUs

**Answer:**

Actually, `const int` at namespace scope has **internal linkage** in C++ (unlike C), so it does NOT cause ODR violations - each TU gets its own copy. The issue is different for non-const:

```cpp
// globals.h
int value = 42;              // external linkage, non-inline
// If included in a.cpp and b.cpp: TWO definitions of 'value' -> linker error

const int cval = 42;         // internal linkage in C++ -> each TU gets a copy
// No linker error, but: &cval differs between TUs (separate objects)

inline int ival = 42;        // external linkage, inline -> ONE shared instance
// &ival is the same in all TUs
```

The problem with `const` having internal linkage:

1. Each TU has a **separate copy** - wastes memory for large objects.
2. `&cval` in TU1 is not equal to `&cval` in TU2 - problematic if addresses are compared.
3. Non-`const` variables at namespace scope have external linkage -> ODR violation if defined in a header.

`inline` variables solve both: external linkage + single instance + can be defined in headers.

### Q2: Define an inline constexpr variable in a header and verify there is only one definition

Both translation units print the same address because the linker kept exactly one copy:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// ---- constants.h ----
#pragma once

inline constexpr double pi = 3.14159265358979;
inline constexpr int max_buffer = 4096;

// ---- a.cpp ----
#include "constants.h"
#include <iostream>

void print_from_a() {
    std::cout << "a: pi = " << pi << " at " << &pi << "\n";
    std::cout << "a: max = " << max_buffer << " at " << &max_buffer << "\n";
}

// ---- b.cpp ----
#include "constants.h"
#include <iostream>

void print_from_b() {
    std::cout << "b: pi = " << pi << " at " << &pi << "\n";
    std::cout << "b: max = " << max_buffer << " at " << &max_buffer << "\n";
}

// ---- main.cpp ----
void print_from_a();
void print_from_b();

int main() {
    print_from_a();
    print_from_b();
    // Both print the SAME address for pi and max_buffer
    // -> only one instance exists despite being defined in a header
}
```

**How it works:**

- `inline constexpr` variables can be defined in a header included by multiple TUs.
- The linker deduplicates them -> single instance, single address.
- `constexpr` at namespace scope is **not** implicitly `inline` (unlike `constexpr` static members), so the `inline` keyword matters.

### Q3: Compare inline variable vs a static member variable for defining class-level constants

**Answer:**

```cpp
// Approach 1: static constexpr member (C++17 -- implicitly inline)
struct Config_A {
    static constexpr int max_retries = 3;   // implicitly inline since C++17
    static constexpr double timeout = 5.0;
};
// Can take address: &Config_A::max_retries -- same address in all TUs

// Approach 2: static inline member (mutable)
struct Config_B {
    static inline int request_count = 0;   // mutable shared state
};
// Config_B::request_count can be modified and is shared across all TUs

// Approach 3: namespace-scope inline constexpr (no class needed)
inline constexpr int MAX_RETRIES = 3;

// Approach 4: Pre-C++17 -- enum hack (compile-time constant, no storage)
struct Config_Old {
    enum { max_retries = 3 };   // always compile-time, no address
};
```

If the table feels like a lot, the practical rule is: `static constexpr` for class constants, `inline constexpr` for free-standing constants, `static inline` for mutable shared state:

**Comparison:**

| Feature | `static constexpr` member | `static inline` member | namespace `inline constexpr` |
| --- | --- | --- | --- |
| Mutable? | No | Yes | No |
| Needs class? | Yes | Yes | No |
| Has address | Yes (inline since C++17) | Yes | Yes |
| Defined in header | Yes (C++17+) | Yes (C++17+) | Yes (C++17+) |
| Pre-C++17 | Needs out-of-line definition for ODR-use | Not available | Not available |

**Best practice:** Use `static constexpr` for class-scoped constants, `inline constexpr` for namespace-scoped constants, and `static inline` for mutable shared state.

---

## Notes

- `constexpr` static data members are implicitly `inline` since C++17 - no need to write `static inline constexpr`.
- `inline` does NOT mean "substitute at call site" for variables - it means "allow multiple definitions, keep one."
- `inline constexpr` at namespace scope is the modern replacement for `#define CONSTANT 42` macros.
- `thread_local inline` is also valid - one instance per thread, but the same instance across TUs within a thread.
