# Understand constinit (C++20) to guarantee static initialization

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constinit>  

---

## Topic Overview

`constinit` is a C++20 keyword that guarantees a variable with static or thread-local storage
duration is **initialized at compile time** (constant/zero initialization), preventing the
**static initialization order fiasco** (SIOF). The important twist: unlike `constexpr`, it
doesn't make the variable immutable - you can still modify it at runtime.

### Key Distinction: constinit vs constexpr vs const

If the table feels like a lot, the one-liner is: `constinit` = "initialize at compile time,
but allow runtime mutation."

| Keyword | Compile-time init? | Const after init? | Can be modified at runtime? |
| --- | --- | --- | --- |
| `const` | Not guaranteed | Yes | No |
| `constexpr` | Yes (required) | Yes (implied const) | No |
| `constinit` | Yes (required) | No (not implied const) | Yes |

### Basic Usage

Any initializer that isn't a constant expression is rejected at compile time - that's the
whole point:

```cpp
// OK: literal value - constant initialization
constinit int counter = 0;         // initialized at compile time, but mutable!

// OK: constexpr function result
constexpr int compute() { return 42; }
constinit int value = compute();   // initialized at compile time

// ERROR: runtime function
int runtime_func();
// constinit int bad = runtime_func();   // ERROR: not a constant expression
```

### Where constinit Can Be Used

- **Global variables** (namespace scope)
- **Static local variables** (`static constinit` inside a function)
- **Thread-local variables** (`thread_local constinit`)
- **Static class members** (`static constinit`)

```cpp
struct Config {
    static constinit int max_threads;  // declaration
};
constinit int Config::max_threads = 8; // definition - constant init

void func() {
    static constinit int call_count = 0;  // constant init
    call_count++;  // OK: constinit does NOT make it const
}
```

### Preventing the Static Initialization Order Fiasco

Without `constinit`, the initialization order of globals across translation units is
undefined - `global_b` might see an uninitialized `global_a`. With `constinit`, the
compiler rejects any initializer that can't be evaluated at compile time, so the
problem is caught immediately rather than manifesting as a mysterious runtime bug:

```cpp
// file_a.cpp
constinit int global_a = 10;          // guaranteed: initialized before any dynamic init

// file_b.cpp
extern constinit int global_a;
constinit int global_b = global_a;    // ERROR if global_a is not constexpr-available
// If global_a is truly constinit with a literal, this works.
// If global_a depended on a runtime value, constinit catches the error at compile time.
```

### constinit with thread_local

Thread-local variables risk running a dynamic initializer on every new thread. `constinit`
ensures the initial value is baked in at compile time instead, so each thread starts with
a known value at zero cost:

```cpp
// thread_local variables risk dynamic initialization on each thread creation
// constinit ensures the initial value is baked in at compile time

constinit thread_local int tls_counter = 0;   // zero-init at compile time
// Each thread starts with tls_counter == 0, no dynamic initializer needed
```

---

## Self-Assessment

### Q1: Add constinit to a global variable and verify the compiler rejects runtime initialization

Notice that `x` gets modified at runtime just fine - `constinit` only constrains the
*initial* value, not subsequent assignments:

```cpp
#include <iostream>
#include <string>

// OK: compile-time initialization
constinit int x = 42;
constinit double pi = 3.14159;
constinit const char* greeting = "Hello";

// ERROR: runtime initialization
// std::string requires a constructor call at runtime
// constinit std::string name = "World";   // ERROR: not a constant expression

// ERROR: function result that isn't constexpr
int get_value() { return 100; }
// constinit int y = get_value();   // ERROR: get_value() is not constexpr

// OK: constexpr function
constexpr int square(int n) { return n * n; }
constinit int sq = square(7);   // OK: 49, evaluated at compile time

int main() {
    std::cout << "x  = " << x << "\n";    // 42
    std::cout << "sq = " << sq << "\n";    // 49

    // constinit does NOT mean const - we can modify at runtime:
    x = 100;
    std::cout << "x after modification: " << x << "\n";   // 100
}
```

**How it works:**

- `constinit int x = 42;` - literal initialization is always constant -> accepted.
- `constinit std::string name = "World";` - `std::string`'s constructor runs at runtime -> rejected by the compiler.
- The compiler error message will say something like "variable does not have a constant initializer."

### Q2: Explain the difference between constinit, constexpr, and const for global variables

Here's all three side by side so the contrast is clear:

```cpp
#include <iostream>

// 1) const - promises immutability, but initialization MAY be at runtime
const int a = 42;              // happens to be compile-time (integral constant)
int runtime();
const int b = runtime();       // OK: const allows runtime initialization

// 2) constexpr - MUST be initialized at compile time AND is const
constexpr int c = 42;         // compile-time, immutable
// constexpr int d = runtime();  // ERROR: not a constant expression
// c = 10;                       // ERROR: constexpr implies const

// 3) constinit - MUST be initialized at compile time but is NOT const
constinit int e = 42;         // compile-time init, but mutable
// constinit int f = runtime();  // ERROR: not a constant expression

int main() {
    e = 100;   // OK! constinit doesn't make it const
    std::cout << e << "\n";   // 100

    // Summary table:
    // | Keyword   | Compile-time init | Immutable | Use case                     |
    // |-----------|-------------------|-----------|------------------------------|
    // | const     | Not required      | Yes       | General immutability         |
    // | constexpr | Required          | Yes       | Compile-time constants       |
    // | constinit | Required          | No        | Prevent SIOF, allow mutation |
}
```

The key insight: `constinit` solves a specific problem - guaranteeing that a global/static
variable doesn't suffer from the static initialization order fiasco, while still being
modifiable at runtime (e.g., a counter, a cache).

### Q3: Use constinit to prevent the static initialization order fiasco for a global resource

The pattern below shows the problem, the fix, and how to handle values that genuinely need
runtime initialization (the Meyers singleton):

```cpp
#include <iostream>

// === Problem: SIOF without constinit ===
// file1.cpp:  int global_a = expensive_init();       // dynamic init
// file2.cpp:  int global_b = global_a + 1;           // may see 0 (uninitialized!)
// Order of dynamic initialization across TUs is UNDEFINED.

// === Solution: constinit catches the problem at compile time ===

// Safe: everything is constant-initialized
constinit int max_connections = 100;
constinit int timeout_ms = 5000;
constinit const char* app_name = "MyServer";

// If someone tries to use a runtime value:
// int compute_default() { return 42; }
// constinit int bad = compute_default();  // COMPILE ERROR - caught immediately!

// For values that genuinely need runtime init, use the Meyers singleton instead:
struct DatabasePool {
    int pool_size;
    // ... expensive to construct
};

DatabasePool& get_pool() {
    static DatabasePool pool{max_connections};   // lazy init, thread-safe
    return pool;
}

int main() {
    std::cout << "App: " << app_name << "\n";
    std::cout << "Max connections: " << max_connections << "\n";

    // Modify at runtime (constinit allows this)
    max_connections = 200;
    std::cout << "Updated max: " << max_connections << "\n";

    auto& pool = get_pool();
    std::cout << "Pool size: " << pool.pool_size << "\n";
}
```

**How it works:**

- `constinit` on globals guarantees they are initialized during the **constant initialization** phase (before `main()`), eliminating SIOF.
- If a developer accidentally changes an initializer to something runtime-computed, the compiler immediately errors out.
- For truly dynamic resources, combine `constinit` for simple config values with the Meyers singleton pattern for complex objects.

---

## Notes

- `constinit` is only for variables with **static** or **thread-local** storage duration - not for local automatic variables.
- You can combine `constinit` with `const`: `constinit const int x = 42;` - compile-time init AND immutable.
- `constinit` does not affect the variable's linkage - it's purely about initialization timing.
- The compiler checks `constinit` at the **point of initialization**, so it catches SIOF at compile time rather than at runtime.
- GCC, Clang, and MSVC all support `constinit` since their C++20 implementations.
