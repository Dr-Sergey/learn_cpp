# Understand how to read complex C++ type declarations

**Category:** Core Language Fundamentals  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/declarations>  

---

## Topic Overview

C++ inherited its declaration syntax from C, which can lead to notoriously hard-to-read declarations. The **clockwise/spiral rule** and **type aliases** are your tools for decoding and simplifying them.

### The Clockwise/Spiral Rule

Starting from the identifier, read the declaration by spiraling clockwise:

1. Start at the variable name.
2. Move **right** - read `()` (function) or `[]` (array).
3. Move **left** - read `*` (pointer) or type qualifiers.
4. Repeat until the entire declaration is consumed.

### Simple Examples

Work through these one at a time and you'll see the pattern click into place:

```cpp
int *p;              // p is a pointer to int
int *p[10];          // p is an array of 10 pointers to int
int (*p)[10];        // p is a pointer to an array of 10 ints
int (*fp)(int);      // fp is a pointer to a function taking int, returning int
int *(*fp)(int);     // fp is a pointer to a function taking int, returning pointer to int
```

### The `signal` Declaration

The classic example that stumps most readers - but the spiral rule handles it mechanically:

```cpp
// The most famous complex C declaration:
void (*signal(int sig, void (*handler)(int)))(int);
```

Reading with the spiral rule:

1. `signal` - identifier
2. `(int sig, void (*handler)(int))` - signal is a **function** taking `int` and a pointer-to-function
3. `*` - which returns a **pointer**
4. `(int)` - to a **function** taking `int`
5. `void` - returning `void`

In English: "`signal` is a function that takes an `int` and a pointer to a function `(int) -> void`, and returns a pointer to a function `(int) -> void`."

### Simplification with Type Aliases

Once you spot the repeated type, a `using` alias makes the whole declaration transparent:

```cpp
// Step 1: Define the function pointer type
using SignalHandler = void (*)(int);    // pointer to function (int) -> void

// Step 2: Rewrite signal with the alias
SignalHandler signal(int sig, SignalHandler handler);
// Now it's crystal clear!
```

### Modern C++ Alternatives

Modern C++ gives you several ways to avoid writing the raw complex form in the first place:

```cpp
// Using std::function (runtime overhead but readable)
#include <functional>
std::function<void(int)> handler;

// Using trailing return type
auto signal(int sig, void (*handler)(int)) -> void (*)(int);

// Using type alias + trailing return
using Handler = void(*)(int);
auto signal(int sig, Handler handler) -> Handler;
```

### Common Patterns to Recognize

If the table feels like a lot, focus on the const-placement rows first - those trip up the most people:

| Declaration | Meaning |
| --- | --- |
| `int *a[5]` | array of 5 pointers to int |
| `int (*a)[5]` | pointer to array of 5 ints |
| `int (*f)()` | pointer to function returning int |
| `int *(*f)()` | pointer to function returning pointer to int |
| `int (*f[5])()` | array of 5 pointers to functions returning int |
| `const int *p` | pointer to const int |
| `int *const p` | const pointer to int |
| `const int *const p` | const pointer to const int |

---

## Self-Assessment

### Q1: Parse `void (*signal(int, void (*fp)(int)))(int);` using the clockwise/spiral rule

**Answer:**

Step-by-step parsing:

```cpp
void (*signal(int, void (*fp)(int)))(int);
       ^^^^^^
       Start here: "signal"

void (*signal(int, void (*fp)(int)))(int);
             ^^^^^^^^^^^^^^^^^^^^^^^^^
       Move right: signal is a FUNCTION taking:

         - int
         - void (*fp)(int)  [fp is a pointer to function(int)->void]

void (*signal(int, void (*fp)(int)))(int);
      ^
       Move left: signal returns a POINTER...

void (*signal(int, void (*fp)(int)))(int);
                                    ^^^^^
       Move right from *: ...to a FUNCTION taking int...

void (*signal(int, void (*fp)(int)))(int);
^^^^
       Move left: ...returning void
```

**Result:** `signal` is a function that takes `(int, void(*)(int))` and returns `void(*)(int)`.

In plain English: "signal takes a signal number and a handler function pointer, and returns the previous handler function pointer."

### Q2: Use a type alias to simplify the above declaration to a readable form

With the alias in place you can read the function signature at a glance - no spiral required:

```cpp
#include <iostream>

// Original (unreadable):
// void (*signal(int, void (*fp)(int)))(int);

// Step 1: Identify the repeated type -- "pointer to function taking int, returning void"
using SignalHandler = void (*)(int);

// Step 2: Rewrite
SignalHandler my_signal(int sig, SignalHandler handler);

// Step 3: Implement
void my_handler(int sig) {
    std::cout << "Caught signal " << sig << "\n";
}

SignalHandler my_signal(int sig, SignalHandler handler) {
    std::cout << "Registering handler for signal " << sig << "\n";
    static SignalHandler previous = nullptr;
    auto old = previous;
    previous = handler;
    return old;   // return the previous handler
}

int main() {
    auto prev = my_signal(2, my_handler);
    std::cout << "Previous handler was " << (prev ? "set" : "null") << "\n";

    // Call the registered handler
    my_handler(2);   // "Caught signal 2"
}
```

**How it works:**

- `using SignalHandler = void (*)(int);` creates a readable alias for the function pointer type.
- The function signature becomes `SignalHandler my_signal(int, SignalHandler)` - instantly comprehensible.
- Modern C++ strongly prefers `using` over `typedef` for type aliases.

### Q3: Show how cdecl.org or a type printer tool can decode complex declarations

When you're stuck on a declaration, you can ask the compiler to name the type for you:

```cpp
#include <iostream>
#include <typeinfo>

#ifdef __GNUG__
#include <cxxabi.h>
#include <cstdlib>
#include <memory>

std::string demangle(const char* name) {
    int status;
    std::unique_ptr<char, void(*)(void*)> res(
        abi::__cxa_demangle(name, nullptr, nullptr, &status),
        std::free
    );
    return status == 0 ? res.get() : name;
}
#else
std::string demangle(const char* name) { return name; }
#endif

template<typename T>
void print_type() {
    std::cout << demangle(typeid(T).name()) << "\n";
}

int main() {
    // Decode complex types at compile time
    print_type<void (*)(int)>();
    // Output: void (*)(int)

    print_type<int (*(*)(int, int))[5]>();
    // Output: int (*(*(int, int)))[5]
    // "pointer to function(int,int) returning pointer to array of 5 ints"

    print_type<void (*(*[3])())(int)>();
    // Array of 3 pointers to functions returning pointer to function(int)->void

    // Online tool: cdecl.org
    // Input:  "declare signal as function (int, pointer to function (int) returning void)
    //          returning pointer to function (int) returning void"
    // Output: void (*signal(int, void (*)(int)))(int)

    // Tip: In practice, always use type aliases instead of writing complex declarations:
    using FuncPtr = void(*)(int);
    using ArrayOf5 = int(*)[5];
    // These are self-documenting and much harder to get wrong.
}
```

**How it works:**

- `__cxa_demangle` (GCC/Clang) converts mangled type names to human-readable form.
- cdecl.org accepts English descriptions and produces C/C++ declarations (and vice versa).
- The best practice: never write complex declarations - use `using` aliases to break them into readable pieces.

---

## Notes

- The `using` alias is strictly better than `typedef` for readability: `using F = int(*)(double);` vs `typedef int(*F)(double);`.
- `decltype` can capture complex types without writing them: `decltype(&signal) p = &signal;`.
- C++11 trailing return types help: `auto f() -> int(*)[5]` is more readable than `int (*f())[5]`.
- When reading legacy C code, cdecl.org is invaluable - bookmark it.
