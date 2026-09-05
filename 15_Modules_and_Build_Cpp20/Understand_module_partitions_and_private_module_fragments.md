# Understand module partitions and private module fragments

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/modules>  

---

## Topic Overview

C++20 modules support **partitions** to split a large module into smaller files, and **private module fragments** to hide implementation details within the interface unit.

These two features solve the same underlying problem - keeping implementation details private - but at different scales. Partitions are for large modules that naturally belong in multiple files. Private fragments are for small, self-contained modules where you want everything in one file but still want a clean separation between the public interface and the private implementation.

### Module Partition Types

Here's how the pieces fit together for a `graphics` module split across partitions:

```cpp
+-----------------------------------------------------+
|                    Module "graphics"                    |
|                                                         |
|  +-----------------+  Primary interface unit             |
|  | graphics.cppm   |  export module graphics;            |
|  |                 |  export import :shapes;  <- re-export|
|  |                 |  import :internal;       <- internal |
|  +-----------------+                                     |
|       |                                                  |
|  +----+-----------+  Interface partition (re-exported)   |
|  | shapes.cppm     |  export module graphics:shapes;     |
|  | (export visible)|  export class Circle { ... };       |
|  +-----------------+                                     |
|                                                         |
|  +-----------------+  Implementation partition           |
|  | internal.cppm   |  module graphics:internal;          |
|  | (NOT exported)  |  void helper() { ... }              |
|  +-----------------+                                     |
+-----------------------------------------------------+

Importer sees: graphics (with shapes re-exported)
Importer does NOT see: :internal partition
```

### Private Module Fragment

A **private module fragment** (`module :private;`) allows hiding implementation in the same file as the interface. Everything above the `module :private;` line is part of the public interface. Everything below it is completely hidden from importers - and importantly, changes to the private fragment do not force importers to recompile:

```cpp
export module single_file;
export int compute(int x);  // declaration visible to importers

module :private;            // everything below is hidden
int compute(int x) {        // definition NOT visible to importers
    return x * x + 1;      // changing this does NOT recompile importers
}
```

---

## Self-Assessment

### Q1: Split a large module into partitions using `export module foo:bar;` and `import foo:bar;`

Here's a `math` module split into three partitions. Notice the roles: the `:arithmetic` partition holds the public math operations, `:detail` holds private helpers, and the primary `math.cppm` interface re-exports `:arithmetic` (making it visible to consumers) while only importing `:detail` internally:

#### File: math.cppm (primary interface)

```cpp
export module math;

// Re-export the "arithmetic" partition - importers see it
export import :arithmetic;

// Import the "detail" partition - internal use only
import :detail;

// The primary interface can add more exports
export double distance(double x, double y) {
    return detail_sqrt(x * x + y * y);  // uses :detail
}
```

#### File: math_arithmetic.cppm (interface partition)

```cpp
export module math:arithmetic;

// These are visible to anyone who imports "math"
// because the primary interface does: export import :arithmetic;

export int add(int a, int b) { return a + b; }
export int subtract(int a, int b) { return a - b; }
export int multiply(int a, int b) { return a * b; }

export double divide(int a, int b) {
    if (b == 0) throw "Division by zero";
    return static_cast<double>(a) / b;
}
```

#### File: math_detail.cppm (implementation partition)

```cpp
module;  // global module fragment for C headers
#include <cmath>

module math:detail;  // NOT "export module" - internal partition

// These functions are visible to other parts of module "math"
// but NOT to importers of "math"
double detail_sqrt(double x) {
    return std::sqrt(x);
}

double detail_clamp(double x, double lo, double hi) {
    return (x < lo) ? lo : (x > hi) ? hi : x;
}
```

#### File: main.cpp (consumer)

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
import math;
#include <iostream>

int main() {
    // From :arithmetic partition (re-exported)
    std::cout << "3 + 4 = " << add(3, 4) << '\n';
    std::cout << "10 / 3 = " << divide(10, 3) << '\n';

    // From primary interface
    std::cout << "distance(3,4) = " << distance(3.0, 4.0) << '\n';

    // detail_sqrt(9);  // ERROR: not exported, not visible
}
// Expected output:
// 3 + 4 = 7
// 10 / 3 = 3.33333
// distance(3,4) = 5
```

Here's the quick reference for the partition syntax rules:

- `export module M:P;` - **interface partition** (can be re-exported)
- `module M:P;` - **implementation partition** (internal only)
- `export import :P;` in primary - **re-export** a partition to importers
- `import :P;` in primary - **internal import** (not visible to importers)

### Q2: Use a private module fragment (`module :private;`) to hide implementation details

A private module fragment is ideal when you want a single-file module with a clear interface/implementation split. The `Counter` example below has the entire public contract declared above the `module :private;` line, and all the method bodies below it. Importers only see the declaration section:

```cpp
// counter.cppm - single-file module with private fragment
export module counter;

// === Public interface (visible to importers) ===
export class Counter {
public:
    Counter();
    void increment();
    void decrement();
    int value() const;
    void reset();

private:
    int count_;
    int max_seen_;  // track highest value
};

export Counter make_counter(int initial = 0);

// === Private fragment (hidden from importers) ===
module :private;

Counter::Counter() : count_(0), max_seen_(0) {}

void Counter::increment() {
    ++count_;
    if (count_ > max_seen_) max_seen_ = count_;
}

void Counter::decrement() {
    if (count_ > 0) --count_;
}

int Counter::value() const { return count_; }
void Counter::reset() { count_ = 0; }

Counter make_counter(int initial) {
    Counter c;
    for (int i = 0; i < initial; ++i) c.increment();
    return c;
}
```

The consumer just sees and uses the clean public interface:

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
import counter;
#include <iostream>

int main() {
    auto c = make_counter(5);
    std::cout << "Initial: " << c.value() << '\n';

    c.increment();
    c.increment();
    std::cout << "After +2: " << c.value() << '\n';

    c.decrement();
    std::cout << "After -1: " << c.value() << '\n';

    c.reset();
    std::cout << "After reset: " << c.value() << '\n';
}
// Expected output:
// Initial: 5
// After +2: 7
// After -1: 6
// After reset: 0
```

The key benefit worth remembering: changing the implementation in `module :private;` does NOT require recompilation of importers - only the module itself is recompiled. This is analogous to the pimpl idiom but built into the language.

### Q3: Explain the difference between internal linkage in a module vs a named namespace

This is one of the genuinely tricky conceptual corners of C++20 modules. `static` and anonymous namespaces still mean what they always meant - TU-local. But non-exported names in a module unit get a new kind of linkage that spans the whole module. The contrast:

| Concept | `static` / anonymous namespace | Module non-exported names |
| --- | --- | --- |
| Scope | Single TU only | Entire module (all partitions) |
| Linkage | Internal (no linkage) | Module linkage (new in C++20) |
| Visible to | Only the defining TU | All TUs in same module |
| Visible to importers | No | No |
| Name mangling | TU-specific | Module-scoped |

You can see this in action with two partitions of the same module. `module_internal` (no `static`, no anonymous namespace) is visible across both, but `tu_local` and `anon_val` are not:

```cpp
// partition_a.cppm
export module mylib:a;

static int tu_local = 1;       // internal linkage: only this TU
int module_internal = 2;       // module linkage: visible within mylib
export int public_val = 3;     // external linkage: visible to importers

namespace {
    int anon_val = 4;          // internal linkage: only this TU
}

// partition_b.cppm
module mylib:b;
import :a;

void test() {
    // tu_local;    // ERROR: internal linkage, not visible
    // anon_val;    // ERROR: internal linkage, not visible
    int x = module_internal;  // OK: module linkage, same module
    int y = public_val;       // OK: exported
}
```

And from a consumer, only `public_val` gets through:

```cpp
import mylib;

void consumer() {
    // tu_local;         // ERROR: internal linkage
    // anon_val;         // ERROR: internal linkage
    // module_internal;  // ERROR: module linkage (not exported)
    int z = public_val;  // OK: exported with external linkage
}
```

**Module linkage** is a new C++20 concept: names are visible across all TUs of a module but invisible outside it. This is stricter than `namespace detail {}` (which is just a convention, not enforced).

---

## Notes

- Partitions cannot be imported by external consumers - only the primary module name is importable.
- A module can have **at most one** private module fragment, and only in single-file modules.
- The private module fragment cannot be used together with partitions (they solve the same problem differently).
- Use partitions for **large modules** that need separate files; use private fragments for **small modules** that fit in one file.
- `module :private;` must appear after all export declarations in the file.
