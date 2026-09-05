# Use range-based for loops correctly and know their caveats

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP11.md#range-based-for-loops>  

---

## Topic Overview

The **range-based for loop** (C++11) is the loop you reach for when you just want to walk over every element of a container without fussing over indices or iterators. It reads almost like English: "for each `x` in `numbers`, do this."

### Basic Syntax

The whole game is choosing how you grab each element - by value, by reference, or by const reference - because that choice decides whether you copy, whether you can modify, and how much it costs:

```cpp
std::vector<int> numbers = {1, 2, 3, 4, 5};

// Three forms - choose based on your needs:

// (1) By value - copies each element
for (auto x : numbers) {
    x *= 2;  // modifies the COPY, not the original
}
// numbers is unchanged: {1, 2, 3, 4, 5}

// (2) By reference - can modify elements
for (auto& x : numbers) {
    x *= 2;  // modifies the ORIGINAL
}
// numbers is now: {2, 4, 6, 8, 10}

// (3) By const reference - read-only, no copy (BEST for reading)
for (const auto& x : numbers) {
    std::cout << x << " ";
    // x *= 2;  // ERROR: x is const
}
```

If you remember only one thing: reach for `const auto&` by default, and switch to `auto&` only when you actually need to change the elements.

### How It Works Internally

The loop isn't magic - the compiler simply rewrites it into an ordinary iterator loop. Seeing the expansion makes every later pitfall obvious, because each pitfall is really just a problem in this generated code:

```cpp
for (auto& x : container) { body; }

// is equivalent to:
{
    auto&& __range = container;
    auto __begin = std::begin(__range);
    auto __end = std::end(__range);
    for (; __begin != __end; ++__begin) {
        auto& x = *__begin;
        body;
    }
}
```

Keep that `auto&& __range = container;` line in mind - it's where the dangling-temporary trap lives.

### Common Pitfall: Copying Expensive Objects

For `int` the copy is free, so nobody cares. For `std::string` (or any heavy object) the by-value form silently copies every single element, and that adds up:

```cpp
std::vector<std::string> names = {"Alice", "Bob", "Charlie"};

// BAD: copies every string! Expensive for long strings.
for (auto name : names) {
    std::cout << name << "\n";
}

// GOOD: no copies, just references
for (const auto& name : names) {
    std::cout << name << "\n";
}
```

### Dangerous: Iterating Over Temporaries

This is the subtle one. If the range expression produces a temporary and you bind a reference into it, the temporary can be gone by the time the loop body runs - leaving you reading freed memory:

```cpp
std::vector<int> get_data() { return {1, 2, 3}; }

// DANGEROUS: temporary vector destroyed before loop body!
// for (auto& x : get_data()) {  // vector is temporary
//     std::cout << x;            // UB: vector already destroyed!
// }

// SAFE: capture the temporary in a variable
auto data = get_data();
for (auto& x : data) {
    std::cout << x;  // OK: data lives until end of scope
}

// SAFE in C++23: lifetime extended for range-for temporaries
// for (auto& x : get_data()) { ... } // OK in C++23
```

The reliable habit is to give the temporary a name first. C++23 finally extends the lifetime for you, but until you can assume C++23 everywhere, the named-variable rule keeps you safe.

### Iterating Over Different Types

The same syntax works across arrays, maps, strings, and even brace-enclosed lists - anything that exposes a begin/end pair:

```cpp
// Arrays:
int arr[] = {10, 20, 30};
for (int x : arr) { std::cout << x; }

// std::map (structured bindings):
std::map<std::string, int> scores = {{"Alice", 95}, {"Bob", 87}};
for (const auto& [name, score] : scores) {
    std::cout << name << ": " << score << "\n";
}

// std::string (characters):
for (char c : std::string("hello")) {
    std::cout << c << " ";
}
// Output: h e l l o

// Initializer list:
for (int x : {10, 20, 30}) {
    std::cout << x << " ";
}
```

### Making Custom Types Work With Range-For

Because the loop only needs a `begin()` and an `end()`, you can make any type of your own iterable just by supplying those two functions plus a minimal iterator:

```cpp
class IntRange {
    int start_, end_;
public:
    IntRange(int s, int e) : start_(s), end_(e) {}

    struct Iterator {
        int current;
        int operator*() const { return current; }
        Iterator& operator++() { ++current; return *this; }
        bool operator!=(const Iterator& other) const { return current != other.current; }
    };

    Iterator begin() const { return {start_}; }
    Iterator end() const   { return {end_}; }
};

// Usage:
for (int x : IntRange(1, 6)) {
    std::cout << x << " ";
}
// Output: 1 2 3 4 5
```

If you can't add members to a type (say it's a legacy struct), free `begin`/`end` functions work just as well - the loop finds them through ADL:

```cpp
struct LegacyContainer {
    int data[5] = {10, 20, 30, 40, 50};
    int size = 5;
};

// Free functions enable range-for:
int* begin(LegacyContainer& c) { return c.data; }
int* end(LegacyContainer& c)   { return c.data + c.size; }

LegacyContainer lc;
for (int x : lc) { std::cout << x << " "; }  // 10 20 30 40 50
```

---

## Self-Assessment

### Q1: Explain why for (auto x : vec) copies elements and when for (auto& x : vec) is preferred

The clearest way to feel the difference is to try to modify elements with each form and see which one actually sticks:

```cpp
#include <iostream>
#include <vector>
#include <string>

int main() {
    std::vector<std::string> words = {"hello", "world", "test"};

    // for (auto x : vec) COPIES each element:
    for (auto word : words) {
        word = "MODIFIED";  // modifies the copy, not the original
    }
    // words is still: {"hello", "world", "test"}

    std::cout << "After auto x loop: ";
    for (const auto& w : words) std::cout << w << " ";
    std::cout << "\n"; // hello world test - unchanged!

    // for (auto& x : vec) REFERENCES each element:
    for (auto& word : words) {
        word = "MODIFIED";  // modifies the original!
    }
    std::cout << "After auto& x loop: ";
    for (const auto& w : words) std::cout << w << " ";
    std::cout << "\n"; // MODIFIED MODIFIED MODIFIED

    // Rule of thumb:
    // - auto x : v        -> use when elements are cheap to copy (int, char, etc.)
    // - auto& x : v       -> use when you need to modify elements
    // - const auto& x : v -> use for read-only access to expensive elements (strings, etc.)
}
```

By value gives you a private copy, so writes vanish when the iteration moves on. By reference, you're touching the real element, so writes persist. That's the entire distinction.

### Q2: Show a bug where range-based for iterates over a temporary that gets destroyed

Recall the expansion from earlier - the temporary gets bound to `__range`, and the danger is whether it survives long enough. The really sneaky version is a *chained* call, where you bind a reference into a member of a temporary:

```cpp
#include <iostream>
#include <vector>

std::vector<int> make_numbers() {
    return {1, 2, 3, 4, 5};
}

int main() {
    // BUG: The temporary returned by make_numbers() is destroyed
    // BEFORE the loop body executes.
    //
    // Range-for internally does:
    //   auto&& __range = make_numbers();  // temporary created and bound
    //   auto __begin = __range.begin();   // OK so far
    //   // ... but the temporary may be destroyed here (implementation-defined)
    //
    // This was problematic in C++11-C++20 for certain expressions.
    //
    // More insidious example:
    // for (auto& x : get_object().get_container()) {
    //     // get_object() returns a temporary
    //     // .get_container() returns a reference to a member of that temporary
    //     // The temporary is destroyed; the reference is dangling
    //     std::cout << x; // UNDEFINED BEHAVIOR
    // }

    // FIX: Store in a named variable
    auto numbers = make_numbers();
    for (auto x : numbers) {
        std::cout << x << " ";
    }
    std::cout << "\n";

    // Also safe: the temporary's lifetime is extended for SIMPLE expressions
    for (auto x : make_numbers()) {  // this specific case IS safe
        std::cout << x << " ";
    }
    // The DANGEROUS case is chained calls: temporary.method() where method returns reference
}
```

A bare `make_numbers()` is actually fine, because lifetime extension covers that simple case. The killer is `get_object().get_container()`: lifetime extension only applies to the *outermost* temporary, not to a reference handed back from inside it.

### Q3: Demonstrate how to write a custom type that supports range-based for by providing begin()/end()

Here's a slightly meatier custom range - a Fibonacci generator. The iterator carries the running state and `end()` is just a sentinel that compares equal when the count runs out:

```cpp
#include <iostream>

// A Fibonacci sequence range
class Fibonacci {
    int count_;
public:
    explicit Fibonacci(int count) : count_(count) {}

    struct Iterator {
        int remaining;
        int current = 0;
        int next = 1;

        int operator*() const { return current; }

        Iterator& operator++() {
            int temp = current + next;
            current = next;
            next = temp;
            --remaining;
            return *this;
        }

        bool operator!=(const Iterator& other) const {
            return remaining != other.remaining;
        }
    };

    Iterator begin() const { return {count_, 0, 1}; }
    Iterator end() const   { return {0, 0, 0}; }
};

int main() {
    // Works with range-based for because it has begin() and end():
    std::cout << "First 10 Fibonacci numbers: ";
    for (int fib : Fibonacci(10)) {
        std::cout << fib << " ";
    }
    // Output: 0 1 1 2 3 5 8 13 21 34
    std::cout << "\n";

    // Also works with range-for shorthand:
    for (int x : Fibonacci(5)) {
        std::cout << x << " ";
    }
    // Output: 0 1 1 2 3
}
```

Notice the loop never knows or cares that these numbers are computed on the fly - as far as range-for is concerned, `begin()`/`end()` plus `*`, `++`, and `!=` is the whole contract.

---

## Notes

- **Default to `const auto&`** for reading elements - avoids copies and prevents accidental modification.
- `auto&&` (forwarding reference) is useful in generic code but rarely needed in everyday loops.
- `std::vector<bool>` is a special case: `for (auto x : boolVec)` returns proxy objects, not `bool&`. Use `auto&&` or explicit `bool`.
- C++20 adds `for (auto x : range | views::...)` - range adaptors compose naturally with range-for.
- C++23 fixes the dangling temporary problem: `for (auto& x : get_obj().member())` is safe in C++23.
- **Don't modify the container** (insert/erase) inside a range-for loop - it invalidates iterators and causes UB. Use `std::erase_if` (C++20) instead.
- Range-for requires `begin()` and `end()` - alternatively, free functions found by ADL work too.
