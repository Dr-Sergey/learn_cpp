# Use nullptr instead of NULL or 0 for null pointers

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP11.md#nullptr>  

---

## Topic Overview

`nullptr` is a keyword that C++11 added to mean "a null pointer." The thing that makes it special is its type: it has the distinct type `std::nullptr_t`, which is deliberately *not* an integer type. That one design choice is what fixes all the old headaches.

### The Problem with NULL and 0

Before C++11, you wrote null pointers as either the literal `0` or the macro `NULL`, and that macro was really just `0` in disguise:

```cpp
// In <cstddef> or <stddef.h>:
#define NULL 0    // or #define NULL 0L
```

So when you wrote `NULL`, the compiler genuinely saw an **integer**, not a pointer. Most of the time you got away with it, but the moment overloads enter the picture it bites:

```cpp
void process(int value) {
    std::cout << "Called with int: " << value << "\n";
}
void process(int* ptr) {
    std::cout << "Called with pointer: " << ptr << "\n";
}

process(0);       // Calls process(int)! Not the pointer version.
process(NULL);    // AMBIGUOUS! NULL is 0 (int), both overloads match.
                  // Some compilers pick int, some report error.
process(nullptr); // Calls process(int*) - unambiguous!
```

The lesson is that `0` and `NULL` *look* like they mean "pointer" to a human reader, but to the compiler they say "integer that happens to be allowed to become a pointer." `nullptr` removes that gap between what you mean and what the compiler sees.

### nullptr Type: std::nullptr_t

`nullptr` carries its own unique type. It will happily turn into any pointer type you ask for, but it refuses to become an integer:

```cpp
#include <type_traits>

// nullptr has its own unique type:
static_assert(std::is_same_v<decltype(nullptr), std::nullptr_t>);

// nullptr_t converts to ANY pointer type:
int* p1 = nullptr;         // OK
double* p2 = nullptr;      // OK
std::string* p3 = nullptr; // OK
void* p4 = nullptr;        // OK

// But NOT to integer types:
// int x = nullptr;        // ERROR: cannot convert nullptr to int
// bool b = nullptr;       // works only as a contextual conversion to bool
```

That asymmetry - "yes to pointers, no to ints" - is exactly the property `0` lacked.

### In Templates

Templates are where the difference really shows up, because the type gets *deduced* and then locked in. `nullptr` deduces as `std::nullptr_t`, while `NULL` quietly deduces as `int`:

```cpp
template<typename T>
void inspect(T value) {
    if constexpr (std::is_null_pointer_v<T>) {
        std::cout << "null pointer\n";
    } else if constexpr (std::is_integral_v<T>) {
        std::cout << "integer: " << value << "\n";
    } else if constexpr (std::is_pointer_v<T>) {
        std::cout << "pointer\n";
    }
}

inspect(NULL);    // "integer: 0" - T = int (or long)
inspect(nullptr); // "null pointer" - T = std::nullptr_t
inspect(0);       // "integer: 0" - T = int
```

Notice that `NULL` and `0` end up in the *integer* branch. If your template was written assuming it received a pointer, that is where things start to go wrong.

### Forwarding Through Templates

Here is the concrete failure. A template grabs whatever you give it and forwards it to something expecting a pointer:

```cpp
template<typename T>
void forward_to_api(T arg) {
    legacy_c_function(arg);  // expects a pointer
}

forward_to_api(NULL);    // T = int -> ERROR: int is not a pointer!
forward_to_api(nullptr); // T = nullptr_t -> converts to pointer automatically
```

With `NULL`, `T` becomes `int`, and an `int` will not convert to a pointer, so the call breaks. With `nullptr`, `T` is `std::nullptr_t`, which converts cleanly. Same-looking code, completely different outcome.

### Using nullptr in Conditions

For checking whether a pointer is null, you have a couple of equivalent styles. Pick whichever reads more clearly to you:

```cpp
int* ptr = get_optional_result();

// All of these are equivalent:
if (ptr != nullptr) { ... }
if (ptr) { ... }            // implicit conversion to bool

// For smart pointers too:
std::unique_ptr<Widget> up = make_widget();
if (up) { ... }              // checks if not null
if (up != nullptr) { ... }   // same thing, more explicit
```

---

## Self-Assessment

### Q1: Write code where passing NULL to an overloaded function taking int vs pointer causes an ambiguity that nullptr resolves

The whole drama here happens at overload resolution. Watch what each of the three "null-ish" arguments does:

```cpp
#include <iostream>

// Two overloads: one for int, one for pointer
void process(int value) {
    std::cout << "int overload: " << value << "\n";
}
void process(double* ptr) {
    std::cout << "pointer overload: " << ptr << "\n";
}

int main() {
    // process(NULL);    // Ambiguous: NULL is 0 (int literal)
    //                   // Both process(int) and process(double*) match
    //                   // Compiler error: "ambiguous overload"

    process(nullptr);    // "pointer overload: 0"
    //                   // nullptr is std::nullptr_t -> converts to double*
    //                   // Cannot convert to int -> only one match

    process(0);          // "int overload: 0"
    //                   // 0 is an int - matches process(int) directly
}
```

The reason `NULL` is ambiguous is that `0` is *both* a perfectly good `int` and a legal null-pointer constant for `double*`. The compiler sees two equally-good options and refuses to guess. `nullptr` breaks the tie because `std::nullptr_t` converts to pointers but never to `int` - so only the pointer overload survives.

### Q2: Show how nullptr has type std::nullptr_t and can be used in templates

You can even write a function that accepts *only* `nullptr` and nothing else, which is a nice demonstration that `std::nullptr_t` is a real, distinct type:

```cpp
#include <iostream>
#include <type_traits>
#include <cstddef>

// nullptr has type std::nullptr_t
static_assert(std::is_same_v<decltype(nullptr), std::nullptr_t>);
static_assert(std::is_null_pointer_v<decltype(nullptr)>);

// std::nullptr_t can be used in templates:
template<typename T>
std::string type_name(T) {
    if constexpr (std::is_null_pointer_v<T>)
        return "nullptr_t";
    else if constexpr (std::is_pointer_v<T>)
        return "pointer";
    else if constexpr (std::is_integral_v<T>)
        return "integer";
    else
        return "other";
}

int main() {
    std::cout << type_name(nullptr) << "\n";  // "nullptr_t"
    std::cout << type_name(0) << "\n";         // "integer"
    std::cout << type_name((int*)0) << "\n";   // "pointer"

    // You can have a function that specifically accepts nullptr:
    auto only_null = [](std::nullptr_t) {
        std::cout << "received nullptr\n";
    };
    only_null(nullptr);  // OK
    // only_null(0);      // ERROR: int is not nullptr_t
}
```

### Q3: Demonstrate a compile error that NULL produces but nullptr does not when deducing template type

This pulls the previous ideas together: a template that only makes sense if `T` is genuinely a pointer. Feed it `NULL` and `T` becomes an integer, and the pointer-only operations inside stop compiling:

```cpp
#include <iostream>
#include <cstddef>

// Template that expects to receive a pointer:
template<typename T>
void call_with_ptr(T ptr) {
    // Try to dereference - only works if T is actually a pointer
    if (ptr != nullptr) {  // ERROR if T = int (from NULL)
        std::cout << *ptr << "\n";
    }
}

template<typename T>
void pass_to_api(T arg) {
    int* destination = arg;  // only works if T converts to int*
    // If T = int (from NULL), this is int* = int -> ERROR
}

int main() {
    int value = 42;

    // With NULL:
    // call_with_ptr(NULL);
    // T deduces to int (or long), then:
    //   - "ptr != nullptr" -> comparing int with nullptr_t -> ERROR
    //   - "*ptr" -> dereferencing int -> ERROR

    // pass_to_api(NULL);
    // T = int -> "int* destination = int_value" -> ERROR

    // With nullptr - everything works:
    call_with_ptr(&value);   // T = int*, works perfectly
    call_with_ptr(nullptr);  // T = std::nullptr_t, nullptr != nullptr is false, OK
    pass_to_api(&value);     // T = int*, works perfectly
    pass_to_api(nullptr);    // T = std::nullptr_t, converts to int*, OK
}
```

The single root cause behind every one of these errors: `NULL` makes `T` deduce as an integer type, and integers do not behave like pointers. `nullptr` deduces as `std::nullptr_t`, which slots into any pointer context you throw at it.

---

## Notes

- `nullptr` is a keyword, not a macro - it's always available without any `#include`.
- `std::nullptr_t` is implicitly convertible to any pointer type and any pointer-to-member type.
- In template code, `NULL` deduces as `int` (or `long`) while `nullptr` deduces as `std::nullptr_t` - this is the most common source of template errors with `NULL`.
- `nullptr` can be contextually converted to `bool` (evaluates to `false`), but cannot be implicitly converted to integer types.
- C++ Core Guidelines: ES.47 - "Use nullptr rather than 0 or NULL."
- Some legacy C APIs accept `0` for null - you can still pass `nullptr` to them since it converts to any pointer type.
