# Know implicit vs explicit default, copy, and move member generation rules

**Category:** Core Language Fundamentals  
**Standard:** C++11, C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/default_constructor>  

---

## Topic Overview

### The Six Special Member Functions

Every class has six functions the compiler might write for you behind the scenes. Knowing *which* ones it writes - and when it quietly stops writing them - is one of those things that separates "my class works by accident" from "my class works because I understand it":

| # | Function | Signature |
| --- | --- | --- |
| 1 | Default constructor | `T()` |
| 2 | Destructor | `~T()` |
| 3 | Copy constructor | `T(const T&)` |
| 4 | Copy assignment | `T& operator=(const T&)` |
| 5 | Move constructor | `T(T&&)` |
| 6 | Move assignment | `T& operator=(T&&)` |

The compiler can **implicitly generate** any of these. The catch is that declaring one of them yourself can switch off the automatic generation of others, in ways that aren't always obvious.

### The Generation Rules Table (The "Rule of Zero/Three/Five" Foundation)

This table is the whole topic in one grid. Read it as "if you declare the thing on the left, here's what you still get for free":

| You declare... | Default ctor | Destructor | Copy ctor | Copy assign | Move ctor | Move assign |
| --- | --- | --- | --- | --- | --- | --- |
| Nothing | implicit | implicit | implicit | implicit | implicit | implicit |
| Any constructor | suppressed | implicit | implicit | implicit | implicit | implicit |
| Destructor | implicit | user | implicit* | implicit* | suppressed | suppressed |
| Copy constructor | suppressed | implicit | user | implicit* | suppressed | suppressed |
| Copy assignment | implicit | implicit | implicit* | user | suppressed | suppressed |
| Move constructor | suppressed | implicit | =delete | =delete | user | suppressed |
| Move assignment | implicit | implicit | implicit* | implicit* | suppressed | user |

`*` = generation that's *deprecated* (since C++11) - it still happens, but the committee treats it as a defect, so spell it out with `= default`.

### Key Rules

If the table feels like a lot, it boils down to six rules you can actually remember:

1. **Declaring any constructor** (even a parameterized one) suppresses the implicit default constructor.
2. **Declaring a destructor** suppresses the implicit move operations.
3. **Declaring a copy operation** suppresses the implicit move operations.
4. **Declaring a move operation** suppresses the implicit copy operations (they become `= delete`).
5. **`= default`** asks for the compiler-generated version explicitly.
6. **`= delete`** turns a function off explicitly.

### The Rule of Zero

The happiest path: if your class doesn't directly own a raw resource, declare *none* of the six and let the members manage themselves:

```cpp
struct Person {
    std::string name;
    int age;
    // Rule of Zero: compiler generates all 6 correctly
    // because std::string and int handle themselves
};
```

`std::string` already knows how to copy, move, and destroy itself - so by declaring nothing, you inherit all of that correctness for free. This is the design you should reach for by default.

### The Rule of Five

But the moment you *do* manage a resource by hand - and so need a destructor, copy constructor, or copy assignment - you've signed up to write **all five**. Half-doing it is how you get double-frees and silent copies:

```cpp
class Buffer {
    int* data_;
    std::size_t size_;
public:
    Buffer(std::size_t n) : data_(new int[n]), size_(n) {}

    ~Buffer()                          { delete[] data_; }
    Buffer(const Buffer& o)            : data_(new int[o.size_]), size_(o.size_) {
        std::copy(o.data_, o.data_ + size_, data_);
    }
    Buffer& operator=(const Buffer& o) {
        if (this != &o) { Buffer tmp(o); swap(*this, tmp); }
        return *this;
    }
    Buffer(Buffer&& o) noexcept        : data_(o.data_), size_(o.size_) {
        o.data_ = nullptr; o.size_ = 0;
    }
    Buffer& operator=(Buffer&& o) noexcept {
        if (this != &o) { delete[] data_; data_ = o.data_; size_ = o.size_;
            o.data_ = nullptr; o.size_ = 0; }
        return *this;
    }

    friend void swap(Buffer& a, Buffer& b) noexcept {
        std::swap(a.data_, b.data_);
        std::swap(a.size_, b.size_);
    }
};
```

---

## Self-Assessment

### Q1: Show that declaring any constructor suppresses the implicit default constructor

The lesson here is rule #1 in action - and its escape hatch. `Widget` declares a constructor and loses its default one; `Restored` shows you how to get it back with `= default`:

```cpp
#include <iostream>
#include <string>

struct Widget {
    int value;
    std::string name;

    // Declaring this parameterized constructor suppresses the default constructor
    Widget(int v) : value(v), name("unnamed") {
        std::cout << "Parameterized ctor: " << value << "\n";
    }
};

struct Gadget {
    int value;
    std::string name;
    // No constructors declared -> default constructor is implicitly generated
};

struct Restored {
    int value;
    std::string name;

    Restored(int v) : value(v), name("unnamed") {}

    // Explicitly bring back the default constructor
    Restored() = default;  // <-- restores it!
};

int main() {
    // Gadget g;      // OK: implicit default constructor
    // Widget w;      // ERROR: no default constructor!
    //                // error: no matching function for call to 'Widget::Widget()'

    Widget w(42);      // OK: uses the declared constructor
    std::cout << "w = " << w.value << "\n";

    Restored r;        // OK: = default brings it back
    std::cout << "r.value = " << r.value << "\n"; // indeterminate (default-initialized)

    Restored r2(10);   // Also OK
    std::cout << "r2.value = " << r2.value << "\n"; // 10

    return 0;
}
```

**How this works:**

- `Widget` declares `Widget(int)`, so the compiler stops generating `Widget()`.
- Writing `Widget w;` is therefore a hard compile error.
- `Restored` also declares `Restored(int)`, but adds `Restored() = default;` to bring the default constructor back.
- This applies to *any* user-declared constructor - copy, move, conversion - they all suppress the default one.

### Q2: Demonstrate that declaring a destructor suppresses implicit move constructor generation

This is the single most common gotcha in the whole topic, and it's nasty because it doesn't break compilation - it silently turns your moves into copies. Watch what happens to `WithDestructor`:

```cpp
#include <iostream>
#include <string>
#include <type_traits>

struct NoDestructor {
    std::string data;
    // No user-declared destructor -> move operations are implicitly generated
};

struct WithDestructor {
    std::string data;
    ~WithDestructor() { /* user-declared destructor */ }
    // Move constructor: NOT generated (suppressed)
    // Move assignment: NOT generated (suppressed)
    // Copy constructor: still generated (but deprecated!)
    // Copy assignment: still generated (but deprecated!)
};

struct Fixed {
    std::string data;
    ~Fixed() = default;
    // Even "= default" counts as user-declared!
    // Move operations are still suppressed!
    // But since it's trivial, some compilers may optimize anyway
};

struct ProperlyFixed {
    std::string data;
    ~ProperlyFixed() = default;
    ProperlyFixed() = default;
    ProperlyFixed(const ProperlyFixed&) = default;
    ProperlyFixed& operator=(const ProperlyFixed&) = default;
    ProperlyFixed(ProperlyFixed&&) = default;            // Explicitly requested
    ProperlyFixed& operator=(ProperlyFixed&&) = default; // Explicitly requested
};

int main() {
    // Check move constructibility:
    std::cout << "NoDestructor move constructible: "
              << std::is_move_constructible_v<NoDestructor> << "\n";     // 1
    std::cout << "WithDestructor move constructible: "
              << std::is_move_constructible_v<WithDestructor> << "\n";   // 1 (falls back to copy!)
    std::cout << "WithDestructor nothrow move constructible: "
              << std::is_nothrow_move_constructible_v<WithDestructor> << "\n"; // 0! (copy might throw)

    // The difference: WithDestructor "moves" by copying (expensive!)
    NoDestructor nd;
    nd.data = std::string(1000, 'x');
    NoDestructor nd2 = std::move(nd);
    std::cout << "After move, nd.data.size() = " << nd.data.size() << "\n";  // 0 (moved from)

    WithDestructor wd;
    wd.data = std::string(1000, 'x');
    WithDestructor wd2 = std::move(wd);
    std::cout << "After 'move', wd.data.size() = " << wd.data.size() << "\n"; // 1000! (copied, not moved!)

    return 0;
}
```

**Output:**

```text
NoDestructor move constructible: 1
WithDestructor move constructible: 1
WithDestructor nothrow move constructible: 0
After move, nd.data.size() = 0
After 'move', wd.data.size() = 1000
```

**How this works:**

- `NoDestructor` gets all six members implicitly, so `std::move` genuinely moves the string and leaves the source empty (size 0).
- `WithDestructor` has a user-declared destructor, which suppresses the move constructor. `std::move(wd)` still produces an rvalue, but with no move constructor available, overload resolution falls back to the **copy constructor** (`const T&` happily binds an rvalue). The string is *copied*.
- Notice the tell: `is_move_constructible_v` is still `1` (because copy can stand in for move), but `is_nothrow_move_constructible_v` is `0`. That gap is your warning sign.
- And yes - even `~T() = default;` counts as "user-declared" and suppresses the moves.

### Q3: Use `= default` and `= delete` to control generation

Once you understand suppression, `= default` and `= delete` let you dial in *exactly* which operations a type supports. Three common shapes - move-only, copy-only, and neither:

```cpp
#include <iostream>
#include <string>

// Example 1: Non-copyable, movable resource handle
class FileHandle {
    int fd_;
public:
    explicit FileHandle(int fd) : fd_(fd) {}
    ~FileHandle() { if (fd_ >= 0) { /* close(fd_); */ } }

    // Delete copy operations - file handles shouldn't be duplicated
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    // Explicitly default move operations
    FileHandle(FileHandle&& other) noexcept : fd_(other.fd_) { other.fd_ = -1; }
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) { fd_ = other.fd_; other.fd_ = -1; }
        return *this;
    }
};

// Example 2: Copyable, non-movable (rare but valid)
class SharedState {
    std::string data_;
public:
    SharedState(std::string d) : data_(std::move(d)) {}
    SharedState(const SharedState&) = default;
    SharedState& operator=(const SharedState&) = default;
    SharedState(SharedState&&) = delete;            // Explicitly non-movable
    SharedState& operator=(SharedState&&) = delete;
};

// Example 3: Neither copyable nor movable (singleton pattern)
class Singleton {
public:
    static Singleton& instance() {
        static Singleton s;
        return s;
    }

    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
    Singleton(Singleton&&) = delete;
    Singleton& operator=(Singleton&&) = delete;

private:
    Singleton() = default;
    ~Singleton() = default;
};

// Resulting special member table:
//
// | Class       | Default | Dtor    | Copy ctor | Copy =  | Move ctor | Move =  |
// |-------------|---------|---------|-----------|---------|-----------|---------|
// | FileHandle  | (none)  | user    | =delete   | =delete | user      | user    |
// | SharedState | (none)  | default | =default  | =default| =delete   | =delete |
// | Singleton   | =default| =default| =delete   | =delete | =delete   | =delete |

int main() {
    FileHandle fh(42);
    // FileHandle fh2 = fh;              // ERROR: copy deleted
    FileHandle fh2 = std::move(fh);      // OK: move is defined

    SharedState ss("hello");
    SharedState ss2 = ss;                 // OK: copy works
    // SharedState ss3 = std::move(ss);   // ERROR: move deleted

    Singleton& s = Singleton::instance();
    // Singleton s2 = s;                  // ERROR: copy deleted
    // Singleton s3 = std::move(s);       // ERROR: move deleted

    return 0;
}
```

**How this works:**

- `= default` says "give me the compiler's version of this function."
- `= delete` says "this function exists for overload resolution, but calling it is an error."
- That distinction matters: a deleted function still *participates* in overload resolution, so it can be selected and then rejected - giving you a clear, intentional error rather than silently falling through to something else.
- Mixing `= default` and `= delete` is how you express precisely "this type is move-only" or "this type is a singleton."

---

## Additional Examples

### Detecting Suppressed Moves with Type Traits

The reliable way to *catch* an accidentally-suppressed move is `std::is_nothrow_move_constructible_v` - a `false` here usually means "your move silently became a copy":

```cpp
#include <type_traits>
#include <iostream>
#include <string>

struct A { std::string s; };                    // All defaults
struct B { std::string s; ~B() {} };            // Destructor suppresses moves
struct C { std::string s; C(const C&) = default; C& operator=(const C&) = default; }; // Copy suppresses moves
struct D { std::string s; ~D() = default;
           D() = default; D(const D&) = default; D& operator=(const D&) = default;
           D(D&&) = default; D& operator=(D&&) = default; }; // All explicit

int main() {
    std::cout << std::boolalpha;
    std::cout << "A nothrow_move: " << std::is_nothrow_move_constructible_v<A> << "\n"; // true
    std::cout << "B nothrow_move: " << std::is_nothrow_move_constructible_v<B> << "\n"; // false (copies!)
    std::cout << "C nothrow_move: " << std::is_nothrow_move_constructible_v<C> << "\n"; // false (copies!)
    std::cout << "D nothrow_move: " << std::is_nothrow_move_constructible_v<D> << "\n"; // true
}
```

---

## Notes

- **Rule of Zero:** if you don't manage resources directly, declare nothing - the compiler gets all six right.
- **Rule of Five:** if you declare a destructor, copy constructor, or copy assignment, declare all five.
- Declaring a destructor - *even `= default`* - suppresses the move operations. This is the most common gotcha by far.
- Use `std::is_nothrow_move_constructible_v` to confirm your moves weren't silently downgraded to copies.
- `= delete` participates in overload resolution - a deleted function beats "no function," which is what gives you a clean, pointed error.
