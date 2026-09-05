# Understand the difference between value types and reference types

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP11.md>  

---

## Topic Overview

In C++, every object has a type that determines how it stores and accesses data. The two fundamental
categories are **value types** and **reference types**.

### Value Types

A value type stores its data directly. When you copy a value, you get a completely independent copy.

```cpp
int a = 10;
int b = a;   // b gets its own copy of the value 10
b = 20;      // a is still 10, b is now 20
```

This applies to all fundamental types (`int`, `double`, `char`, ...) and user-defined types (structs, classes).

```cpp
struct Point {
    double x, y;
};

Point p1{1.0, 2.0};
Point p2 = p1;    // p2 is an independent copy
p2.x = 99.0;      // p1.x is still 1.0
```

Because `p2` owns its own copy of the data, mutating it doesn't touch `p1` at all.

### Reference Types

A **reference** (`T&`) is an alias - another name for an existing object. It does NOT create a copy.

```cpp
int x = 42;
int& ref = x;  // ref is another name for x
ref = 100;     // x is now 100!
std::cout << x; // prints 100
```

A **pointer** (`T*`) stores the address of an object:

```cpp
int x = 42;
int* ptr = &x;  // ptr holds the address of x
*ptr = 100;     // x is now 100
```

Both forms let you reach an object without copying it, but references are always bound and can't be reseated, while pointers can be null and can point to different things over time.

### Dangling References - The Critical Danger

A reference becomes **dangling** when the object it refers to is destroyed. This is one of the most common and dangerous bugs in C++.

```cpp
int& make_dangling() {
    int local = 42;     // local lives on the stack
    return local;        // WARNING: returning reference to local!
}   // local is destroyed here

int& ref = make_dangling();
std::cout << ref;  // UNDEFINED BEHAVIOR — local no longer exists
```

The compiler usually warns about this with `-Wall`, but it's not always detectable. When it goes unnoticed, the program may seem to work until something else overwrites that stack memory.

### Pass-by-Value vs. Pass-by-Reference

Here's the practical consequence of the distinction. Three calling conventions, three different behaviors.

```cpp
// Pass by value — copies the argument
void by_value(std::string s) {
    s += " world";  // modifies the COPY, not the original
}

// Pass by reference — modifies the original
void by_ref(std::string& s) {
    s += " world";  // modifies the ORIGINAL
}

// Pass by const reference — no copy, no modification
void by_const_ref(const std::string& s) {
    std::cout << s;  // can read, cannot modify
    // s += " world"; // ERROR — s is const
}

std::string greeting = "hello";
by_value(greeting);     // greeting is still "hello" (copy was modified)
by_ref(greeting);       // greeting is now "hello world"
by_const_ref(greeting); // greeting unchanged, no copy made
```

`const T&` is often the best choice for read-only access to a large object: no copy, and the caller's data is safe.

### Performance Guidelines

| Type size | Recommended passing |
| --- | --- |
| 16 bytes or less (int, double, pointers) | By value |
| More than 16 bytes (string, vector, large structs) | By `const T&` |
| When modification needed | By `T&` |
| When ownership transfer needed | By value (+ move) |

The difference matters most for large types. Here's an extreme example to make the point concrete.

```cpp
// Benchmark example: Large struct
struct LargeData { std::array<double, 1000> values; };

void process_copy(LargeData data) { /* copies 8KB every call */ }
void process_ref(const LargeData& data) { /* zero-cost, just a pointer */ }
```

---

## Self-Assessment

### Q1: Can you explain why int& ref = x; does not copy x, and demonstrate a case where a dangling reference is created

When you write `int& ref = x;`, the reference `ref` becomes an alias for `x`. They share the
same memory location - no copy is made. Changing `ref` changes `x`, and vice versa.

```cpp
#include <iostream>

int main() {
    int x = 42;
    int& ref = x;  // ref is an alias for x — no copy

    std::cout << "x = " << x << ", ref = " << ref << "\n";   // x = 42, ref = 42
    std::cout << "&x = " << &x << ", &ref = " << &ref << "\n"; // same address!

    ref = 100;
    std::cout << "x = " << x << "\n";  // x = 100 — modified through ref
}
```

`&x` and `&ref` print the same address, which proves they are the same object with two names.

**Dangling reference example:**

```cpp
#include <iostream>

int& create_dangling() {
    int value = 42;
    return value;  // returns reference to local variable
}   // value is destroyed here — stack frame reclaimed

int main() {
    int& ref = create_dangling(); // ref refers to destroyed object
    // The stack memory may be overwritten by the next function call
    std::cout << ref;  // UNDEFINED BEHAVIOR
    // May print 42, garbage, or crash
}
```

Compile with `-Wall -Wextra -Werror` to catch this. GCC/Clang emit:
`warning: reference to local variable 'value' returned`

### Q2: Write a function that accidentally returns a reference to a local variable and explain the undefined behavior

The buggy and fixed versions do the same thing logically, but the return mechanism makes all the difference.

```cpp
#include <iostream>
#include <string>

// BAD: returns reference to a local variable
std::string& bad_greet(const std::string& name) {
    std::string result = "Hello, " + name + "!";
    return result;  // result is destroyed at function exit!
}

// GOOD: return by value (copy elision makes this efficient)
std::string good_greet(const std::string& name) {
    std::string result = "Hello, " + name + "!";
    return result;  // NRVO: no copy, constructed directly in caller's space
}

int main() {
    // Using the buggy version:
    std::string& ref = bad_greet("Alice");
    // ref now points to deallocated stack memory
    std::cout << ref << "\n"; // UNDEFINED BEHAVIOR:
    // - might print "Hello, Alice!" (memory not yet overwritten)
    // - might print garbage
    // - might crash (memory protection fault)
    // - might appear to work but corrupt other data

    // Using the fixed version:
    std::string val = good_greet("Alice");
    std::cout << val << "\n"; // always correct: "Hello, Alice!"
}
```

**Why is this UB?** The local `result` lives in the stack frame of `bad_greet()`. When the function
returns, its stack frame is reclaimed. The reference still points to that memory, but the memory
can now be reused by any subsequent function call.

### Q3: Show how pass-by-value, pass-by-reference, and pass-by-const-ref differ in a benchmark context

The numbers will vary by machine, but the relative difference is consistent: copying 40 KB ten thousand times adds up fast.

```cpp
#include <iostream>
#include <string>
#include <chrono>
#include <array>

struct Large {
    std::array<int, 10000> data{};  // ~40KB
};

// Pass by value — copies 40KB every call
void by_value(Large l) {
    l.data[0] = 999;  // modifies the copy only
}

// Pass by reference — can modify the original
void by_ref(Large& l) {
    l.data[0] = 999;  // modifies the original!
}

// Pass by const reference — no copy, no modification (best for reading)
void by_const_ref(const Large& l) {
    int x = l.data[0];  // read OK
    // l.data[0] = 999;  // ERROR: l is const
}

int main() {
    Large obj;

    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 10000; ++i) by_value(obj);     // copies 400MB total!
    auto mid = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 10000; ++i) by_const_ref(obj);  // zero copies
    auto end = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::milliseconds;
    std::cout << "by_value:     " << std::chrono::duration_cast<ms>(mid - start).count() << "ms\n";
    std::cout << "by_const_ref: " << std::chrono::duration_cast<ms>(end - mid).count() << "ms\n";
    // by_const_ref will be dramatically faster due to zero copies and better cache behavior
}
```

**Guidelines:**

- Small types (`int`, `double`, `char`, pointers, iterators): pass by value.
- Large types (`std::string`, `std::vector`, custom structs > 16 bytes): pass by `const T&`.
- When you need to modify the original: pass by `T&`.
- When transferring ownership: pass by value and let the caller `std::move()`.

---

## Notes

- C++ does NOT have "reference types" like Java/C# - a `T&` is always an alias, never an object itself.
- References must be initialized when declared and cannot be reseated (reassigned to another object).
- `std::reference_wrapper<T>` provides a copyable, assignable reference wrapper for use in containers.
- Move semantics (C++11) allow transferring resources instead of copying: `std::string s2 = std::move(s1);`
- Slicing occurs when a derived object is assigned by value to a base variable: `Base b = derived;` - the derived portion is lost.
- Use `const T&` parameters by default for types larger than a pointer; use value for small/trivially copyable types.
- Smart pointers (`unique_ptr`, `shared_ptr`) express ownership - prefer them over raw pointers for heap objects.
