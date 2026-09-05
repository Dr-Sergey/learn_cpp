# Use std::initializer_list correctly in constructors and assignment

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/initializer_list>  

---

## Topic Overview

`std::initializer_list<T>` is what makes the `{1, 2, 3}` brace-initialization syntax work for your own types. It's a lightweight proxy - just two pointers wrapping a compiler-generated array of `const T` objects. You take it in a constructor, iterate the values, and copy them into your storage. It's simple to use, but it has a few sharp edges that bite people regularly.

### Key Properties

| Property | Value |
| --- | --- |
| Element type | `const T` (always const) |
| Copyable | Yes (cheap - copies pointers, not elements) |
| Move from elements | **No** - elements are `const` |
| Storage | Compiler-generated array on the **caller's stack** |
| Lifetime | Same as the full expression containing the braces |

### Brace-Init Priority Rule

This is the rule that surprises people most. When a class has **both** an `initializer_list` constructor and other constructors, brace-init **strongly prefers** the `initializer_list` overload - even when the other constructor seems like a better fit:

```cpp
struct Widget {
    Widget(int a, int b);                    // (1)
    Widget(std::initializer_list<int> il);   // (2)
};

Widget w1(10, 20);    // calls (1) — parentheses
Widget w2{10, 20};    // calls (2) — braces prefer initializer_list!
Widget w3 = {10, 20}; // calls (2) — braces
```

### Core Syntax

Here's a minimal class that accepts brace-initialization. The `size()` member of the `initializer_list` tells you how many elements were provided:

```cpp
#include <initializer_list>
#include <vector>
#include <iostream>

class IntList {
    std::vector<int> data_;
public:
    IntList(std::initializer_list<int> il) : data_(il) {
        std::cout << "init_list ctor: " << il.size() << " elements\n";
    }

    void print() const {
        for (int v : data_) std::cout << v << " ";
        std::cout << "\n";
    }
};

int main() {
    IntList a{1, 2, 3, 4, 5};
    a.print();
    // Output:
    // init_list ctor: 5 elements
    // 1 2 3 4 5
}
```

---

## Self-Assessment

### Q1: Write a constructor taking initializer_list<int> and explain why it takes priority in brace-init

**Answer:**

The example below makes the priority rule tangible. Notice that `d{}` (empty braces) also calls the `initializer_list` constructor - with an empty list. That's the correct behavior, but it can be surprising if you expected the default constructor to run:

```cpp
#include <initializer_list>
#include <iostream>
#include <string>

class Container {
    int count_;
    int value_;
public:
    // Constructor 1: count + value
    Container(int count, int value) : count_(count), value_(value) {
        std::cout << "  (int, int) ctor: count=" << count_
                  << " value=" << value_ << "\n";
    }

    // Constructor 2: initializer_list
    Container(std::initializer_list<int> il) : count_(0), value_(0) {
        std::cout << "  init_list ctor: { ";
        for (int v : il) std::cout << v << " ";
        std::cout << "} (" << il.size() << " elements)\n";
    }
};

int main() {
    std::cout << "a(5, 10):\n";
    Container a(5, 10);       // (int, int) — parentheses bypass init_list
    // Output: (int, int) ctor: count=5 value=10

    std::cout << "b{5, 10}:\n";
    Container b{5, 10};       // init_list<int>{5, 10} — braces prefer init_list!
    // Output: init_list ctor: { 5 10 } (2 elements)

    std::cout << "c = {5, 10}:\n";
    Container c = {5, 10};    // init_list — copy-list-initialization
    // Output: init_list ctor: { 5 10 } (2 elements)

    std::cout << "d{}:\n";
    Container d{};            // Default? No — empty init_list!
    // Output: init_list ctor: { } (0 elements)

    // To force the (int, int) constructor with braces, you'd need to
    // remove the init_list constructor, or use parentheses.
}
```

**Why init_list takes priority:**
The C++ standard (§12.6.2) specifies that if a class has an `initializer_list` constructor and the initializer is a braced-init-list, the `initializer_list` overload is **always considered first** before any other constructors. Only if `initializer_list` conversion fails does the compiler fall back to other constructors.

This is a deliberate design: `std::vector<int> v{10, 20}` creates a 2-element vector `{10, 20}`, not a 10-element vector filled with `20`.

---

### Q2: Show that iterating a initializer_list gives const references and you cannot move from it

**Answer:**

The reason you cannot move from an `initializer_list` trips people up because it looks like it should work. You have a temporary list of strings, you want to move them into a vector - that seems reasonable. But the elements are `const T`, and `std::move` on a `const` object yields `const T&&`, which binds to the copy constructor, not the move constructor. The move silently becomes a copy:

```cpp
#include <initializer_list>
#include <iostream>
#include <string>
#include <type_traits>
#include <vector>

int main() {
    // --- Elements are CONST ---
    auto il = {1, 2, 3};
    static_assert(std::is_same_v<decltype(*il.begin()), const int&>);
    // The elements are `const int`, not `int`

    // Cannot modify:
    // *il.begin() = 42;  // COMPILE ERROR: assignment of read-only location

    // --- Cannot move from initializer_list elements ---
    auto strings = {std::string("hello"), std::string("world")};
    for (auto& s : strings) {
        static_assert(std::is_same_v<decltype(s), const std::string&>);
        // s is const string& — cannot move from it
    }

    // This means: initializer_list<string> always COPIES strings
    std::vector<std::string> vec = {"alpha", "beta", "gamma"};
    // Each string was COPIED from the init_list backing array into the vector
    // No moves occurred!

    // --- Proof: move doesn't actually move ---
    struct Tracked {
        std::string name;
        Tracked(const char* n) : name(n) { std::cout << "  ctor: " << name << "\n"; }
        Tracked(const Tracked& o) : name(o.name) { std::cout << "  COPY: " << name << "\n"; }
        Tracked(Tracked&& o) noexcept : name(std::move(o.name)) { std::cout << "  MOVE: " << name << "\n"; }
    };

    std::cout << "Creating vector from init_list:\n";
    std::vector<Tracked> v = {Tracked("a"), Tracked("b")};
    // Output shows:
    //   ctor: a
    //   ctor: b
    //   COPY: a   (copied from init_list, NOT moved!)
    //   COPY: b

    std::cout << "\nSize: " << v.size() << "\n";
}
```

**Why no moves?** The backing array of `initializer_list` contains `const T` objects. `std::move` on a `const` object yields a `const T&&`, which binds to the **copy** constructor, not the move constructor. This is an inherent limitation of `initializer_list`.

---

### Q3: Explain the lifetime: an initializer_list references a backing array with the caller's lifetime

**Answer:**

This is the most dangerous `initializer_list` pitfall. The object itself is just a pair of pointers into a compiler-generated temporary array. If that array is destroyed - because the function that owned it returned, or because the full expression ended - your `initializer_list` is left dangling.

The rule of thumb is simple: pass it, use it immediately, and let it go. Never store it, never return it.

```cpp
#include <initializer_list>
#include <iostream>
#include <string>
#include <vector>

// SAFE: backing array lives for the duration of the full expression
void print(std::initializer_list<int> il) {
    for (int v : il) std::cout << v << " ";
    std::cout << "\n";
}

// DANGEROUS: returning an initializer_list
std::initializer_list<int> bad_function() {
    return {1, 2, 3};  // backing array is local to this function!
}
// The returned init_list points to a destroyed temporary array -> DANGLING!

// SAFE alternative: return a container
std::vector<int> good_function() {
    return {1, 2, 3};  // vector copies the data, owns it
}

class Widget {
    std::initializer_list<int> il_;  // DANGEROUS: don't store init_lists!
public:
    Widget(std::initializer_list<int> il) : il_(il) {}
    // il_ now points to a backing array whose lifetime may have ended

    void print() {
        for (int v : il_) std::cout << v << " ";  // UB if backing array is gone
        std::cout << "\n";
    }
};

int main() {
    // SAFE: direct use in the same expression
    print({10, 20, 30});
    // Output: 10 20 30

    // SAFE: bound to a local variable — lifetime extended
    auto local = {1, 2, 3};
    for (int v : local) std::cout << v << " ";
    std::cout << "\n";
    // Output: 1 2 3

    // DANGEROUS: returning from function
    // auto bad = bad_function();
    // for (int v : bad) std::cout << v;  // UB: dangling pointers!

    // DANGEROUS: storing in a member
    Widget w({4, 5, 6});
    // w.print();  // UB: backing array {4,5,6} was a temporary, now destroyed

    // SAFE: use a container instead of storing init_list
    auto safe = good_function();
    for (int v : safe) std::cout << v << " ";
    std::cout << "\n";
    // Output: 1 2 3
}
```

**Rules of thumb:**

1. **Never store** `std::initializer_list` as a member variable.
2. **Never return** `std::initializer_list` from a function.
3. **Always copy** elements into a container (`vector`, `array`) if you need to keep them.
4. Passing `initializer_list` as a function parameter is fine - the backing array lives for the duration of the caller's full expression.

---

## Notes

- `std::initializer_list` is in `<initializer_list>`, but including any standard container header typically brings it in.
- Copying an `initializer_list` is cheap - it copies the pointers, not the elements. But the copy shares the same backing array.
- `auto x = {1, 2, 3}` deduces `std::initializer_list<int>`. `auto x = {1}` also deduces `initializer_list<int>` (not `int`).
- In C++17, `auto x{1}` deduces `int` (changed from C++11 where it was `initializer_list<int>`).
- `std::vector<int> v(10, 1)` creates 10 elements of value 1. `std::vector<int> v{10, 1}` creates 2 elements: 10 and 1. Know the difference!
