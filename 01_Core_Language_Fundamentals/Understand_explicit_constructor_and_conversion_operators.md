# Understand explicit constructor and conversion operators

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/explicit>  

---

## Topic Overview

The `explicit` keyword prevents a constructor or conversion operator from being used in **implicit conversions**. This is one of the most important tools for preventing subtle bugs in C++.

### The Problem: Implicit Conversions

Without `explicit`, the compiler is free to use a single-argument constructor as a silent conversion path. Here is a case where that goes wrong:

```cpp
struct Meters {
    double value;
    Meters(double v) : value(v) {}   // implicit converting constructor
};

void print_distance(Meters m) { std::cout << m.value << "m\n"; }

print_distance(3.14);    // Compiles! double -> Meters implicitly
// Is 3.14 meters? Or was this a bug where we forgot to wrap it?
```

The compiler happily converts the bare `3.14` into a `Meters` object. That may not be what you wanted, and the compiler gives no warning.

### The Fix: explicit

Mark the constructor `explicit` and the compiler stops doing that conversion for you:

```cpp
struct Meters {
    double value;
    explicit Meters(double v) : value(v) {}
};

// print_distance(3.14);           // ERROR: no implicit conversion
print_distance(Meters{3.14});      // OK: explicit construction
```

Now the intent is unambiguous in the code.

### explicit on Conversion Operators

The same keyword applies to conversion operators. Without `explicit`, a class that converts to `bool` can silently slide into arithmetic - a classic source of bugs with stream types and handle classes:

```cpp
struct FileHandle {
    int fd;
    explicit operator bool() const { return fd >= 0; }
};

FileHandle f{3};
if (f) { /* OK: contextual bool conversion allowed */ }
// bool b = f;       // ERROR: explicit prevents this
// int n = f;        // ERROR: no implicit conversion to int
// f + 1;            // ERROR: no implicit conversion chain
```

### Contextual Conversions (Where explicit bool Still Works)

Even with `explicit operator bool()`, these contexts allow the conversion because they have an unambiguous boolean meaning:

- `if (obj)`, `while (obj)`, `for(...; obj; ...)`
- `!obj`, `obj && x`, `obj || x`
- `obj ? a : b`
- `static_assert(obj)` (if constexpr)

So you get safety from accidental arithmetic while keeping all the natural boolean contexts.

### C++20: Conditional explicit - `explicit(bool)`

C++20 lets you make explicitness conditional on a compile-time expression. This is useful when a wrapper type should mirror the convertibility of whatever it wraps:

```cpp
template<typename T>
struct Wrapper {
    T value;

    // Implicit if T is implicitly constructible from U, explicit otherwise
    template<typename U>
    explicit(!std::is_convertible_v<U, T>) Wrapper(U&& u)
        : value(std::forward<U>(u)) {}
};

// If T=int, U=int: is_convertible -> true -> explicit(false) -> implicit OK
Wrapper<int> w1 = 42;         // OK

// If T=std::string, U=int: is_convertible -> false -> explicit(true)
// Wrapper<std::string> w2 = 42;  // ERROR
Wrapper<std::string> w3{std::string("hello")};  // OK: explicit construction
```

---

## Self-Assessment

### Q1: Show an implicit conversion bug fixed by adding explicit to a single-argument constructor

Here you can see exactly how an accidental integer gets silently converted into a buffer - and why making the constructor `explicit` forces the call site to be clear about its intent:

```cpp
#include <iostream>
#include <vector>

// === BUG VERSION ===
struct BadBuffer {
    std::vector<char> data;
    BadBuffer(int size) : data(size) {}   // implicit!
    int size() const { return static_cast<int>(data.size()); }
};

void process(BadBuffer buf) {
    std::cout << "Processing buffer of size " << buf.size() << "\n";
}

// === FIXED VERSION ===
struct GoodBuffer {
    std::vector<char> data;
    explicit GoodBuffer(int size) : data(size) {}   // explicit!
    int size() const { return static_cast<int>(data.size()); }
};

void process_good(GoodBuffer buf) {
    std::cout << "Processing buffer of size " << buf.size() << "\n";
}

int main() {
    // BUG: accidentally creating a 1024-byte buffer from a typo
    process(1024);          // Compiles! Creates BadBuffer with 1024 bytes
    process(0);             // Compiles! Creates empty BadBuffer -- was this intended?

    // The same mistakes with explicit are caught:
    // process_good(1024);  // ERROR: cannot convert int to GoodBuffer
    process_good(GoodBuffer{1024});   // OK: intent is clear
}
```

**How it works:**

- Without `explicit`, `process(1024)` silently creates a `BadBuffer` with 1024 bytes - this is almost certainly a bug.
- With `explicit`, the compiler forces you to write `GoodBuffer{1024}`, making the intent unambiguous.
- Real-world example: `std::vector<int>(5)` is explicit - you can't write `std::vector<int> v = 5;`.

### Q2: Write an explicit `operator bool()` for a resource handle and show how `if (handle)` still works

Watch how `explicit operator bool()` blocks all the silent conversions while leaving the `if`/`while`/`&&` contexts fully working:

```cpp
#include <iostream>

class DatabaseConnection {
    int connection_id;
public:
    explicit DatabaseConnection(int id) : connection_id(id) {}

    // Returns whether the connection is valid
    explicit operator bool() const noexcept {
        return connection_id > 0;
    }

    int id() const { return connection_id; }
};

int main() {
    DatabaseConnection valid{42};
    DatabaseConnection invalid{-1};

    // Contextual bool conversions work:
    if (valid) {
        std::cout << "Connection " << valid.id() << " is valid\n";
    }

    if (!invalid) {
        std::cout << "Connection " << invalid.id() << " is invalid\n";
    }

    // Logical operators work:
    bool both = valid && !invalid;   // OK
    std::cout << "Both conditions: " << both << "\n";

    // Ternary works:
    std::cout << (valid ? "connected" : "disconnected") << "\n";

    // These are blocked by explicit:
    // bool b = valid;           // ERROR
    // int n = valid;            // ERROR
    // valid + 1;                // ERROR
    // void* p = valid;          // ERROR (prevents bool -> int -> pointer chain)
}
```

**How it works:**

- `explicit operator bool()` prevents silent conversions to `bool`, `int`, or other types.
- C++ defines specific "contextual conversion" contexts where `explicit operator bool()` is still called.
- This is exactly how `std::optional`, `std::shared_ptr`, and stream objects (`if (cin >> x)`) work.

### Q3: Explain when `explicit(bool)` (C++20) is used to make explicitness conditional on a type property

**Answer:**

`explicit(condition)` makes a constructor or conversion operator `explicit` only when the compile-time `condition` is `true`. This is invaluable for **wrapper types** and **type-erased containers** that should mirror the explicitness of the wrapped type.

Here is a simplified `Optional` that uses this to propagate convertibility correctly:

```cpp
#include <iostream>
#include <type_traits>
#include <string>

template<typename T>
class Optional {
    alignas(T) unsigned char storage[sizeof(T)];
    bool has_value_ = false;

public:
    // Mirror T's construction explicitness:
    // If U->T is implicit, Optional(U) is implicit.
    // If U->T requires explicit, Optional(U) requires explicit.
    template<typename U>
    explicit(!std::is_convertible_v<U, T>)
    Optional(U&& val) : has_value_(true) {
        new (storage) T(std::forward<U>(val));
    }

    Optional() = default;
    bool has_value() const { return has_value_; }
    T& value() { return *reinterpret_cast<T*>(storage); }
    ~Optional() { if (has_value_) reinterpret_cast<T*>(storage)->~T(); }
};

int main() {
    // std::string is implicitly constructible from const char*
    Optional<std::string> s = "hello";   // OK: implicit
    std::cout << s.value() << "\n";

    // int is NOT implicitly constructible from std::string
    // Optional<int> n = std::string("42");  // ERROR: explicit needed

    // int IS implicitly constructible from short
    Optional<int> n = short{5};   // OK: implicit (short -> int is promotion)
    std::cout << n.value() << "\n";
}
```

**Real-world usage:**

- `std::pair` and `std::tuple` in C++20 use `explicit(bool)` so that `pair<int,int> p = {1,2}` works (implicit) but `pair<string, unique_ptr<int>> p = {1, nullptr}` doesn't (requires explicit construction).
- `std::optional<T>` uses it so `optional<int> o = 42;` works but `optional<explicit_type> o = value;` doesn't.

---

## Notes

- **Rule of thumb:** Always mark single-argument constructors `explicit` unless you intentionally want implicit conversions (e.g., `std::string` from `const char*`).
- `explicit` applies to constructors with **any** number of parameters since C++11 (prevents brace-init implicit conversions).
- `explicit(false)` is the same as no `explicit` - useful as the "else" branch of `explicit(condition)`.
- The standard library uses `explicit` extensively: `vector(size_t)`, `unique_ptr(T*)`, `shared_ptr(T*)`, `optional(T)` in some cases.
