# Know That `std::initializer_list` Always Copies and Never Moves Elements

**Category:** Move Semantics and Value Categories  
**Standard:** C++11 and later  
**Reference:** https://en.cppreference.com/w/cpp/utility/initializer_list  

---

## Topic Overview

`std::initializer_list<T>` is backed by a hidden `const T[N]` array. The keyword here is **`const`** - the elements are immutable. This means that even if you `std::move` the `initializer_list` itself, the elements inside cannot be moved from because they are `const`. Every element extraction from an `initializer_list` results in a **copy**, never a move.

This is one of those things that feels like it should work but silently doesn't. The reason it trips people up is that `std::move` on a `const` object produces a `const T&&` - which the overload resolution rules send to the **copy constructor**, not the move constructor. So your attempted move becomes a copy, with no compiler warning.

Here is the mental model for what the list actually looks like in memory:

```cpp
┌──────────────────────────────────────────────────────────────────┐
│              std::initializer_list Internal Model                 │
│                                                                  │
│   initializer_list<string> il = {"alpha", "beta", "gamma"};     │
│                                                                  │
│   Stack/Static Storage:                                          │
│   ┌─────────────────────────────────────────────────┐            │
│   │  const string[3] backing_array = {              │            │
│   │      "alpha",  // const! cannot move from       │            │
│   │      "beta",   // const! cannot move from       │            │
│   │      "gamma"   // const! cannot move from       │            │
│   │  };                                             │            │
│   └─────────────────────────────────────────────────┘            │
│                     ^             ^                               │
│   initializer_list: │ begin()     │ end()                        │
│   (pointer + size)  └─────────────┘                              │
│   Copying this only copies the pointer+size, not the array.     │
└──────────────────────────────────────────────────────────────────┘
```

This has significant **performance implications**. If you construct a container of move-only or expensive-to-copy types using brace initialization, each element will be copied, not moved. For `std::vector<std::string>`, this means every string is copied even when the source is a temporary string literal that could have been moved. For `std::vector<std::unique_ptr<T>>`, brace initialization is simply **impossible** - `unique_ptr` is not copyable.

Here is how the two approaches compare:

```cpp
┌────────────────────────────────────────────────────────────────┐
│            Performance Comparison: init-list vs reserve+emplace│
├────────────────────────┬───────────────────────────────────────┤
│  Approach              │  Copies / Moves per element           │
├────────────────────────┼───────────────────────────────────────┤
│  vector<string> v =    │  1 copy per element (from const arr)  │
│    {"a", "b", "c"};    │  + possible reallocation copies       │
│                        │                                       │
│  v.reserve(3);         │  0 copies, 1 move per element         │
│  v.emplace_back("a");  │  (or even 0 moves with in-place       │
│  v.emplace_back("b");  │   construction)                       │
│  v.emplace_back("c");  │                                       │
└────────────────────────┴───────────────────────────────────────┘
```

The workaround is to avoid `initializer_list` when move semantics matter. Use `reserve` + `emplace_back`, or use variadic templates, or construct a `std::array` and move from it. Some libraries provide `make_vector`-style helpers that use parameter packs instead of `initializer_list`.

---

## Self-Assessment

### Q1: Why does `std::initializer_list` force copies, and what does `std::move` on the list actually do

Let's make the copies visible. The `Verbose` type below prints every construction so you can see exactly what happens when you try to move from an `initializer_list`.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <initializer_list>
#include <utility>

struct Verbose {
    std::string name;

    Verbose(const char* n) : name(n) {
        std::cout << "  construct: " << name << "\n";
    }
    Verbose(const Verbose& o) : name(o.name) {
        std::cout << "  COPY: " << name << "\n";
    }
    Verbose(Verbose&& o) noexcept : name(std::move(o.name)) {
        std::cout << "  MOVE: " << name << "\n";
    }
    Verbose& operator=(const Verbose&) = default;
    Verbose& operator=(Verbose&&) = default;
    ~Verbose() = default;
};

void take_by_init_list(std::initializer_list<Verbose> il) {
    std::cout << "  --- inserting from initializer_list ---\n";
    std::vector<Verbose> v;
    v.reserve(il.size());
    for (const auto& elem : il) {
        // elem is const Verbose& - we MUST copy
        v.push_back(elem);
    }
}

void take_by_init_list_attempt_move(std::initializer_list<Verbose> il) {
    std::cout << "  --- attempting to move from initializer_list ---\n";
    std::vector<Verbose> v;
    v.reserve(il.size());
    for (auto& elem : il) {
        // elem is const Verbose& (even without 'const' in the loop,
        // because the underlying array is const T[])
        // std::move(elem) produces const Verbose&&
        // which matches the copy constructor, NOT move constructor!
        v.push_back(std::move(elem));  // STILL COPIES!
    }
}

int main() {
    std::cout << "=== Passing initializer_list ===\n";
    take_by_init_list({"Alice", "Bob"});

    std::cout << "\n=== Attempting std::move on elements ===\n";
    take_by_init_list_attempt_move({"Charlie", "Diana"});

    // Even moving the initializer_list itself doesn't help:
    std::cout << "\n=== Moving the list object itself ===\n";
    std::initializer_list<Verbose> il = {"Eve", "Frank"};
    auto il2 = std::move(il);
    // il2 just copies the pointer+size - elements are still const
    std::cout << "  il2.size() = " << il2.size() << "\n";

    return 0;
}
```

Watch the output carefully: both functions print COPY, never MOVE. Moving the list object itself (`std::move(il)`) only moves the pointer and size - the backing `const` array stays put, and its elements remain un-moveable.

### Q2: How do you efficiently initialize a `vector` of expensive-to-copy types without `initializer_list`

Once you know that `initializer_list` always copies, the fix is straightforward: bypass it entirely. Here are three alternatives, all of which allow moves or in-place construction.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <utility>
#include <array>

struct HeavyObject {
    std::string data;
    HeavyObject(std::string d) : data(std::move(d)) {
        std::cout << "  construct: " << data << "\n";
    }
    HeavyObject(const HeavyObject& o) : data(o.data) {
        std::cout << "  COPY: " << data << "\n";
    }
    HeavyObject(HeavyObject&& o) noexcept : data(std::move(o.data)) {
        std::cout << "  MOVE: " << data << "\n";
    }
    HeavyObject& operator=(HeavyObject&&) = default;
    HeavyObject& operator=(const HeavyObject&) = default;
};

// Alternative 1: reserve + emplace_back
std::vector<HeavyObject> make_vec_emplace() {
    std::cout << "--- reserve + emplace_back ---\n";
    std::vector<HeavyObject> v;
    v.reserve(3);
    v.emplace_back("X");  // constructs in place, no copy or move
    v.emplace_back("Y");
    v.emplace_back("Z");
    return v;  // NRVO, no copy
}

// Alternative 2: variadic template helper
template<typename T, typename... Args>
std::vector<T> make_vector(Args&&... args) {
    std::vector<T> v;
    v.reserve(sizeof...(args));
    (v.emplace_back(std::forward<Args>(args)), ...);
    return v;
}

// Alternative 3: std::array + move (for fixed-size init)
std::vector<HeavyObject> make_vec_from_array() {
    std::cout << "--- from std::array with move ---\n";
    std::array<HeavyObject, 3> arr = {
        HeavyObject{"A"}, HeavyObject{"B"}, HeavyObject{"C"}
    };
    std::vector<HeavyObject> v;
    v.reserve(arr.size());
    for (auto& elem : arr) {
        v.push_back(std::move(elem));  // arr is NOT const, so move works
    }
    return v;
}

int main() {
    std::cout << "=== Method 1: emplace_back ===\n";
    auto v1 = make_vec_emplace();

    std::cout << "\n=== Method 2: variadic make_vector ===\n";
    auto v2 = make_vector<HeavyObject>("P", "Q", "R");

    std::cout << "\n=== Method 3: array + move ===\n";
    auto v3 = make_vec_from_array();

    // For move-only types, initializer_list is completely unusable:
    std::cout << "\n=== unique_ptr vector (init_list impossible) ===\n";
    // std::vector<std::unique_ptr<int>> bad = {
    //     std::make_unique<int>(1),
    //     std::make_unique<int>(2)
    // };  // ERROR: unique_ptr is not copyable

    // Must use emplace_back or push_back with std::move:
    std::vector<std::unique_ptr<int>> good;
    good.push_back(std::make_unique<int>(1));
    good.push_back(std::make_unique<int>(2));
    std::cout << "  good.size() = " << good.size() << "\n";

    return 0;
}
```

The `std::array` approach is worth noting: `std::array` is not `const`, so iterating over it with `auto&` gives you a mutable reference and `std::move` actually moves. That is the fundamental difference from `initializer_list`.

### Q3: Can you detect the performance difference between `initializer_list` construction and `emplace_back`

This benchmark makes the cost visible in wall-clock time. The `BigString` type is cheap to move and expensive to copy, which exaggerates the difference.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <chrono>

// A simplified "large string" type to make copies expensive
struct BigString {
    std::string data;

    BigString(const char* s) : data(s) {}
    BigString(std::string s) : data(std::move(s)) {}

    // Expensive copy
    BigString(const BigString& o) : data(o.data) {}

    // Cheap move
    BigString(BigString&& o) noexcept : data(std::move(o.data)) {}

    BigString& operator=(const BigString&) = default;
    BigString& operator=(BigString&&) noexcept = default;
};

int main() {
    const int N = 100'000;
    const std::string sample(1000, 'x');  // 1KB string

    // Method 1: initializer_list (forces copies)
    {
        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) {
            // Each element is copied from the const backing array
            std::vector<BigString> v = {BigString{sample}};
            (void)v;
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "initializer_list (copy): " << ms.count() << " ms\n";
    }

    // Method 2: emplace_back (avoids copies)
    {
        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) {
            std::vector<BigString> v;
            v.reserve(1);
            v.emplace_back(sample);  // constructs in place
            (void)v;
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << "emplace_back (no copy):  " << ms.count() << " ms\n";
    }

    return 0;
}
```

---

## Notes

- `std::initializer_list<T>` is backed by a hidden `const T[N]` array - the `const` is the reason moves are impossible.
- `std::move` on an `initializer_list` only copies the internal pointer and size - it does NOT move the elements.
- Attempting `std::move(*it)` on an `initializer_list` iterator yields `const T&&`, which binds to the **copy** constructor, not the move constructor. This is a silent performance trap.
- Move-only types like `std::unique_ptr` cannot be used with `initializer_list` at all - it won't compile.
- Prefer `reserve()` + `emplace_back()` or a variadic template helper when move semantics matter.
- This is a known deficiency in the language. Proposals to fix it (like `std::initializer_list` with move semantics) have been discussed but not adopted as of C++26.
