# Understand member initializer lists and their order of initialization

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/constructor>  

---

## Topic Overview

A **member initializer list** appears after a constructor's parameter list, prefixed by a colon. It specifies how each data member and base class is initialized before the constructor body executes.

### Why Use Initializer Lists

Here's the thing - using an initializer list and doing assignment in the body look similar, but they are not the same operation:

| Initialization Method | Behavior |
| --- | --- |
| Member initializer list | Direct-initializes each member - one construction |
| Assignment in constructor body | Default-constructs, then assigns - two operations |

For `const` members, references, and types without default constructors, the initializer list is **mandatory**.

### Initialization Order Rules

**The order of initialization is always:**

1. Virtual base classes (depth-first, left-to-right)
2. Direct base classes (left-to-right in the inheritance list)
3. Non-static data members (**in declaration order**, NOT initializer list order)
4. Constructor body executes

This is the rule that trips people up. No matter what order you write the initializer list, the compiler initializes members in the order they appear in the class definition. The example below shows exactly how that goes wrong:

```cpp
class Widget {
    int a;     // initialized 1st
    int b;     // initialized 2nd
    int c;     // initialized 3rd
public:
    // WARNING: initializer list order does NOT match declaration order!
    Widget(int x) : c(x), b(c), a(b) {}  // BUG!
    // Actual initialization order: a(b), b(c), c(x)
    // a uses uninitialized b, b uses uninitialized c
};
```

The fix is simple: write your initializer list in the same order as your member declarations.

### Destruction Order

Members are destroyed in **reverse declaration order** - the exact mirror of construction. This guarantees that if member B was initialized after member A, B is destroyed before A.

---

## Self-Assessment

### Q1: Show a bug where member initializer list order differs from member declaration order

Pay close attention to which member is declared first. The initializer list in `BadOrder` looks like it initializes `capacity` first, but it doesn't.

```cpp
#include <iostream>

class BadOrder {
    int length;   // declared 1st -> initialized 1st
    int capacity; // declared 2nd -> initialized 2nd
public:
    // Initializer list says: capacity first, then length
    // But actual init order is: length first (uses capacity which is garbage!)
    BadOrder(int cap)
        : capacity(cap),          // LOOKS like this runs first, but it doesn't!
          length(capacity / 2)    // LOOKS second, but actually runs FIRST
    {
        std::cout << "length   = " << length << "\n";    // garbage value!
        std::cout << "capacity = " << capacity << "\n";  // correct: cap
    }
};

class GoodOrder {
    int length;
    int capacity;
public:
    // Match initializer list order to declaration order
    GoodOrder(int cap)
        : length(cap / 2),    // 1st - matches declaration order
          capacity(cap)        // 2nd - matches declaration order
    {
        std::cout << "length   = " << length << "\n";    // cap/2
        std::cout << "capacity = " << capacity << "\n";  // cap
    }
};

int main() {
    std::cout << "--- BadOrder ---\n";
    BadOrder bad(100);    // length = garbage, capacity = 100

    std::cout << "--- GoodOrder ---\n";
    GoodOrder good(100);  // length = 50, capacity = 100
}
```

**How this works:**

- In `BadOrder`, `length` is declared before `capacity`, so `length` is initialized first.
- The initializer list says `length(capacity / 2)` - but `capacity` hasn't been initialized yet at that point.
- `length` gets an **uninitialized garbage value**, causing undefined behavior.
- The fix: always write the initializer list in the same order as the member declarations.

### Q2: Explain that C++ initializes members in declaration order, not initializer list order

**Answer:**

The C++ standard (§[class.base.init]/13) specifies:

- Non-static data members are initialized in the order they are **declared in the class definition**.
- The order in the member initializer list is **irrelevant** to the actual initialization sequence.

**Rationale:** Destructors must run in a single, deterministic reverse order. If initialization order depended on which constructor was called, different constructors could produce different destruction orders - making cleanup unpredictable.

Notice below that both constructors produce the same initialization order regardless of how the initializer list is written:

```cpp
class Example {
    std::string name;   // 1st declared -> 1st initialized, last destroyed
    std::vector<int> v; // 2nd declared -> 2nd initialized, destroyed before name
    int count;          // 3rd declared -> 3rd initialized, destroyed first
public:
    // These all initialize in the SAME order: name, v, count
    Example() : count(0), v{}, name("default") {}   // list order ignored
    Example(int n) : name("custom"), count(n), v(n) {} // list order ignored
};
```

| What determines order? | Answer |
| --- | --- |
| Declaration order in class | Yes - Always |
| Initializer list order | No - Never |
| Constructor parameter order | No |
| Base class order | Left-to-right in inheritance list |

### Q3: Add a compiler warning for out-of-order initializers (-Wreorder in GCC/Clang)

The good news is you don't have to catch this by eye. Enable `-Wreorder` (part of `-Wall`) and the compiler will flag it for you.

```cpp
// Compile with: g++ -std=c++20 -Wall -Wreorder init_order.cpp

#include <iostream>
#include <string>

class Config {
    std::string name;   // declared 1st
    int priority;       // declared 2nd
    bool active;        // declared 3rd
public:
    // Intentionally wrong order to trigger -Wreorder warning
    Config(const std::string& n, int p)
        : priority(p),       // WARNING: 'priority' will be initialized after 'name'
          name(n),           // WARNING: 'name' will be initialized before 'priority'
          active(true)       // OK: declared last, listed last
    {
        std::cout << name << " prio=" << priority << " active=" << active << "\n";
    }
};

int main() {
    Config c("server", 5);
}

// GCC/Clang output with -Wreorder:
// warning: 'Config::priority' will be initialized after 'Config::name'
//          [-Wreorder]
//
// MSVC equivalent: /W4 enables warning C5038
// warning C5038: data member 'Config::priority' will be initialized
//                after data member 'Config::name'
```

**How this works:**

- `-Wreorder` (included in `-Wall`) warns when the initializer list order doesn't match declaration order.
- MSVC uses warning C5038 (enabled by `/W4`).
- **Best practice:** Always write the initializer list in declaration order, and compile with warnings enabled. Many teams treat `-Werror` as policy to catch this.

---

## Notes

- Member initializer lists are more efficient than assignment in the body - they avoid default-construction + copy-assignment for non-trivial types.
- `const` members, reference members, and members of types with no default constructor **must** be initialized in the initializer list.
- Default member initializers (C++11 `int x = 0;` in-class) are overridden by the initializer list - they act as a fallback when a member isn't mentioned in the list.
- Virtual base classes are always initialized by the **most derived class**, regardless of where they appear in intermediate constructors.
- Use `-Wreorder` (GCC/Clang) or `/W4` (MSVC) to catch mismatched initializer list order at compile time.
