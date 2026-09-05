# Understand how to mix modules with header-based code using header units

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/modules>  

---

## Topic Overview

**Header units** are a transitional mechanism between `#include` and modules. They compile a header into a module-like format that can be imported with `import <header>;` syntax.

If you're looking at a large existing codebase and wondering how to get some of the benefits of modules without rewriting everything at once, header units are your answer. They sit in the middle of the spectrum: faster than raw `#include`, but not as clean as a proper named module.

### The Spectrum from Headers to Modules

It helps to think of the migration path as three distinct stages:

```cpp
+-----------------+    +-----------------+    +-----------------+
|  #include        | -> |  import <hdr>;  | -> |  import module; |
|  (textual)       |    |  (header unit)  |    |  (named module) |
|                  |    |                 |    |                 |
| - macros leak    |    | - macros export |    | - no macro leak |
| - reparsed/TU    |    | - compiled once |    | - compiled once |
| - no dep info    |    | - module-like   |    | - full isolation|
+-----------------+    +-----------------+    +-----------------+
    Legacy              Transitional           Modern
```

Each stage is a real improvement. You don't have to jump straight from legacy to modern.

### Comparison Table

| Feature | `#include <vector>` | `import <vector>;` | `import std;` |
| --- | --- | --- | --- |
| Parse cost | Every TU | Once | Once |
| Macros visible | Yes | Yes (!) | No |
| Available since | Always | C++20 | C++23 |
| Compiler support | All | MSVC, GCC, Clang | MSVC 17.5+ |
| Migration effort | None | Minimal | Moderate |

One thing in the table that surprises people: macros from header units *are* still visible to the importer, unlike named modules. Header units are not full modules - they're a compiled-once version of a textual header, and they preserve macro behavior.

---

## Self-Assessment

### Q1: Import a legacy header as a header unit using `import <vector>;`

The syntax change is minimal - you just replace `#include` with `import`. Here's what it looks like in practice:

```cpp
// Using import <header> syntax for header units
import <iostream>;
import <vector>;
import <algorithm>;
import <string>;

int main() {
    // All standard library features available via header unit imports
    std::vector<std::string> names = {"Charlie", "Alice", "Bob"};
    std::sort(names.begin(), names.end());

    for (const auto& name : names)
        std::cout << name << ' ';
    std::cout << '\n';

    // Macros from header units ARE visible (unlike named modules)
    // <cassert> defines the assert macro
    // import <cassert>;  // assert() macro would be available
}
// Expected output:
// Alice Bob Charlie
```

The code looks almost identical to the `#include` version. The real benefit is on the build side - the compiler processes each header unit once and reuses the result across all translation units that import it.

How you invoke this depends on your compiler. Both MSVC and GCC need to be told which headers to treat as header units:

**Compilation (MSVC):**

```bash
# MSVC needs /headerUnit for each header unit
cl /std:c++20 /EHsc /headerUnit:angle <vector> <algorithm> main.cpp

# Or use /translateInclude to auto-convert #include to import
cl /std:c++20 /EHsc /translateInclude main.cpp
```

**Compilation (GCC):**

```bash
# GCC uses -fmodule-header for header units
g++ -std=c++20 -fmodules-ts -fmodule-header=<vector> main.cpp
```

The key point is that `import <vector>;` compiles `<vector>` into a header unit once, then reuses it across all TUs - faster than re-parsing every time.

### Q2: Explain the difference between `import <header>` (header unit) and `#include <header>`

At first glance they look similar, but the processing model is different in some important ways:

| Behavior | `#include <header>` | `import <header>;` |
| --- | --- | --- |
| **Processing** | Textual copy-paste into TU | Read pre-compiled header unit |
| **Macros** | Injected into TU | Also injected (header units export macros!) |
| **Parse time** | O(header_size) per TU | O(1) per TU (read BMI) |
| **ODR safety** | Fragile (different define contexts) | Safer (single compilation) |
| **Include order** | Matters (macros affect later includes) | Independent of order |
| **Preprocessor state** | Changed by macros from header | Changed by macros from header unit |

The subtlety around macros is especially worth understanding. With `#include`, macros you define before the include can affect how the header is processed. With `import`, that's not the case - the header unit was already compiled in its own fixed environment:

```cpp
// With #include:
#define _LIBCPP_DEBUG 1
#include <vector>           // picks up debug mode via macro

// With import:
#define _LIBCPP_DEBUG 1
import <vector>;            // the header unit was compiled WITHOUT this macro!
                            // The define has no effect on the import
```

Header units are compiled in a **fixed macro environment** - macros defined by the importer before `import` do NOT affect the header unit's compilation. This is fundamentally different from `#include`. If your code depends on setting macros before including a header to change its behavior, switching to a header unit for that header will break things.

### Q3: Show how to gradually migrate a codebase from headers to modules

One of the things that makes modules intimidating is the idea that you have to rewrite everything at once. You don't. Here's a four-phase migration path that lets you incrementally move forward while keeping the codebase working at every step.

#### Phase 1: No changes (baseline)

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// math_utils.h
#pragma once
#include <cmath>

inline double distance(double x, double y) {
    return std::sqrt(x * x + y * y);
}

// main.cpp
#include "math_utils.h"
#include <iostream>

int main() {
    std::cout << distance(3.0, 4.0) << '\n';  // 5
}
```

#### Phase 2: Switch standard includes to header units

Your own project headers stay as `#include` - you're just getting the build speedup on the standard library headers:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// math_utils.h (unchanged)
#pragma once
#include <cmath>

inline double distance(double x, double y) {
    return std::sqrt(x * x + y * y);
}

// main.cpp - use header unit for standard header
#include "math_utils.h"  // still #include for project header
import <iostream>;       // switch to header unit for std headers

int main() {
    std::cout << distance(3.0, 4.0) << '\n';
}
```

#### Phase 3: Convert project headers to modules

Now you tackle your own code. Move `math_utils.h` to a `.cppm` module file. Use the global module fragment for any C headers it needs:

```cpp
// math_utils.cppm - now a module
module;
#include <cmath>  // C header in global module fragment
#include <iostream>

export module math_utils;

export double distance(double x, double y) {
    return std::sqrt(x * x + y * y);
}

// main.cpp - import the module
import math_utils;
import <iostream>;

int main() {
    std::cout << distance(3.0, 4.0) << '\n';
}
```

#### Phase 4: Use `import std;` (C++23)

Once C++23 support is available, you can drop all the standard library includes entirely:

```cpp
// math_utils.cppm
export module math_utils;
import std;  // replaces <cmath> include

export double distance(double x, double y) {
    return std::sqrt(x * x + y * y);
}

// main.cpp
import math_utils;
import std;

int main() {
    std::cout << distance(3.0, 4.0) << '\n';
}
```

Here's a summary of the risk and benefit at each phase:

| Phase | Change | Risk | Benefit |
| --- | --- | --- | --- |
| 1 | Baseline | None | None |
| 2 | `import <header>` for std | Low | Faster std header parsing |
| 3 | Convert own headers to modules | Medium | Macro isolation, better encapsulation |
| 4 | `import std;` | Low | Simplest possible includes |

---

## Notes

- Header units and named modules can coexist in the same project - use header units for libraries not yet modularized.
- Not all headers can be header units (headers with complex macro interactions may fail).
- MSVC `/translateInclude` flag can auto-convert `#include` to header unit imports for standard headers.
- Header units are **not** the same as precompiled headers (PCH) - they have module semantics and proper isolation.
