# Know the initialization order of global and static local variables

**Category:** Core Language Fundamentals  
**Standard:** C++98, C++11, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/initialization>  

---

## Topic Overview

### Initialization Phases

C++ global/static variables are initialized in two phases. The split between them is what causes the infamous static initialization order fiasco:

| Phase | When | What happens |
| --- | --- | --- |
| **Static initialization** | Before `main()`, at load time | Zero-initialization, then constant initialization |
| **Dynamic initialization** | Before `main()` (or first use for locals) | Constructors, function calls, complex expressions |

### Static Initialization (Safe)

1. **Zero-initialization**: All globals/statics with static storage duration are zero-initialized first.
2. **Constant initialization**: If the initializer is a constant expression, evaluated at compile time.

These two steps are entirely safe because they happen deterministically, without depending on any runtime state:

```cpp
int x;                    // Zero-initialized to 0
constexpr int y = 42;     // Constant-initialized, guaranteed at compile time
const double pi = 3.14;   // Constant-initialized (if compiler can evaluate)
```

### Dynamic Initialization (Dangerous)

For globals initialized by runtime expressions (constructors, function calls):

```cpp
std::string name = "hello";    // Dynamic: std::string constructor runs
int z = compute_value();       // Dynamic: function called at runtime
```

**The problem:** The order of dynamic initialization across translation units (TUs) is **unspecified**.

### The Static Initialization Order Fiasco

If global `A` in file1.cpp depends on global `B` in file2.cpp, and B hasn't been initialized yet when A's constructor runs - you get undefined behavior. The linker decides the order, and you don't control the linker:

```cpp
file1.cpp: extern Logger logger;     // defined in file2.cpp  
file1.cpp: Config config(logger);    // uses logger - but is logger ready?
```

### Meyers Singleton - The Fix

Wrapping a global in a function-local static sidesteps the problem entirely - the variable is guaranteed to exist by the time you first use it:

```cpp
Logger& get_logger() {
    static Logger instance("app.log");  // Initialized on first call
    return instance;                     // Thread-safe since C++11
}
```

**Function-local statics are initialized the first time control passes through the declaration.** This guarantees they're ready when needed, regardless of initialization order.

### `constinit` (C++20) - Guarantee Static Initialization

`constinit` turns a potential runtime bug into a compile-time error - the best kind of fix:

```cpp
constinit int counter = 0;                    // OK: constant initializer
constinit const char* name = "hello";         // OK: constant initializer
// constinit std::string s = "hello";         // ERROR: dynamic initialization required
```

`constinit` ensures a variable is **constant-initialized** (no runtime constructor). If the initializer isn't a constant expression, you get a compile error instead of a runtime surprise.

---

## Self-Assessment

### Q1: Explain the static initialization order fiasco and show a bug

The bug is subtle: both globals look fine in isolation, but their relative initialization order is unspecified by the standard:

```cpp
// ==== file: registry.cpp ====
#include <map>
#include <string>
#include <iostream>

// Global map - dynamically initialized (constructor runs before main)
std::map<std::string, int> registry;

void register_item(const std::string& name, int value) {
    registry[name] = value;
}

int get_item(const std::string& name) {
    return registry.at(name);
}

// ==== file: feature.cpp ====
// This global's initializer runs before main() - but WHEN relative to registry?
struct FeatureRegistrar {
    FeatureRegistrar() {
        // BUG: registry might not be constructed yet!
        // If feature.cpp is initialized BEFORE registry.cpp - crash or corruption
        register_item("feature_x", 42);
    }
};

FeatureRegistrar auto_register;  // Dynamic initialization - order vs registry is UNSPECIFIED

// ==== main.cpp ====
int main() {
    // If we're lucky, registry was initialized first and this works.
    // If not, the program crashed in FeatureRegistrar's constructor.
    std::cout << "feature_x = " << get_item("feature_x") << "\n";
    return 0;
}
```

The bug may hide for a long time - it works on your machine, then breaks when you add a file or change link order.

The bug:

- `registry` (in registry.cpp) and `auto_register` (in feature.cpp) are both globals with dynamic initialization.
- The C++ standard doesn't specify the order of dynamic initialization **across translation units**.
- If `auto_register` initializes before `registry`, it calls `register_item()` on an unconstructed `std::map` - undefined behavior (crash, corruption, or appears to work).
- The order may change when you add/remove files, change link order, or switch compilers.

### Q2: Use the Meyers singleton to fix it

The fix is elegant: instead of a global object, expose a global function that contains a local static. The first caller pays the initialization cost; everyone else gets the already-initialized value:

```cpp
#include <map>
#include <string>
#include <iostream>

// Fix: wrap the global in a function - initialized on first use
std::map<std::string, int>& get_registry() {
    static std::map<std::string, int> registry;  // Created on first call
    return registry;                               // Thread-safe since C++11
}

void register_item(const std::string& name, int value) {
    get_registry()[name] = value;  // get_registry() guarantees map exists
}

int get_item(const std::string& name) {
    return get_registry().at(name);
}

// Now feature.cpp can safely register at global init time:
struct FeatureRegistrar {
    FeatureRegistrar() {
        register_item("feature_x", 42);  // SAFE: get_registry() creates the map on first call
    }
};

FeatureRegistrar auto_register;  // OK - no fiasco

int main() {
    std::cout << "feature_x = " << get_item("feature_x") << "\n"; // 42
    register_item("feature_y", 100);
    std::cout << "feature_y = " << get_item("feature_y") << "\n"; // 100
    return 0;
}
```

How it works:

- `static` local variables are initialized **the first time control flow reaches the declaration**.
- This means `registry` is guaranteed to exist before anyone uses it.
- C++11 guarantees this initialization is **thread-safe** (the compiler inserts a hidden guard variable and mutex-like mechanism).
- **Destruction order:** Static locals are destroyed in reverse order of construction. Accessing the registry during program shutdown could still be a problem (destruction order fiasco).

### Q3: Demonstrate `constinit` (C++20)

`constinit` is a compile-time safety net. If your initializer requires runtime code, you get an error at build time rather than a mysterious crash at startup:

```cpp
#include <iostream>
#include <array>

// constinit ensures the variable is initialized at compile time (static initialization)
// This PREVENTS the initialization order fiasco for this variable

constinit int global_count = 0;                // OK: 0 is a constant expression
constinit const char* app_name = "MyApp";      // OK: string literal is constant
constinit std::array<int, 3> defaults = {1, 2, 3}; // OK: aggregate of constants

// constinit does NOT mean const - the variable can be modified at runtime!
void increment() {
    global_count++;  // OK - constinit only constrains initialization, not usage
}

// constinit catches dynamic initialization at compile time:
int compute() { return 42; }
// constinit int bad = compute();  // ERROR: compute() is not constexpr
//                                 // error: 'bad' does not have a constant initializer

// But constexpr functions work:
constexpr int compute_const() { return 42; }
constinit int good = compute_const();  // OK: constexpr function at compile time

// Real-world: ensure a lookup table is compile-time initialized
constexpr std::array<int, 256> build_ascii_table() {
    std::array<int, 256> table{};
    for (int i = 0; i < 256; ++i) {
        table[i] = (i >= 'A' && i <= 'Z') ? i + 32 : i;  // to_lower
    }
    return table;
}

constinit auto ascii_lower = build_ascii_table();  // Guaranteed compile-time init!

int main() {
    std::cout << "global_count = " << global_count << "\n";  // 0
    increment();
    increment();
    std::cout << "global_count = " << global_count << "\n";  // 2

    // constinit prevents the fiasco: if initialization can't be done at compile time,
    // you get a compile error - not a runtime bug.

    // Use the lookup table:
    char c = 'H';
    std::cout << "Lower of 'H': " << static_cast<char>(ascii_lower[c]) << "\n"; // h

    return 0;
}
```

The three keywords are often confused. Here's the mental model:

**`constinit` vs `constexpr` vs `const`:**

| Keyword | Compile-time init? | Immutable? | Can modify later? |
| --- | --- | --- | --- |
| `constinit` | Required | No | Yes |
| `constexpr` | Required | Yes (implicitly const) | No |
| `const` | Not required | Yes | No |
| (nothing) | Not required | No | Yes |

`constinit` = "I want compile-time initialization but runtime mutability."

---

## Notes

- The static initialization order fiasco is one of the most insidious C++ bugs - it may work for years then break when you change link order.
- Fix 1: Meyers singleton (function-local static) - works pre-C++20.
- Fix 2: `constinit` (C++20) - guarantees compile-time initialization, compile error if impossible.
- Function-local statics are thread-safe in C++11+.
- Beware the **destruction** order fiasco too - static locals are destroyed in reverse construction order.
