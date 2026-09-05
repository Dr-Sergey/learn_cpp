# Understand linkage: internal, external, no linkage, and module linkage

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/storage_duration>  

---

## Topic Overview

**Linkage** determines whether a name in one translation unit (TU) can refer to the same entity in another TU. C++ defines four kinds of linkage.

### The Four Linkage Types

| Linkage | Meaning | Visible Across TUs? |
| --- | --- | --- |
| **External** | Name refers to the same entity in all TUs | Yes |
| **Internal** | Name refers to a unique entity per TU | No |
| **No linkage** | Name is local to its scope (block-scope) | No |
| **Module** (C++20) | Name is visible within the same module but not outside it | Within module only |

### Default Linkage Rules

If the table feels like a lot, just remember the one surprising rule: `const` at namespace scope is **internal** in C++, not external. Everything else behaves the way you'd intuitively expect.

| Entity | Default Linkage |
| --- | --- |
| Functions at namespace scope | External |
| Non-const variables at namespace scope | External |
| `const` / `constexpr` variables at namespace scope | **Internal** (C++ only; external in C) |
| `inline` variables/functions | External |
| `static` at namespace scope | Internal |
| Entities in unnamed namespace | Internal |
| Local variables | No linkage |
| Class members | External (same as the class) |

### Making Linkage Explicit

Here's how you spell out each kind of linkage in code. Notice that the unnamed namespace is the modern C++ way to get internal linkage - it also works for classes and templates, not just plain functions.

```cpp
// External linkage (default for functions)
void f();                    // external
extern int x;               // declaration, external

// Internal linkage
static void g() {}          // old C-style internal linkage
namespace {                  // C++ preferred: unnamed namespace
    void h() {}             // internal linkage
    int secret = 42;        // internal linkage
}

// const at namespace scope = internal by default
const int MAX = 100;        // internal linkage in C++
extern const int MAX2 = 200; // explicit external linkage
```

The `static` keyword at namespace scope is old-school C style. Prefer the unnamed namespace in new code - it's more general.

### Module Linkage (C++20)

C++20 modules introduce a fourth option that sits between external and internal.

```cpp
// mymodule.cppm
export module mymodule;

export void public_func();     // external linkage - visible to importers
void module_internal_func();   // module linkage - visible within module only
static void file_local();      // internal linkage - this TU only

// Importers can call public_func() but NOT module_internal_func()
```

Module linkage sits between external and internal: the name is shared across all TUs of the same module but not exported to importers.

---

## Self-Assessment

### Q1: Show that a `const` variable at namespace scope has internal linkage by default in C++

The key thing to watch here is the addresses. Both files define `const int value = 42`, but because `const` gives internal linkage, each TU gets its own copy - the linker never merges them.

```cpp
// ---- file_a.cpp ----
#include <iostream>

const int value = 42;   // internal linkage in C++!

void print_a() {
    std::cout << "file_a: value = " << value << " at " << &value << "\n";
}

// ---- file_b.cpp ----
#include <iostream>

const int value = 42;   // DIFFERENT entity - also internal linkage

void print_b() {
    std::cout << "file_b: value = " << value << " at " << &value << "\n";
}

// ---- main.cpp ----
void print_a();
void print_b();

int main() {
    print_a();  // value at address 0x...AAA
    print_b();  // value at address 0x...BBB - different address!
    // Both are 42, but they are SEPARATE objects because const = internal linkage
}

// To make it external (single shared instance):
// file_a.cpp: extern const int shared_value = 42;  // extern overrides internal
// file_b.cpp: extern const int shared_value;        // declaration only
```

**How it works:**

- In C++, `const` at namespace scope gives the variable **internal linkage** (as if `static`).
- This means each TU has its own independent copy - no linker error, but separate objects.
- This differs from C, where `const` has external linkage by default.
- Use `extern const` to force external linkage if a single shared instance is needed.

### Q2: Explain what module linkage means in C++20 and how it differs from internal/external

**Answer:**

| Linkage | Scope of Visibility | Redefinable in Other TUs? |
| --- | --- | --- |
| **External** | All TUs in the program | No - ODR applies globally |
| **Module** | All TUs of the **same module** | Yes - different modules can have same name |
| **Internal** | Only the **current TU** | Yes - each TU has its own entity |

Think of module linkage as "package-private" - helpers your module implementation units can share, but that importers never see. Here's the breakdown in code:

```cpp
// ---- mylib.cppm ----
export module mylib;

export int public_api() { return 1; }      // external linkage -> everyone sees it

int helper() { return 2; }                  // module linkage -> only mylib sees it

namespace {
    int truly_private() { return 3; }       // internal linkage -> this file only
}

// ---- mylib_impl.cppm ----
module mylib;   // part of same module

void use() {
    public_api();     // OK - external linkage
    helper();         // OK - module linkage, same module
    // truly_private(); // ERROR - internal linkage, different TU
}

// ---- consumer.cpp ----
import mylib;

void consumer() {
    public_api();     // OK - exported, external linkage
    // helper();      // ERROR - module linkage, not exported
}
```

**Key insight:** Module linkage is the C++20 way to have "package-private" functions - visible within your module implementation units but invisible to importers. This replaces the common pattern of putting helpers in anonymous namespaces or `detail::` namespaces.

### Q3: Use the unnamed namespace to give a function internal linkage and show it cannot be ODR-used from another TU

The unnamed namespace makes `compute_secret` and `InternalHelper` invisible to the linker. Another TU that tries to declare them `extern` will get an undefined-reference error at link time, not just a compile error.

```cpp
// ---- helpers.cpp ----
#include <iostream>

namespace {
    // Internal linkage - invisible outside this TU
    int compute_secret() {
        return 42;
    }

    class InternalHelper {
        int x;
    public:
        InternalHelper(int v) : x(v) {}
        void print() { std::cout << "Internal: " << x << "\n"; }
    };
}

// This function has external linkage - it CAN be called from other TUs
void use_secret() {
    std::cout << "Secret: " << compute_secret() << "\n";
    InternalHelper h(99);
    h.print();
}

// ---- main.cpp ----
// extern int compute_secret();  // ERROR: linker can't find it - internal linkage
void use_secret();               // OK: this has external linkage

int main() {
    use_secret();
    // compute_secret();  // would fail at link time
}
```

**How it works:**

- The unnamed namespace wraps `compute_secret` and `InternalHelper` with internal linkage.
- They exist only within `helpers.cpp` - the linker has no symbol for them.
- Another TU declaring `extern int compute_secret()` will get a **linker error** (undefined reference).
- The unnamed namespace is preferred over `static` for functions because it also works for classes, templates, and enums.

---

## Notes

- `static` at namespace scope and unnamed namespace achieve the same effect for functions/variables - but unnamed namespace is preferred in modern C++.
- `inline` functions/variables always have **external** linkage - that's how they can be defined in headers without ODR violations.
- `constexpr` functions have external linkage; `constexpr` variables at namespace scope have **internal** linkage (same as `const`).
- Module linkage (C++20) finally gives C++ proper encapsulation at the module level, similar to Java's package-private or Rust's `pub(crate)`.
- The ODR (One Definition Rule) applies differently per linkage: external names must have exactly one definition across all TUs; internal names can be redefined independently per TU.
