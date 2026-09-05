# Understand the Concepts Standard Library: `std::regular`, `std::semiregular`, `std::copyable`

**Category:** Type System and Deduction  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/concepts>  

---

## Topic Overview

### The Standard Concept Hierarchy

C++20 defines a hierarchy of concepts that describe the "regularity" of a type - how value-like it behaves. Think of it as a ladder: each rung adds one more thing a type must be able to do:

```cpp
std::destructible
  └── std::movable              (destructible + move constructible + move assignable + swappable)
        └── std::copyable       (movable + copy constructible + copy assignable)
              └── std::semiregular  (copyable + default_initializable)
                    └── std::regular    (semiregular + equality_comparable)
```

### What Each Concept Requires

| Concept | Requirements | Example types that satisfy |
| --- | --- | --- |
| `std::movable` | Move ctor, move assign, destructible, swappable | `unique_ptr`, `thread` |
| `std::copyable` | Movable + copy ctor + copy assign | `shared_ptr`, `string` |
| `std::semiregular` | Copyable + default constructible | `int`, `string`, `vector` |
| `std::regular` | Semiregular + `==` and `!=` | `int`, `string`, `vector` |

### `std::regular` - The Gold Standard for Value Types

A **regular** type behaves like `int` - you can default-construct it, copy it, compare it for equality, and copies are independent values. If you're designing a value type, regular is the target:

```cpp
#include <concepts>

// int is regular:
static_assert(std::regular<int>);

// std::string is regular:
static_assert(std::regular<std::string>);

// std::vector<int> is regular:
static_assert(std::regular<std::vector<int>>);
```

**Semantic requirements** (beyond syntax):

- Copies are equal: `T a = b;` implies `a == b`
- Copy is independent: modifying `a` after `T a = b;` doesn't change `b`
- Default value exists: `T{}` is a valid, well-defined state

### Why These Concepts Matter

The power of this hierarchy is that you can constrain a template to exactly the capabilities it needs - no more, no less:

```cpp
// Only needs movable (e.g., unique ownership transfer)
template<std::movable T>
void take_ownership(T obj);

// Needs copyable (e.g., storing copies in a cache)
template<std::copyable T>
class Cache { /* stores copies of T */ };

// Needs semiregular (e.g., default-constructible sentinel)
template<std::semiregular T>
class Optional { T default_value = T{}; };

// Needs regular (e.g., elements in a set — need == for uniqueness)
template<std::regular T>
class UniqueList { /* compares elements for equality */ };
```

Using the weakest concept that satisfies your needs is good design: it makes your template usable with more types.

### Types That Fail at Each Level

Here's the hierarchy illustrated by what breaks at each step. This is the most useful mental model to internalize:

```cpp
#include <concepts>
#include <memory>
#include <mutex>

// std::mutex: not movable, not copyable, not anything
static_assert(!std::movable<std::mutex>);

// std::unique_ptr: movable but NOT copyable
static_assert(std::movable<std::unique_ptr<int>>);
static_assert(!std::copyable<std::unique_ptr<int>>);

// A type with no default ctor: copyable but NOT semiregular
struct NoDflt {
    int x;
    NoDflt(int v) : x(v) {}
    auto operator<=>(const NoDflt&) const = default;
};
static_assert(std::copyable<NoDflt>);
static_assert(!std::semiregular<NoDflt>);

// A type with no ==: semiregular but NOT regular
struct NoEq {
    int x = 0;
};
static_assert(std::semiregular<NoEq>);
static_assert(!std::regular<NoEq>);
```

### Formal Definitions (C++20 Standard)

For completeness, here are the actual concept definitions. Notice how each one builds on the one before:

```cpp
// Actual concept definitions (simplified from <concepts>):
template<class T>
concept movable = std::is_object_v<T> && std::move_constructible<T>
    && std::assignable_from<T&, T> && std::swappable<T>;

template<class T>
concept copyable = std::copy_constructible<T> && std::movable<T>
    && std::assignable_from<T&, T&>
    && std::assignable_from<T&, const T&>
    && std::assignable_from<T&, const T>;

template<class T>
concept semiregular = std::copyable<T> && std::default_initializable<T>;

template<class T>
concept regular = std::semiregular<T> && std::equality_comparable<T>;
```

---

## Self-Assessment

### Q1: Explain what `std::regular` requires (default constructible, copyable, equality comparable)

The `check_regular_requirements` function here is a handy diagnostic tool - it prints all the sub-requirements so you can see which one is failing for a given type:

```cpp
#include <iostream>
#include <concepts>
#include <type_traits>
#include <string>
#include <vector>

// std::regular = std::semiregular + std::equality_comparable
// Which expands to:
//   std::copyable + std::default_initializable + operator== and operator!=
// Which further expands to:
//   std::movable + copy ctor + copy assign + default ctor + == + !=

// Let's verify each requirement individually:
template<typename T>
void check_regular_requirements() {
    std::cout << "  destructible:          " << std::destructible<T> << "\n";
    std::cout << "  move_constructible:    " << std::move_constructible<T> << "\n";
    std::cout << "  copy_constructible:    " << std::copy_constructible<T> << "\n";
    std::cout << "  default_initializable: " << std::default_initializable<T> << "\n";
    std::cout << "  equality_comparable:   " << std::equality_comparable<T> << "\n";
    std::cout << "  movable:               " << std::movable<T> << "\n";
    std::cout << "  copyable:              " << std::copyable<T> << "\n";
    std::cout << "  semiregular:           " << std::semiregular<T> << "\n";
    std::cout << "  REGULAR:               " << std::regular<T> << "\n";
}

// Example: int satisfies everything
// Example: unique_ptr fails at copyable

int main() {
    std::cout << std::boolalpha;

    std::cout << "=== int ===\n";
    check_regular_requirements<int>();

    std::cout << "\n=== std::string ===\n";
    check_regular_requirements<std::string>();

    std::cout << "\n=== std::vector<int> ===\n";
    check_regular_requirements<std::vector<int>>();

    std::cout << "\n=== std::unique_ptr<int> ===\n";
    check_regular_requirements<std::unique_ptr<int>>();

    // Semantic requirements (verified by convention, not by compiler):
    std::cout << "\n=== Semantic requirements of regular ===\n";
    int a = 42, b = a;
    std::cout << "Copy equality: (a == b) = " << (a == b) << "\n";

    b = 100;
    std::cout << "Independence: a=" << a << ", b=" << b
              << " (modifying copy doesn't change original)\n";

    int c{};
    std::cout << "Default value: int{} = " << c << "\n";

    return 0;
}
```

**Output:**

```text
=== int ===
  destructible:          true
  move_constructible:    true
  copy_constructible:    true
  default_initializable: true
  equality_comparable:   true
  movable:               true
  copyable:              true
  semiregular:           true
  REGULAR:               true

=== std::string ===
  (all true, REGULAR: true)

=== std::vector<int> ===
  (all true, REGULAR: true)

=== std::unique_ptr<int> ===
  destructible:          true
  move_constructible:    true
  copy_constructible:    false
  default_initializable: true
  equality_comparable:   true
  movable:               true
  copyable:              false
  semiregular:           false
  REGULAR:               false
```

### Q2: Show why `std::unique_ptr` does not satisfy `std::copyable`

`std::unique_ptr` is deliberately move-only. Allowing copies would mean two `unique_ptr`s pointing to the same object - a double-free waiting to happen. The `= delete` on the copy operations is a design choice, not an oversight:

```cpp
#include <iostream>
#include <concepts>
#include <memory>
#include <type_traits>
#include <mutex>
#include <thread>
#include <utility>

int main() {
    std::cout << std::boolalpha;

    // std::unique_ptr is move-only — it has deleted copy operations
    std::cout << "=== std::unique_ptr<int> ===\n";
    std::cout << "  move_constructible:    " << std::move_constructible<std::unique_ptr<int>> << "\n";  // true
    std::cout << "  copy_constructible:    " << std::copy_constructible<std::unique_ptr<int>> << "\n";  // false
    std::cout << "  movable:               " << std::movable<std::unique_ptr<int>> << "\n";             // true
    std::cout << "  copyable:              " << std::copyable<std::unique_ptr<int>> << "\n";             // false

    // Why? unique_ptr explicitly deletes copy operations:
    // unique_ptr(const unique_ptr&) = delete;
    // unique_ptr& operator=(const unique_ptr&) = delete;

    // This is by design: unique_ptr models exclusive ownership
    // Allowing copies would violate the "unique" invariant

    // Demonstrate: move works, copy doesn't
    auto p1 = std::make_unique<int>(42);
    // auto p2 = p1;              // ERROR: copy deleted
    auto p2 = std::move(p1);     // OK: move transfers ownership

    std::cout << "\nAfter move:\n";
    std::cout << "  p1 is null: " << (p1 == nullptr) << "\n";
    std::cout << "  p2 value: " << *p2 << "\n";

    // Other move-only types:
    std::cout << "\n=== Other non-copyable types ===\n";
    std::cout << "  std::mutex copyable: " << std::copyable<std::mutex> << "\n";           // false
    std::cout << "  std::thread copyable: " << std::copyable<std::jthread> << "\n";         // false
    std::cout << "  std::ifstream copyable: " << std::copyable<std::ifstream> << "\n";      // false

    // These all represent non-shareable resources (heap memory, OS mutex, thread, file handle)

    return 0;
}
```

**Output:**

```text
=== std::unique_ptr<int> ===
  move_constructible:    true
  copy_constructible:    false
  movable:               true
  copyable:              false

After move:
  p1 is null: true
  p2 value: 42

=== Other non-copyable types ===
  std::mutex copyable: false
  std::thread copyable: false
  std::ifstream copyable: false
```

### Q3: Write a generic algorithm constrained on `std::regular` and verify it rejects non-regular types

The concept constraint here does real work - it's not decoration. Without `==` you can't check for duplicates, and without copyability you can't `push_back`. The compiler will tell you exactly which concept fails when you try a non-regular type:

```cpp
#include <iostream>
#include <concepts>
#include <vector>
#include <string>
#include <memory>
#include <utility>

// Algorithm constrained on std::regular:
// "Remove consecutive duplicates" — needs == (equality) and copy
template<std::regular T>
std::vector<T> unique_elements(const std::vector<T>& input) {
    std::vector<T> result;
    for (const auto& elem : input) {
        if (result.empty() || result.back() != elem) {
            result.push_back(elem);  // requires copyable
        }
    }
    return result;
}

// A function that needs semiregular (default ctor but no ==)
template<std::semiregular T>
T create_default() {
    return T{};  // requires default_initializable
}

// A function that needs only movable
template<std::movable T>
void consume(T obj) {
    // Just takes ownership
    (void)obj;
}

// Non-regular types for testing:
struct NoEquality {
    int x = 0;
    // No operator==!
};

struct NonCopyable {
    int x;
    NonCopyable(int v) : x(v) {}
    NonCopyable(NonCopyable&&) = default;
    NonCopyable(const NonCopyable&) = delete;
    auto operator<=>(const NonCopyable&) const = default;
};

int main() {
    // Works with regular types:
    std::vector<int> nums = {1, 1, 2, 3, 3, 3, 4};
    auto unique_nums = unique_elements(nums);
    std::cout << "Unique ints: ";
    for (int x : unique_nums) std::cout << x << " ";
    std::cout << "\n";

    std::vector<std::string> words = {"hello", "hello", "world", "world", "!"};
    auto unique_words = unique_elements(words);
    std::cout << "Unique strings: ";
    for (const auto& w : unique_words) std::cout << w << " ";
    std::cout << "\n";

    // These would FAIL to compile — uncomment to see concept errors:

    // NoEquality has no operator==, so it's not regular:
    // std::vector<NoEquality> ne = {{1}, {2}};
    // auto bad1 = unique_elements(ne);
    // ERROR: constraints not satisfied: std::regular<NoEquality> is false
    //        because std::equality_comparable<NoEquality> is false

    // NonCopyable is not copyable, so not regular:
    // std::vector<NonCopyable> nc;
    // auto bad2 = unique_elements(nc);
    // ERROR: constraints not satisfied: std::regular<NonCopyable> is false
    //        because std::copyable<NonCopyable> is false

    // But they work with less-constrained functions:
    NoEquality ne;
    create_default<NoEquality>();       // OK: NoEquality is semiregular

    NonCopyable nc{42};
    consume(std::move(nc));             // OK: NonCopyable is movable

    std::cout << "\nConcept hierarchy verification:\n";
    std::cout << std::boolalpha;
    std::cout << "  int:           regular=" << std::regular<int> << "\n";
    std::cout << "  NoEquality:    regular=" << std::regular<NoEquality>
              << ", semiregular=" << std::semiregular<NoEquality> << "\n";
    std::cout << "  NonCopyable:   regular=" << std::regular<NonCopyable>
              << ", movable=" << std::movable<NonCopyable> << "\n";

    return 0;
}
```

**Output:**

```text
Unique ints: 1 2 3 4
Unique strings: hello world !

Concept hierarchy verification:
  int:           regular=true
  NoEquality:    regular=false, semiregular=true
  NonCopyable:   regular=false, movable=true
```

**How this works:**

- `unique_elements` requires `std::regular` because it needs: copy (`push_back`), equality (`!=`), default ctor (vector element)
- Non-regular types are rejected at compile time with clear concept error messages
- The hierarchy lets you choose the **minimum** constraint needed - don't require `regular` if you only need `movable`

---

## Notes

- **Design your value types to be `std::regular` by default.** This is the C++ convention for well-behaved types. If you can't make a type regular, understand which level of the hierarchy it satisfies.
- **The "Rule of Regular":** If a type has `operator==`, it should be a true value equality (copies compare equal). Don't define `==` to mean something other than value equality.
- `std::semiregular` is the minimum for many standard library contexts (e.g., range adaptors, value types in containers).
- The standard ranges library uses `std::regular` for iterator sentinel types and `std::semiregular` for view closure objects.
- Adding `auto operator<=>(const T&) const = default;` to your class gives you all comparison operators, satisfying `equality_comparable` (and more).
