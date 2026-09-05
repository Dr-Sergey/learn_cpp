# Write stateful lambdas and understand their mutable semantics

**Category:** Lambda and Functional  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

Lambdas that capture variables by value carry **state** inside the closure object - the captured value becomes a private data member of the generated struct. By default, the compiler marks `operator()` as `const`, which means you can read your captured data but you cannot modify it. The `mutable` keyword lifts that restriction, making `operator()` non-const so the lambda can update its internal state between calls.

Here is the mental model that makes this click: a capturing lambda is just a struct with data members and a call operator. "Mutable or not" is simply whether that call operator is `const` or not.

```cpp
int count = 0;
auto counter = [count]() mutable { return ++count; };
//                                  ^^^^^^^
// Without mutable: ERROR - can't modify captured 'count'
// With mutable:    OK - count is a non-const member of the closure

counter();  // returns 1
counter();  // returns 2
counter();  // returns 3
// Original 'count' is still 0!
```

The original `count` variable is untouched because the lambda captured a **copy**. The `mutable` keyword only allows the lambda to modify its own internal copy - it has nothing to do with the original.

---

## Self-Assessment

### Q1: Create a counter lambda using `mutable` and show that each copy has its own state

The subtle point here is what happens when you copy a lambda. Because the captured value is a data member of the closure struct, copying the lambda copies the entire struct - including the current value of that data member. Each copy is an independent object with its own state from that point forward.

```cpp
#include <iostream>

int main() {
    // Stateful lambda: captures 'n' by value, mutable allows modification
    auto counter = [n = 0]() mutable { return ++n; };

    std::cout << "counter: " << counter() << "\n";  // 1
    std::cout << "counter: " << counter() << "\n";  // 2
    std::cout << "counter: " << counter() << "\n";  // 3

    // COPY the lambda - each copy gets its OWN state!
    auto counter2 = counter;  // copies current state (n=3)

    std::cout << "counter:  " << counter() << "\n";   // 4
    std::cout << "counter2: " << counter2() << "\n";  // 4 (starts from 3)
    std::cout << "counter:  " << counter() << "\n";   // 5
    std::cout << "counter2: " << counter2() << "\n";  // 5

    // Each is independent:
    std::cout << "\nIndependent states:\n";
    auto a = [n = 0]() mutable { return ++n; };
    auto b = a;  // copies when n=0
    auto c = a;  // copies when n=0

    a(); a(); a();  // a: n=3
    b(); b();       // b: n=2
    c();            // c: n=1

    std::cout << "a: " << a() << "\n";  // 4
    std::cout << "b: " << b() << "\n";  // 3
    std::cout << "c: " << c() << "\n";  // 2
}
// Expected output:
//   counter: 1
//   counter: 2
//   counter: 3
//   counter:  4
//   counter2: 4
//   counter:  5
//   counter2: 5
//
//   Independent states:
//   a: 4
//   b: 3
//   c: 2
```

The independent-state behavior is why stateful lambdas work well when you need a private counter or accumulator that does not share state with any other code. It is also where bugs can hide: if you pass a stateful lambda to a standard algorithm that internally copies it, the copy advances its own counter and the original stays behind.

---

### Q2: Explain why the call operator of a non-mutable lambda is `const` by default

This question gets at the design intent. When you capture by value, you are saying "give me my own copy of this data." The designers of the language then made a conservative choice: unless you explicitly ask for mutation (`mutable`), the call operator is `const`. This prevents a class of bugs where you accidentally modify a captured value and expect to see that change reflected elsewhere.

```cpp
#include <iostream>
#include <string>

int main() {
    std::string greeting = "Hello";

    // Lambda capturing by value WITHOUT mutable:
    auto say = [greeting]() {
        // greeting += " World";  // ERROR: operator() is const!
        std::cout << greeting << "\n";
    };
    say();

    // Lambda WITH mutable:
    auto say_mut = [greeting]() mutable {
        greeting += " World";  // OK: operator() is non-const
        std::cout << greeting << "\n";
    };
    say_mut();  // "Hello World"
    say_mut();  // "Hello World World"

    // Original is UNCHANGED:
    std::cout << "original: " << greeting << "\n";
}
// Expected output:
//   Hello
//   Hello World
//   Hello World World
//   original: Hello
```

**Why `const` by default?**

1. **Safety:** Prevents accidental mutation of captured state. Most lambdas just read.
2. **Consistency:** Calling a lambda shouldn't have visible side effects by default.
3. **Thread safety:** `const` lambdas can be safely shared across threads.
4. **Algorithm compatibility:** STL algorithms may copy predicates - const ensures copies behave identically.

To make the compiler-generated struct explicit, here is what the compiler actually produces for each case:

```cpp
Closure type generated by compiler:

struct __lambda {
    std::string greeting;  // captured by value

    // Without mutable:
    void operator()() const {   // <- CONST!
        // greeting += "x";    // ERROR: 'this' is const
    }

    // With mutable:
    void operator()() {         // <- non-const
        greeting += "x";       // OK
    }
};
```

Once you think of the lambda as a struct, the `const` rule is obvious: it is exactly the same rule that applies to any `const` member function - you cannot modify data members from within it.

---

### Q3: Compare a mutable lambda with a functor struct to understand what the compiler generates

This is the most direct way to demystify lambdas: write out the struct that the compiler would generate and confirm that the behavior is identical. There should be no surprises here - a lambda is syntax sugar for exactly this struct.

```cpp
#include <iostream>

// The functor struct (what the compiler generates)
struct CounterFunctor {
    int n;

    // mutable lambda: operator() is NOT const
    int operator()() {
        return ++n;
    }
};

// Non-mutable equivalent
struct ReadOnlyFunctor {
    int n;

    // non-mutable lambda: operator() IS const
    int operator()() const {
        return n;  // can only read
        // return ++n;  // ERROR: n is not mutable in const context
    }
};

int main() {
    // Lambda version:
    auto counter_lambda = [n = 0]() mutable { return ++n; };

    // Struct version (identical behavior):
    CounterFunctor counter_struct{0};

    // Both behave the same:
    std::cout << "Lambda: " << counter_lambda() << ", "
              << counter_lambda() << ", " << counter_lambda() << "\n";
    std::cout << "Struct: " << counter_struct() << ", "
              << counter_struct() << ", " << counter_struct() << "\n";

    // Sizeof is the same:
    std::cout << "\nsizeof(lambda): " << sizeof(counter_lambda) << "\n";
    std::cout << "sizeof(struct): " << sizeof(counter_struct) << "\n";

    // Copy semantics are the same:
    auto lambda_copy = counter_lambda;  // copies n=3
    CounterFunctor struct_copy = counter_struct;  // copies n=3

    std::cout << "\nAfter copy (both at n=3):\n";
    std::cout << "Lambda original: " << counter_lambda() << "\n";  // 4
    std::cout << "Lambda copy:     " << lambda_copy() << "\n";     // 4
    std::cout << "Struct original: " << counter_struct() << "\n";  // 4
    std::cout << "Struct copy:     " << struct_copy() << "\n";     // 4

    // Multi-capture example:
    int x = 10;
    double y = 3.14;
    auto multi = [x, y, z = std::string{"hello"}]() mutable {
        ++x;
        return x;
    };
    // Equivalent struct:
    // struct __lambda {
    //     int x;             // 4 bytes
    //     double y;          // 8 bytes
    //     std::string z;     // 32 bytes (typically)
    //     int operator()() { return ++x; }
    // };
    std::cout << "\nsizeof(multi-capture): " << sizeof(multi) << "\n";
}
// Expected output:
//   Lambda: 1, 2, 3
//   Struct: 1, 2, 3
//
//   sizeof(lambda): 4
//   sizeof(struct): 4
//
//   After copy (both at n=3):
//   Lambda original: 4
//   Lambda copy:     4
//   Struct original: 4
//   Struct copy:     4
//
//   sizeof(multi-capture): 48   (or similar, depends on alignment)
```

The `sizeof` comparison is especially illuminating. `sizeof(counter_lambda)` equals `sizeof(CounterFunctor)` - 4 bytes for one `int`. The multi-capture lambda's size is the sum of its captured types plus padding, just as if you had laid out a struct manually.

---

## Notes

- **`mutable` is per-lambda**, not per-capture. All captures become modifiable when `mutable` is used.
- **Capture by reference** doesn't need `mutable` - the reference itself is const (can't rebind), but the referred-to object is mutable.
- **STL gotcha:** `std::for_each` returns the function object, preserving state. But `std::count_if` etc. may copy the predicate and discard state.
- **Init-capture with `mutable`:** `[n = 0]() mutable {}` creates a unique-to-lambda variable - like a private member.
- **`std::ref` wrapper:** To share state between copies, capture a `std::reference_wrapper` instead of a plain value.
