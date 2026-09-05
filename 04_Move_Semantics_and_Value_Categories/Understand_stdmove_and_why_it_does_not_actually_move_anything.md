# Understand std::move and Why It Does Not Actually Move Anything

**Category:** Move Semantics and Value Categories  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/move>  

---

## Topic Overview

### What `std::move` Really Does

`std::move` is **not** a function that moves anything. It is an **unconditional cast** to an rvalue reference (`T&&`). The name is genuinely misleading - the reason this trips people up is that the name sounds like an action, but it's really just a label change that tells the compiler "you're allowed to steal from this." Here's the simplified implementation:

```cpp
// Simplified implementation of std::move:
template <typename T>
constexpr std::remove_reference_t<T>&& move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}
```

That's it - a `static_cast`. Nothing is transferred. The actual moving, if any, happens later when a move constructor or move assignment operator accepts that `T&&` and decides what to do with it.

| Misconception | Reality |
| --- | --- |
| `std::move` moves data | It only **casts** to `T&&` - a value category change |
| Movement always happens | Movement only happens if a move constructor/assignment accepts the `T&&` |
| `std::move` on `const` moves | `const T&&` binds to `const T&` -> **copy** instead |
| Moved-from object is destroyed | Object is in a **valid but unspecified** state |

### The Actual Move Chain

Here's the full chain so you can see where the cast ends and the real work begins:

```cpp
std::move(x)  ->  cast to T&&  ->  binds to T(T&&)  ->  move constructor steals resources
     ^                                  ^
  Just a cast              THIS does the actual moving
```

---

## Self-Assessment

### Q1: Show that `std::move` is just a cast to `T&&` and that the actual move happens in the constructor/assignment

This example traces what actually happens at each step so you can see that `std::move(a)` leaves `a` completely intact until something binds to the resulting reference.

```cpp
#include <iostream>
#include <string>
#include <utility>

class Tracker {
    std::string data_;
public:
    Tracker(std::string d) : data_(std::move(d)) {
        std::cout << "  Constructed with: \"" << data_ << "\"\n";
    }

    // THIS is where the actual move happens - not in std::move()
    Tracker(Tracker&& other) noexcept : data_(std::move(other.data_)) {
        std::cout << "  Move constructor called - stole data\n";
        std::cout << "  our data:   \"" << data_ << "\"\n";
        std::cout << "  source now: \"" << other.data_ << "\"\n";
    }

    Tracker(const Tracker& other) : data_(other.data_) {
        std::cout << "  Copy constructor called - copied data\n";
    }

    const std::string& data() const { return data_; }
};

int main() {
    std::cout << "=== std::move is just a cast ===\n\n";

    Tracker a("Hello, World!");

    // std::move(a) does NOT move anything - it's just static_cast<Tracker&&>(a)
    std::cout << "\nAfter std::move(a) but before using result:\n";
    Tracker&& ref = std::move(a);  // Just a cast - 'a' is untouched
    std::cout << "  a.data() is still: \"" << a.data() << "\"\n";
    std::cout << "  (nothing moved yet!)\n";

    // The ACTUAL move happens when the rvalue reference binds to a move constructor
    std::cout << "\nNow constructing b from that rvalue reference:\n";
    Tracker b(std::move(a));  // Move ctor called - THIS moves
    std::cout << "  a.data() after move: \"" << a.data() << "\"\n";
    std::cout << "  b.data() after move: \"" << b.data() << "\"\n";

    // Proof: std::move on a non-movable type just copies
    std::cout << "\n--- If there's no move ctor, it copies ---\n";
    const Tracker c("Immovable");
    std::cout << "Moving from const object:\n";
    Tracker d = std::move(c);  // const Tracker&& -> binds to const Tracker& -> COPY
    std::cout << "  c.data() still: \"" << c.data() << "\" (unchanged - was copied)\n";

    return 0;
}
// Expected output:
//   Constructed with: "Hello, World!"
//   After std::move(a) but before using result:
//     a.data() is still: "Hello, World!"
//     (nothing moved yet!)
//   Now constructing b from that rvalue reference:
//     Move constructor called - stole data
//     our data:   "Hello, World!"
//     source now: ""
//     a.data() after move: ""
//     b.data() after move: "Hello, World!"
//   --- If there's no move ctor, it copies ---
//   Constructed with: "Immovable"
//   Moving from const object:
//     Copy constructor called - copied data
//     c.data() still: "Immovable" (unchanged - was copied)
```

Notice the gap: the cast happens on one line, the resource theft happens on a completely different line when the move constructor runs. That gap is what the name `std::move` hides from you.

### Q2: Demonstrate a bug where `std::move` is applied to a `const` object and silently falls back to copy

This is a surprisingly common performance bug. The compiler won't warn you - it just quietly copies. Watch what happens when `const` meets `std::move`:

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <chrono>
#include <utility>

class ExpensiveData {
    std::vector<int> buffer_;
public:
    ExpensiveData() : buffer_(1'000'000, 42) {}

    // Move - cheap O(1)
    ExpensiveData(ExpensiveData&& other) noexcept
        : buffer_(std::move(other.buffer_)) {
        std::cout << "  [MOVE] O(1) - stole " << buffer_.size() << " elements\n";
    }

    // Copy - expensive O(n)
    ExpensiveData(const ExpensiveData& other)
        : buffer_(other.buffer_) {
        std::cout << "  [COPY] O(n) - copied " << buffer_.size() << " elements\n";
    }

    size_t size() const { return buffer_.size(); }
};

// BUG: taking by const reference then trying to move
class Container {
    ExpensiveData data_;
public:
    // BUGGY: const& parameter -> std::move produces const ExpensiveData&&
    //        -> binds to const ExpensiveData& -> COPY, not move!
    void set_buggy(const ExpensiveData& d) {
        data_ = std::move(d);  // BUG! Silently copies
        // std::move(d) -> const ExpensiveData&& -> binds to copy ctor
    }

    // FIXED version 1: Take by value and move
    void set_fixed_v1(ExpensiveData d) {
        data_ = std::move(d);  // d is non-const -> actual move
    }

    // FIXED version 2: Take by rvalue reference explicitly
    void set_fixed_v2(ExpensiveData&& d) {
        data_ = std::move(d);  // d is non-const T&& -> actual move
    }
};

int main() {
    std::cout << "=== Bug: std::move on const silently copies ===\n\n";

    Container c;
    ExpensiveData data;

    std::cout << "Buggy (const& + move = COPY):\n";
    c.set_buggy(data);

    std::cout << "\nFixed v1 (by value + move):\n";
    c.set_fixed_v1(std::move(data));

    ExpensiveData data2;
    std::cout << "\nFixed v2 (rvalue ref + move):\n";
    c.set_fixed_v2(std::move(data2));

    // Another common bug: storing const member and trying to move
    std::cout << "\n=== Another const-move trap ===\n";
    struct Wrapper {
        const std::string name;  // const member!
    };
    Wrapper w1{"Original"};
    Wrapper w2 = std::move(w1);  // const string -> COPIES, not moves
    std::cout << "w1.name: \"" << w1.name << "\" (unchanged - const member was copied)\n";
    std::cout << "w2.name: \"" << w2.name << "\"\n";

    return 0;
}
```

The `set_buggy` case is the one to internalize. You write `std::move(d)` expecting a move, but because `d` is `const`, the result is `const ExpensiveData&&`, and the only constructor that accepts a const rvalue reference is the copy constructor. You pay O(n) cost every time.

### Q3: Explain why using a moved-from object is not UB for standard types (but value is unspecified)

Contrary to the common myth, **using a moved-from standard library object is NOT undefined behavior**. The C++ standard guarantees that moved-from objects are in a **valid but unspecified state**. That means you can destroy them, reassign them, or call operations that have no preconditions on value - you just can't depend on what value they hold.

```cpp
#include <iostream>
#include <string>
#include <vector>

int main() {
    std::cout << "=== Moved-from Object State ===\n\n";

    // 1. Moved-from std::string
    std::string s = "Hello";
    std::string s2 = std::move(s);
    // s is now in a "valid but unspecified" state
    // These operations are ALL SAFE:
    std::cout << "Moved-from string:\n";
    std::cout << "  s.size()  = " << s.size() << "\n";    // Valid (likely 0)
    std::cout << "  s.empty() = " << s.empty() << "\n";   // Valid
    std::cout << "  s.data()  = \"" << s << "\"\n";        // Valid (likely "")
    s = "Reassigned";                                       // Valid
    std::cout << "  After reassign: \"" << s << "\"\n\n";

    // 2. Moved-from vector
    std::vector<int> v = {1, 2, 3, 4, 5};
    std::vector<int> v2 = std::move(v);
    std::cout << "Moved-from vector:\n";
    std::cout << "  v.size() = " << v.size() << "\n";     // Valid (likely 0)
    std::cout << "  v.empty() = " << v.empty() << "\n";   // Valid
    v.push_back(99);                                        // Valid
    std::cout << "  After push_back: size = " << v.size() << "\n\n";

    // 3. What IS safe with moved-from objects (standard guarantee):
    std::cout << "=== What's safe with moved-from objects ===\n";
    std::cout << "  SAFE:\n";
    std::cout << "    - Destruction\n";
    std::cout << "    - Assignment (copy or move to)\n";
    std::cout << "    - Operations with no preconditions (size, empty, clear)\n";
    std::cout << "  UNSAFE (value-dependent operations):\n";
    std::cout << "    - front(), back() on maybe-empty container\n";
    std::cout << "    - Dereferencing (if value was a pointer and is now null)\n";
    std::cout << "    - Relying on any specific value\n";

    // 4. User-defined types: YOU define the moved-from state
    std::cout << "\n=== User types: you control the contract ===\n";
    struct Buffer {
        int* data = nullptr;
        size_t size = 0;

        Buffer(size_t n) : data(new int[n]), size(n) {}
        Buffer(Buffer&& other) noexcept
            : data(std::exchange(other.data, nullptr))
            , size(std::exchange(other.size, 0)) {}
        ~Buffer() { delete[] data; }

        // After move: data==nullptr, size==0 - perfectly well-defined state
        bool empty() const { return size == 0; }
    };

    Buffer b1(100);
    Buffer b2(std::move(b1));
    std::cout << "  b1 after move: empty=" << b1.empty()
              << ", size=" << b1.size << "\n";

    return 0;
}
```

For your own types, you get to define what "valid but unspecified" means - whatever state makes the destructor safe to call. The `Buffer` example above shows the idiom: `std::exchange` sets both members to safe sentinel values in one clean step.

---

## Notes

- `std::move` is a cast, not an operation - it produces an xvalue that **enables** moving.
- Never `std::move` from `const` objects - the compiler silently falls back to copying.
- Moved-from standard types are valid but unspecified - safe to destroy or reassign.
- Mark move constructors/assignments `noexcept` - this enables `std::vector` to use moves during reallocation.
- Common pitfall: `std::move` in `return` statements can **prevent** NRVO (see separate topic).
