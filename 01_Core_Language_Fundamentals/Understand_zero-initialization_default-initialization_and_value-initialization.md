# Understand zero-initialization, default-initialization, and value-initialization

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/initialization>  

---

## Topic Overview

C++ has multiple forms of initialization, each with different rules for what value an object gets. Confusing them is a common source of bugs - reading uninitialized memory is undefined behavior, and the compiler won't always warn you.

### The Three Core Forms

The table below is the quick reference. Notice that whether you get garbage or zero depends entirely on *how* you write the declaration, not on what type you're using:

| Form | Syntax | Effect on `int x` | Effect on `struct S { int a; }` |
| --- | --- | --- | --- |
| **Default-init** | `int x;` (local) | **Indeterminate** (garbage) | Members default-initialized (garbage for scalars) |
| **Value-init** | `int x{};` or `int x = int();` | **Zero** (0) | Members value-initialized (zero for scalars) |
| **Zero-init** | static/global `int x;` | **Zero** (0) | All bytes zero |

### Default-Initialization

Occurs when a variable is declared with **no initializer** in local scope. For scalars, this means the memory holds whatever was there before - reading it is UB:

```cpp
void func() {
    int x;          // Default-init -> indeterminate value (garbage)
    double d;       // Default-init -> indeterminate value
    int* p;         // Default-init -> indeterminate value
    std::string s;  // Default-init -> calls default ctor -> empty string ""
}
```

- For scalar types (int, float, pointers): **no initialization** - reading is UB!
- For class types with a default constructor: the constructor is called.

### Value-Initialization

Occurs with empty braces `{}` or parentheses `()`. This is the safe default for local variables when you want a clean starting state:

```cpp
void func() {
    int x{};           // Value-init -> 0
    int y = int();     // Value-init -> 0
    double d{};        // Value-init -> 0.0
    int* p{};          // Value-init -> nullptr
    int arr[5]{};      // All elements value-initialized -> {0, 0, 0, 0, 0}

    struct S { int a; double b; };
    S s{};             // Value-init -> {0, 0.0}
}
```

### Zero-Initialization

Occurs automatically for **static** and **global** variables. You can always rely on globals being zero before `main` starts:

```cpp
int global;                  // Zero-init -> 0
static double static_var;   // Zero-init -> 0.0

void func() {
    static int local_static;  // Zero-init -> 0
    int local;                // NOT zero-init -> indeterminate!
}
```

### The Initialization Decision Tree

This tree captures the full ruleset for deciding what kind of initialization a variable gets:

```cpp
Variable declared:
├── Global/static?
│   └── Zero-initialized (then default-init if class type)
├── Has initializer?
│   ├── T x{};    -> Value-initialization
│   ├── T x{val}; -> Direct-list-initialization
│   ├── T x(val); -> Direct-initialization
│   └── T x = v;  -> Copy-initialization
└── No initializer: T x;
    ├── Class type? -> Default constructor called
    └── Scalar type? -> INDETERMINATE (UB to read)
```

### `new` Expressions

The same distinction applies to heap allocation - the parentheses or braces make a real difference:

```cpp
int* a = new int;     // Default-init -> indeterminate
int* b = new int();   // Value-init -> 0
int* c = new int{};   // Value-init -> 0
int* d = new int{42}; // Direct-list-init -> 42

struct S { int x; };
S* p1 = new S;     // Default-init -> x is indeterminate
S* p2 = new S();   // Value-init -> x is 0
S* p3 = new S{};   // Value-init -> x is 0
```

---

## Self-Assessment

### Q1: Show that `int x;` is indeterminate while `int x{};` is zero

This example makes the difference concrete - and shows that the same rule applies to structs, arrays, and heap allocations, not just simple ints:

```cpp
#include <iostream>
#include <cstring>

struct Pod {
    int a;
    double b;
    char c[8];
};

int main() {
    // 1. Default-initialized local int - INDETERMINATE
    int x;  // Indeterminate value - reading it is UB!
    // std::cout << x;  // UB: reading uninitialized variable

    // 2. Value-initialized local int - ZERO
    int y{};
    std::cout << "y (value-init): " << y << "\n";  // 0

    // 3. Same with struct
    Pod p1;   // Default-init: all members indeterminate
    Pod p2{}; // Value-init: all members zero

    std::cout << "p2.a: " << p2.a << "\n";  // 0
    std::cout << "p2.b: " << p2.b << "\n";  // 0.0

    // Verify p2 is all zeros
    unsigned char buf[sizeof(Pod)];
    std::memcpy(buf, &p2, sizeof(Pod));
    bool all_zero = true;
    for (size_t i = 0; i < sizeof(Pod); ++i) {
        if (buf[i] != 0) { all_zero = false; break; }
    }
    std::cout << "p2 all zeros: " << std::boolalpha << all_zero << "\n";  // true

    // 4. Array versions
    int arr1[5];    // Default-init: all elements indeterminate
    int arr2[5]{};  // Value-init: all elements zero
    for (int v : arr2) std::cout << v << " ";  // 0 0 0 0 0
    std::cout << "\n";

    // 5. new expressions
    int* a = new int;     // Indeterminate
    int* b = new int();   // Zero: 0
    int* c = new int{};   // Zero: 0
    std::cout << "*b = " << *b << ", *c = " << *c << "\n";  // 0, 0
    delete a; delete b; delete c;
}
```

**How this works:**

- `int x;` inside a function: no initialization happens. The memory contains whatever was there before - reading it is **undefined behavior**.
- `int x{};` performs value-initialization: for scalars, this means zero.
- For structs without constructors, value-init zeros all members.
- `new int` vs `new int()` - the parentheses (or braces) trigger value-init instead of default-init.

### Q2: Explain what happens when a struct with no user-provided constructor is value-initialized

**Answer:**

When you write `S s{};` for a struct `S` with no user-provided constructor, the key question is whether a user-provided constructor exists at all - because that determines whether zero-initialization runs first:

```cpp
#include <iostream>

// Case 1: No constructor at all (aggregate)
struct Aggregate {
    int x;
    double y;
    char z;
};

// Case 2: Defaulted constructor
struct DefaultedCtor {
    int x;
    double y;
    DefaultedCtor() = default;  // Not user-provided
};

// Case 3: User-provided constructor
struct UserCtor {
    int x;
    double y;
    UserCtor() {}  // User-provided! Does NOT zero members.
};

int main() {
    // Aggregate: value-init -> zero-init all members
    Aggregate a{};
    std::cout << "Aggregate: x=" << a.x << " y=" << a.y << "\n";  // 0, 0.0

    // Defaulted: value-init -> zero-init all members
    DefaultedCtor d{};
    std::cout << "Defaulted: x=" << d.x << " y=" << d.y << "\n";  // 0, 0.0

    // User-provided: value-init -> calls UserCtor() -> members NOT initialized!
    UserCtor u{};
    // u.x and u.y are INDETERMINATE - the constructor body is empty
    // std::cout << u.x;  // UB
    std::cout << "UserCtor: (members are indeterminate)\n";
}
```

**The rule:**

1. If `T` has no user-provided constructor and is not a union: **zero-initialize**, then default-initialize -> all members become zero.
2. If `T` has a user-provided constructor: **call that constructor** -> members are only initialized if the constructor initializes them.

**This is why `= default` and `{}` (empty body) are different:**

```cpp
struct A { int x; A() = default; };  // Not user-provided
struct B { int x; B() {}          };  // User-provided

A a{};  // x = 0 (zero-init happens first)
B b{};  // x = indeterminate (constructor body is empty, no zero-init)
```

The empty braces look identical at the call site, but the outcome is completely different depending on how the constructor was declared.

### Q3: Demonstrate UB from reading a default-initialized int variable

The code below is educational - it shows what operations are UB and what safe alternatives look like. The real-world bug pattern at the bottom is the kind of thing that passes code review without anyone noticing:

```cpp
#include <iostream>

// This function contains UB - for educational purposes only!
void demonstrate_ub() {
    int x;  // Default-initialized: indeterminate value

    // Reading x is UNDEFINED BEHAVIOR per [dcl.init]/12:
    // "If an indeterminate value is produced by an evaluation,
    // the behavior is undefined"

    // The compiler may:
    // 1. Print garbage (whatever was on the stack)
    // 2. Print 0 (if the stack happened to be zeroed)
    // 3. Optimize away the read entirely
    // 4. Cause a trap representation (on some architectures)
    // 5. Use the UB to "prove" this code is unreachable and delete it

    // ALL of these are UB:
    // std::cout << x;           // UB: reading uninitialized
    // int y = x;                // UB: reading uninitialized
    // if (x > 0) { ... }       // UB: reading uninitialized
    // return x;                 // UB: reading uninitialized

    // SAFE operations (no read):
    x = 42;                      // OK: writing to x
    std::cout << x << "\n";     // OK: x is now initialized
    int* p = &x;                 // OK: taking address doesn't read the value
}

// Real-world bug pattern:
struct Config {
    int timeout;
    bool verbose;
    int max_retries;
};

void use_config() {
    Config cfg;  // Default-init: all members indeterminate!
    // Programmer forgets to initialize max_retries:
    cfg.timeout = 30;
    cfg.verbose = true;
    // cfg.max_retries is still indeterminate

    // if (cfg.max_retries > 0) { ... }  // UB!

    // GOOD FIX: use value-initialization
    Config safe_cfg{};  // All members zeroed
    safe_cfg.timeout = 30;
    safe_cfg.verbose = true;
    // safe_cfg.max_retries is 0 - safe to read
    std::cout << "max_retries: " << safe_cfg.max_retries << "\n";  // 0
}

int main() {
    demonstrate_ub();
    use_config();
}
```

**How to catch this:**

| Tool | Flag |
| --- | --- |
| GCC/Clang | `-Wuninitialized`, `-Wsometimes-uninitialized` |
| MSVC | `/W4` (warning C4700) |
| Valgrind | Detects reads of uninitialized memory at runtime |
| ASan/MSan | `-fsanitize=memory` (Clang's MemorySanitizer) |
| Static analysis | clang-tidy `cppcoreguidelines-init-variables` |

---

## Notes

- **Always use `{}` initialization** for local variables when you want them zeroed. `int x{};` is safer than `int x = 0;` because it works generically in templates.
- `static` and global variables are always zero-initialized before any other initialization - you can rely on `static int count;` being 0.
- `std::array<int, N> arr{};` value-initializes all elements to zero. `std::array<int, N> arr;` leaves them indeterminate.
- In constructors, prefer member initializer lists over assignment in the body - the member initializer list is where initialization actually happens.
- C++ Core Guidelines: ES.20 - "Always initialize an object."
