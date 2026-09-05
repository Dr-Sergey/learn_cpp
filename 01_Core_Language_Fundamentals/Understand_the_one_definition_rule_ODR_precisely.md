# Understand the one definition rule (ODR) precisely

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/definition>  

---

## Topic Overview

The **One Definition Rule (ODR)** is one of the most fundamental rules in C++. It governs how many definitions an entity can have across the entire program. Violating it is surprisingly easy - and the consequences are often silent.

### The Three Parts of ODR

**Part 1:** Within a single translation unit (TU), every entity can have at most **one definition**.

**Part 2:** In the entire program, non-inline functions and non-inline variables with external linkage must have exactly **one definition** across all TUs.

**Part 3:** Some entities CAN have multiple definitions across TUs (one per TU), but all definitions must be **token-for-token identical**:

| Entity | Multiple definitions allowed? | Requirement |
| --- | --- | --- |
| Non-inline functions | No - exactly one definition | |
| Non-inline variables | No - exactly one definition | |
| `inline` functions | Yes - one per TU | All definitions must be identical |
| `inline` variables (C++17) | Yes - one per TU | All definitions must be identical |
| Class definitions | Yes - one per TU | All definitions must be identical |
| Templates | Yes - one per TU | All definitions must be identical |
| `constexpr` functions | Yes (implicitly inline) | All definitions must be identical |
| Enumerations | Yes - one per TU | All definitions must be identical |

### What Happens on ODR Violation

ODR violations are most commonly **undefined behavior with no diagnostic required**. The compiler and linker may silently produce wrong code. Here's what that looks like in practice:

```cpp
// file_a.cpp
struct Widget { int x; int y; };  // Widget has {x, y}

// file_b.cpp
struct Widget { int x; double y; };  // DIFFERENT Widget! ODR violation!
// No compiler error (different TUs), no linker error (classes aren't symbols)
// But the program has UNDEFINED BEHAVIOR
```

Notice that nothing stops this from compiling and linking - the mismatch is completely invisible to the toolchain.

---

## Self-Assessment

### Q1: List what can have multiple definitions across TUs vs what cannot

**Answer:**

**CAN have multiple definitions (one per TU, must be identical):**

- Class/struct/union definitions
- `inline` functions and variables
- Templates (function templates, class templates)
- `constexpr` / `consteval` functions (implicitly inline)
- Enumeration types
- Default arguments (must be identical)

**CANNOT have multiple definitions (exactly one in entire program):**

- Non-inline functions with external linkage
- Non-inline variables with external linkage
- Static data members (unless `inline` since C++17)

The key pattern to learn is: anything you'd normally put in a header falls into the "must be identical" camp. Anything you'd normally put in a `.cpp` file must have exactly one definition.

```cpp
// GOOD: inline function in header - multiple TUs include it
inline int helper() { return 42; }  // Same in every TU

// GOOD: template in header
template<typename T> T square(T x) { return x * x; }

// GOOD: class in header (implicitly allowed multiple definitions)
struct Point { double x, y; };

// BAD: non-inline function in header
// int compute() { return 42; }  // Linker error: multiple definitions

// GOOD FIX: make it inline
inline int compute() { return 42; }

// GOOD C++17: inline variable in header
inline int global_count = 0;  // Same object across all TUs
```

### Q2: Show an ODR violation with two definitions of the same class that differ in a member

The sneakiest ODR violations come from `#ifdef` guards that change class layout depending on which flags a translation unit was compiled with. This is a real-world pattern that causes silent corruption:

```cpp
// ---- config.h (BAD - conditional compilation changes the definition) ----
struct Config {
    int width;
    int height;
#ifdef ENABLE_ALPHA
    float alpha;  // This member exists only when ENABLE_ALPHA is defined!
#endif
};

// ---- file_a.cpp ----
// Compiled with -DENABLE_ALPHA
// Config = { int width, int height, float alpha }  - size: 12 bytes
#define ENABLE_ALPHA
#include "config.h"

void use_a(Config& c) {
    c.alpha = 0.5f;  // Accesses the alpha field
}

// ---- file_b.cpp ----
// Compiled WITHOUT -DENABLE_ALPHA
// Config = { int width, int height }  - size: 8 bytes
#include "config.h"

void use_b() {
    Config c{800, 600};
    use_a(c);  // UNDEFINED BEHAVIOR!
    // use_a thinks Config is 12 bytes, use_b thinks it's 8 bytes
    // Writing to c.alpha corrupts stack memory after c!
}

// This compiles and links WITHOUT any errors or warnings.
// The program silently produces wrong results or crashes.

// GOOD FIX: Never use #ifdef to change class layout in headers.
// Use runtime flags or separate types instead.
```

**How this works:**

- Each TU sees a **different definition** of `Config` - this violates ODR.
- The linker doesn't check class definitions (they're not symbols in the object file).
- One TU writes past the object's actual size, corrupting adjacent memory.
- This is an **extremely common bug** caused by inconsistent build flags.

### Q3: Explain how LTO exposes ODR violations that linkers normally miss

**Answer:**

Normal linking works at the **symbol level** - the linker sees function names and addresses but doesn't examine class layouts, inline function bodies, or template instantiation details. ODR violations in these (which are the most common) go completely undetected.

**Link-Time Optimization (LTO)** changes this: the compiler emits intermediate representation (IR) instead of machine code, and optimization happens across all TUs at link time. This means the optimizer sees ALL definitions simultaneously.

```cpp
// Normal compilation:
// file_a.cpp -> file_a.o (machine code - class layout is baked in)
// file_b.cpp -> file_b.o (machine code - different layout baked in)
// Linker: merges symbols, doesn't check class definitions
// -> ODR violation UNDETECTED

// With LTO (-flto):
// file_a.cpp -> file_a.o (LLVM IR / GCC GIMPLE - full type info preserved)
// file_b.cpp -> file_b.o (LLVM IR / GCC GIMPLE - full type info preserved)
// LTO optimizer: sees BOTH definitions of Config
// -> Can detect mismatch -> may warn or produce different (still wrong) code
```

LTO gives the toolchain enough information to notice the contradiction - though even then it's "may warn," not "must warn."

**Practical tools for ODR detection:**

- `-flto` with GCC/Clang - may detect some ODR violations during optimization
- `-fsanitize=undefined` - UBSan can catch some ODR issues at runtime
- `gold` linker with `--detect-odr-violations` flag
- MSVC `/LTCG` (Link-Time Code Generation)

**Even LTO doesn't catch everything** - ODR violations are still "no diagnostic required." The best defense is:

1. Keep headers consistent - don't use `#ifdef` to change class layouts
2. Use `inline` variables in headers (C++17) instead of separate definitions
3. Build all TUs with the same flags

---

## Notes

- ODR violations are the **most common form of silent undefined behavior** in large C++ projects.
- Header-only libraries avoid most ODR issues because every TU includes the same headers.
- `inline` functions/variables must be defined in headers - if defined differently in two TUs, it's an ODR violation.
- C++20 modules help prevent ODR violations by making definitions module-owned rather than textually included.
- ASAN/UBSAN cannot detect all ODR violations - structural mismatches are particularly hard to catch.
