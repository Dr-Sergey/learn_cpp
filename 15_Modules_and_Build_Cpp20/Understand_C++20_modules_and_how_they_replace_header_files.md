# Understand C++20 modules and how they replace header files

**Category:** Modules and Build Cpp20  
**Standard:** C++20, C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/modules>  

---

## Topic Overview

C++20 **modules** are a new compilation model that replaces textual inclusion (`#include`) with a structured import system. Instead of copy-pasting a header's text into every translation unit that needs it, a module is compiled once into a **Binary Module Interface (BMI)** and then imported by name. The compiler reads the BMI directly - no re-parsing, no macro leakage, no include-order headaches.

### Headers vs Modules

The table below captures the practical differences between the two models. If you have ever spent time tracking down a macro that clobbered a variable name, or puzzled over why changing include order broke a build, these properties will feel meaningful.

| Aspect | `#include` (headers) | `import` (modules) |
| --- | --- | --- |
| Mechanism | Textual copy-paste by preprocessor | Compiler reads pre-compiled BMI |
| Macro leakage | Yes - macros leak to all includers | No - macros are NOT exported |
| Include guards | Needed (`#pragma once` / guards) | Not needed - import is idempotent |
| Parse cost | Re-parsed in every TU | Parsed once, BMI reused |
| Symbol visibility | Everything visible unless hidden | Only `export`ed names visible |
| Order dependence | Yes - include order matters | No - imports are order-independent |
| ODR violations | Easy to cause with mismatched includes | Prevented by design |

### Module Structure

A module is typically split into an **interface unit** (the `.cppm` file that declares what is exported) and one or more **implementation units** (`.cpp` files that define the function bodies). Consumers only ever see the interface.

```cpp
┌─────────────────────────────────────┐
│ Module Interface Unit (.cppm/.ixx)  │
│                                     │
│  export module mylib;               │  <- module declaration
│                                     │
│  export int add(int, int);          │  <- exported (visible to importers)
│  int helper(int);                   │  <- module-local (NOT visible)
│                                     │
│  export namespace math { ... }      │  <- export entire namespace
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Module Implementation Unit (.cpp)   │
│                                     │
│  module mylib;                      │  <- belongs to mylib
│                                     │
│  int add(int a, int b) { return a+b; }
│  int helper(int x) { return x*2; } │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Consumer (.cpp)                     │
│                                     │
│  import mylib;                      │  <- import the module
│  int x = add(1, 2);                │  <- OK
│  int y = helper(3);                │  <- ERROR: not exported
└─────────────────────────────────────┘
```

---

## Self-Assessment

### Q1: Write a simple module with `export module my.module;` and import it in another TU

The module interface declares what is visible. The implementation unit (with just `module my.module;` and no `export`) provides the definitions. Notice that the consumer only sees the exported names - `internal_helper` is completely invisible.

**Module interface (my_module.cppm):**

```cpp
export module my.module;

// Export a function
export int greet_length(const char* name);

// Export a class
export class Calculator {
    int value_ = 0;
public:
    void add(int n) { value_ += n; }
    void subtract(int n) { value_ -= n; }
    int result() const { return value_; }
};

// Non-exported helper - visible only within this module
int internal_helper() { return 42; }
```

**Module implementation (my_module_impl.cpp):**

```cpp
module my.module;

// <cstring> can be included in implementation unit
#include <cstring>

int greet_length(const char* name) {
    return static_cast<int>(std::strlen(name));
}
```

**Consumer (main.cpp):**

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
import my.module;     // import the module
#include <iostream>   // still use #include for non-module headers

int main() {
    // Use exported function
    std::cout << "Name length: " << greet_length("Alice") << '\n';

    // Use exported class
    Calculator calc;
    calc.add(10);
    calc.add(5);
    calc.subtract(3);
    std::cout << "Result: " << calc.result() << '\n';

    // internal_helper();  // ERROR: not exported, invisible
}
// Expected output:
// Name length: 5
// Result: 12
```

**How this works:**

- `export module my.module;` declares a named module and makes `export`ed declarations visible to importers.
- `import my.module;` brings in only the exported symbols - no macros, no include-order issues.
- Non-exported names (`internal_helper`) have **module linkage** and are invisible outside.

### Q2: Explain how modules eliminate macro leakage and header guard issues

Macro leakage is one of the most frustrating header-based bugs because the collision can come from a completely unrelated header included somewhere far up the include chain. Modules make this impossible.

**Macro leakage problem with headers:**

```cpp
// problem.h
#define MAX 100
#define ERROR -1

inline int compute() { return MAX; }
```

```cpp
// user1.cpp
#include "problem.h"
// MAX and ERROR are now defined here - can clash with other code!
int my_error = ERROR;  // expands to -1 (unintended!)
```

**Modules solve this:**

```cpp
// safe_module.cppm
export module safe;

#define MAX 100       // This macro exists ONLY within this module file
#define ERROR -1      // NOT exported, NOT visible to importers

export inline int compute() { return MAX; }  // uses MAX internally
export inline int error_code() { return ERROR; }
```

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
// user.cpp
import safe;
#include <iostream>

int main() {
    std::cout << compute() << '\n';     // OK: 100
    std::cout << error_code() << '\n';  // OK: -1

    // MAX is NOT defined here
    // ERROR is NOT defined here
    // No macro pollution!

    #ifdef MAX
        std::cout << "MAX leaked!\n";   // This will NOT print
    #else
        std::cout << "No macro leak\n"; // This WILL print
    #endif
}
// Expected output:
// 100
// -1
// No macro leak
```

Header guard elimination is an equally pleasant side effect. Since `import` is idempotent by definition, you never need `#pragma once` or `#ifndef` guards.

**Header guard elimination:**

```cpp
// With headers, forgetting guards causes errors:
// header.h - no guard!
int x = 5;  // multiple definition if included twice

// With modules:
import my.module;
import my.module;  // OK! No effect - import is always idempotent
```

### Q3: Show how `import std;` (C++23) replaces including every standard library header

One of the best quality-of-life improvements in C++23. Instead of a long list of `#include` directives at the top of every file, a single `import std;` gives you the entire standard library.

**Before (C++17/20):**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <ranges>
#include <memory>
#include <unordered_map>
#include <thread>
// ... 20+ more headers for a typical file
```

**After (C++23):**

```cpp
import std;  // imports the ENTIRE C++ standard library

int main() {
    // Everything is available:
    std::vector<int> v = {3, 1, 4, 1, 5};
    std::ranges::sort(v);

    std::string msg = "Hello";
    std::cout << msg << ": ";
    for (int x : v) std::cout << x << ' ';
    std::cout << '\n';

    auto ptr = std::make_unique<int>(42);
    std::cout << *ptr << '\n';

    std::unordered_map<std::string, int> map;
    map["key"] = 100;
    std::cout << map["key"] << '\n';
}
// Expected output:
// Hello: 1 1 3 4 5
// 42
// 100
```

There is also a `std.compat` variant if you need the C standard library names in the global namespace.

**`import std;` vs `import std.compat;`:**

| Module | Content | C names |
| --- | --- | --- |
| `import std;` | All C++ standard library | `std::` prefixed only (`std::printf`) |
| `import std.compat;` | All C++ + C compatibility | Also global `::printf`, `::malloc`, etc. |

**Compiler support:**

- MSVC 17.5+: `import std;` fully supported
- GCC 15+: experimental
- Clang 18+: experimental
- CMake 3.30+: supports `import std;` with appropriate compiler

---

## Notes

- Module interface files use `.cppm` (GCC/Clang) or `.ixx` (MSVC) extension.
- `export` works on declarations, namespaces, and blocks: `export { int a; double b; }`.
- Modules don't export macros - use the **global module fragment** for legacy macro dependencies.
- `import` declarations must appear at the top of the file, after the module declaration but before any other code.
- Modules enable **better error messages** because the compiler knows exactly what was exported.
