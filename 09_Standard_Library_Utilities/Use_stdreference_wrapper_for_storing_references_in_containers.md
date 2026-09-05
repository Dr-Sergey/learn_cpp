# Use std::reference_wrapper for storing references in containers

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/reference_wrapper>  

---

## Topic Overview

`std::reference_wrapper<T>` (header `<functional>`) is a copyable, assignable wrapper around a reference to `T`. It solves a fundamental problem: C++ references (`T&`) are not objects - they can't be stored in containers, can't be rebound, and can't be copied. `reference_wrapper` is just a thin pointer-sized object that acts like a reference but satisfies all the requirements containers need.

### The Problem

You can't directly put references into a `std::vector`. The compiler says no because references aren't objects:

```cpp
std::vector<int&> refs;  // ERROR: references are not objects
// error: forming pointer to reference type 'int&'

// std::vector requires elements to be:
// - CopyAssignable or MoveAssignable
// - Destructible
// References satisfy none of these — they're aliases, not objects.
```

### The Solution

Wrapping the references in `std::reference_wrapper` turns them into proper objects that containers are happy to store:

```cpp
#include <functional>
#include <vector>

int a = 10, b = 20;
std::vector<std::reference_wrapper<int>> refs = {a, b};
refs[0].get() = 42;  // modifies 'a'
// a is now 42
```

The `.get()` call is how you access the underlying reference explicitly. In most other contexts - passing to a function, comparisons - the implicit conversion to `T&` kicks in automatically.

### Key Properties

| Property | `T&` (reference) | `std::reference_wrapper<T>` |
| --- | --- | --- |
| Is an object | No (alias) | Yes |
| Stored in containers | No | Yes |
| Rebindable | No (bound at init) | Yes (assignment rebinds) |
| Copyable | No | Yes |
| Implicit conversion to `T&` | - | Yes |
| sizeof | N/A | `sizeof(T*)` (pointer-sized) |
| Created via | `T& r = obj;` | `std::ref(obj)` or `std::cref(obj)` |

### Core Usage

Here you can see the wrapper in practice: sort the references by value without touching the original objects, then modify through the wrapper and see the original change.

```cpp
#include <functional>
#include <vector>
#include <algorithm>
#include <iostream>

int main() {
    int a = 3, b = 1, c = 2;

    // Store references in a vector
    std::vector<std::reference_wrapper<int>> refs = {a, b, c};

    // Sort the references (sorts by referenced values)
    std::sort(refs.begin(), refs.end()); // uses implicit conversion to int&

    // The original variables are NOT moved — refs just reorder
    for (auto& r : refs)
        std::cout << r.get() << " "; // 1 2 3
    std::cout << "\n";

    // Modify through the wrapper — changes the original
    refs[0].get() = 100;
    std::cout << "b = " << b << "\n"; // b = 100 (b was the smallest)

    // std::ref and std::cref convenience functions
    auto r1 = std::ref(a);   // reference_wrapper<int>
    auto r2 = std::cref(a);  // reference_wrapper<const int>
    // r2.get() = 5;  // ERROR: const reference
}
```

### With std::bind and std::thread

The reason `std::ref` exists goes beyond containers. `std::bind` and `std::thread` copy their arguments by default - which can silently swallow modifications you expected to see. Wrapping in `std::ref` tells them to store a reference instead:

```cpp
#include <functional>
#include <iostream>

void increment(int& x) { ++x; }

int main() {
    int value = 10;

    // WITHOUT std::ref — bind COPIES the argument
    auto f1 = std::bind(increment, value);
    f1();
    std::cout << value << "\n"; // 10 — unchanged! bind copied it

    // WITH std::ref — bind stores a reference
    auto f2 = std::bind(increment, std::ref(value));
    f2();
    std::cout << value << "\n"; // 11 — changed!

    // Same pattern with std::thread:
    // std::thread t(increment, std::ref(value)); // pass by reference
    // t.join();
}
```

This is the reason this trips people up: when you write `std::bind(f, x)` hoping `x` will be modified, you're copying `x`. The fix is `std::bind(f, std::ref(x))`.

---

## Self-Assessment

### Q1: Store references to objects in a std::vector using `std::reference_wrapper<T>`

**Answer:**

```cpp
#include <functional>
#include <vector>
#include <iostream>
#include <string>
#include <algorithm>

struct Student {
    std::string name;
    int grade;
};

int main() {
    Student alice{"Alice", 85};
    Student bob{"Bob", 92};
    Student charlie{"Charlie", 78};

    // Vector of references to Student objects
    std::vector<std::reference_wrapper<Student>> students = {alice, bob, charlie};

    // Modify through the vector — changes original objects
    for (auto& s : students)
        s.get().grade += 5; // curve: add 5 to everyone

    // Verify originals changed
    std::cout << alice.name << ": " << alice.grade << "\n";    // Alice: 90
    std::cout << bob.name << ": " << bob.grade << "\n";        // Bob: 97
    std::cout << charlie.name << ": " << charlie.grade << "\n"; // Charlie: 83

    // Can also sort the references by grade
    std::sort(students.begin(), students.end(),
              [](const Student& a, const Student& b) {
                  return a.grade > b.grade; // descending
              });

    std::cout << "\nTop students:\n";
    for (const auto& s : students)
        std::cout << "  " << s.get().name << ": " << s.get().grade << "\n";
    // Bob: 97
    // Alice: 90
    // Charlie: 83
}
```

`std::reference_wrapper<Student>` wraps a reference to `Student` as a copyable object that `std::vector` can store. The implicit conversion to `Student&` means algorithms like `std::sort` work transparently. Modifications through the wrapper affect the original objects.

### Q2: Show that containers of references are impossible without reference_wrapper

**Answer:**

```cpp
#include <functional>
#include <vector>
#include <map>
#include <iostream>
#include <array>
#include <string>
#include <utility>

int main() {
    int a = 10, b = 20, c = 30;

    // IMPOSSIBLE — compile errors:
    // std::vector<int&> v1;                    // ERROR: forming pointer to reference
    // std::map<int, int&> m1;                  // ERROR: reference as mapped type
    // std::pair<int, int&> p1;                 // ERROR (in some contexts)
    // std::array<int&, 3> arr;                 // ERROR

    // WHY? Container elements must be:
    // 1. Default-constructible (for resize, map[key], etc.)
    //    -> References MUST be initialized; there's no default
    // 2. Copy-assignable (for push_back, insert, etc.)
    //    -> Assigning to a reference changes the REFERENT, not the reference
    // 3. Destructible
    //    -> References have no destructor

    // WORKS — reference_wrapper IS an object:
    std::vector<std::reference_wrapper<int>> v = {a, b, c};
    std::map<std::string, std::reference_wrapper<int>> m = {
        {"a", std::ref(a)},
        {"b", std::ref(b)},
    };

    // Rebinding: reference_wrapper CAN be reassigned
    v[0] = std::ref(c); // now v[0] refers to c, not a
    v[0].get() = 99;
    std::cout << "c = " << c << "\n"; // c = 99

    // References can't rebind:
    int& r = a;
    r = b;  // this assigns b's VALUE to a, does NOT rebind r!
    std::cout << "a = " << a << "\n"; // a = 20 (a changed, r still refers to a)

    // Map lookup modifying original:
    m["b"].get() = 42;
    std::cout << "b = " << b << "\n"; // b = 42
}
```

C++ references are aliases, not objects. They lack the value semantics (copyable, assignable, destructible) that containers require. `reference_wrapper` is a regular object that internally stores a pointer but provides reference-like semantics with implicit conversion to `T&`. This bridges the gap, allowing "containers of references" that are actually containers of reference_wrapper objects.

### Q3: Use std::cref and std::ref to create reference wrappers and pass them to std::bind

**Answer:**

```cpp
#include <functional>
#include <iostream>
#include <string>

void print_and_modify(std::string& s, const std::string& prefix) {
    std::cout << prefix << s << "\n";
    s += "!";
}

int main() {
    std::string message = "Hello";
    std::string prefix = ">> ";

    // std::bind COPIES arguments by default.
    // Use std::ref to pass by reference, std::cref for const reference.

    // Without ref — bind copies 'message', modifications are invisible
    auto bound_copy = std::bind(print_and_modify,
                                message,            // COPY
                                std::cref(prefix));  // const ref (cheap, no copy)
    bound_copy();
    std::cout << "message after copy-bind: " << message << "\n";
    // >> Hello
    // message after copy-bind: Hello  (unchanged — bind modified its copy)

    // With ref — bind stores a reference to 'message'
    auto bound_ref = std::bind(print_and_modify,
                               std::ref(message),   // reference (modifies original)
                               std::cref(prefix));   // const ref
    bound_ref();
    std::cout << "message after ref-bind: " << message << "\n";
    // >> Hello
    // message after ref-bind: Hello!  (original modified!)

    // std::ref vs std::cref:
    // std::ref(x)  -> reference_wrapper<T>        (read + write)
    // std::cref(x) -> reference_wrapper<const T>  (read-only)

    // Also works with std::thread, std::async:
    // std::thread t(func, std::ref(shared_data));
    // auto fut = std::async(std::launch::async, func, std::ref(result));

    // Modern alternative — lambdas capture by reference directly:
    auto lambda_ref = [&message, &prefix]() {
        print_and_modify(message, prefix);
    };
    lambda_ref();
    std::cout << "message after lambda: " << message << "\n";
    // >> Hello!!
    // message after lambda: Hello!! (modified again)
}
```

`std::bind` copies all its arguments by default. `std::ref(x)` creates a `reference_wrapper<T>` that `bind` stores instead - when the bound function is called, the wrapper implicitly converts back to `T&`, so modifications affect the original. `std::cref(x)` does the same for `const T&`. Modern C++ prefers lambdas with capture-by-reference (`[&x]`) over `bind`, but `ref`/`cref` remain important for `std::thread` constructors and other forwarding contexts.

---

## Notes

- **Implicit conversion:** `reference_wrapper<T>` has `operator T&()`, so it implicitly converts to `T&` in most contexts (function calls, comparisons, etc.). Use `.get()` when the implicit conversion is ambiguous.
- **Not nullable:** Unlike raw pointers, `reference_wrapper` must always refer to a valid object. Constructing one from a null/dangling reference is UB.
- **Cost:** `reference_wrapper` is pointer-sized. Copy/assign are trivial (pointer copy). No overhead vs raw pointers.
- **With `std::thread`:** Always use `std::ref` when passing arguments by reference to `std::thread`: `std::thread(func, std::ref(data))`. Without it, `thread` copies the argument.
- **C++20 ranges:** Some range adaptors that store callable objects use `reference_wrapper` internally for reference semantics.
- **`invoke`:** `std::reference_wrapper<T>` satisfies `std::invocable` when `T` is callable - so `std::ref(func)` can be passed to algorithms expecting callables.
- Compile with `-std=c++20 -Wall -Wextra`.
