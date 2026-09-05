# Understand `aligned_storage`, `aligned_union`, and Their C++23 Deprecation

**Category:** Memory and Ownership  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/types/aligned_storage>  

---

## Topic Overview

### What Were `aligned_storage` and `aligned_union`

Before C++23, if you needed a chunk of raw memory big enough and correctly aligned to hold a `T` without constructing it yet, the standard answer was to reach for these type traits:

```cpp
// C++11-C++20 way to create properly aligned raw storage
std::aligned_storage_t<sizeof(T), alignof(T)> buffer;

// Aligned storage large enough for any of the listed types
std::aligned_union_t<0, int, double, std::string> variant_buf;
```

These provided a **type** with the right `sizeof` and `alignof` for storing objects without constructing them - useful for:

- Type-erased buffers
- Manual variant/optional implementations
- Placement new patterns

### Why Deprecated in C++23

The reason this trips people up is that `aligned_storage_t` is not `T` - it is a completely unrelated type that happens to have the same size and alignment. Every access through a reinterpret-cast technically risks strict aliasing UB. C++23 deprecated both utilities because a much simpler alternative already existed (C++17's `std::byte` and `alignas`).

| Problem | Detail |
| --- | --- |
| **Wrong type** | `aligned_storage_t` is an unrelated type, not `T` - type aliasing issues |
| **Violates strict aliasing** | Accessing through wrong type is technically UB |
| **Overly complex** | `alignas` + `std::byte[]` is simpler and correct |
| **Brittle** | Easy to get size/alignment wrong with the template parameters |

### The Modern Replacement

Here is the before/after in concrete code. The new form reads almost like English - "give me a `byte` array of the right size, aligned for `T`."

```cpp
// Old (deprecated C++23):
std::aligned_storage_t<sizeof(T), alignof(T)> buffer;
new (&buffer) T(args...);

// New (correct):
alignas(T) std::byte buffer[sizeof(T)];
auto* p = std::construct_at(reinterpret_cast<T*>(buffer), args...);
std::destroy_at(p);
```

---

## Self-Assessment

### Q1: Implement a type-erased buffer using `alignas` and `std::byte` instead of `aligned_storage`

This is the pattern that replaces the old `aligned_storage` idiom. Watch how the storage is just a plain `byte` array, and the type-erased destroy/copy operations are stored separately as function pointers.

```cpp
#include <iostream>
#include <cstddef>
#include <memory>
#include <string>
#include <new>
#include <functional>

// Type-erased buffer: can hold any type up to MAX_SIZE bytes
template<std::size_t MAX_SIZE = 64, std::size_t MAX_ALIGN = alignof(std::max_align_t)>
class SmallBuffer {
    alignas(MAX_ALIGN) std::byte storage_[MAX_SIZE];

    // Type-erased operations
    void (*destroy_fn_)(void*) = nullptr;
    void (*copy_fn_)(void* dst, const void* src) = nullptr;

    void clear() {
        if (destroy_fn_) {
            destroy_fn_(storage_);
            destroy_fn_ = nullptr;
            copy_fn_ = nullptr;
        }
    }

public:
    SmallBuffer() = default;

    ~SmallBuffer() { clear(); }

    // Store any type T that fits
    template<typename T, typename... Args>
    T& emplace(Args&&... args) {
        static_assert(sizeof(T) <= MAX_SIZE, "Type too large for buffer");
        static_assert(alignof(T) <= MAX_ALIGN, "Type alignment too strict for buffer");

        clear();

        T* obj = std::construct_at(
            reinterpret_cast<T*>(storage_),
            std::forward<Args>(args)...
        );

        destroy_fn_ = [](void* p) { std::destroy_at(static_cast<T*>(p)); };

        return *obj;
    }

    // Get the stored object (caller must know the type)
    template<typename T>
    T& get() {
        return *reinterpret_cast<T*>(storage_);
    }

    template<typename T>
    const T& get() const {
        return *reinterpret_cast<const T*>(storage_);
    }

    bool has_value() const { return destroy_fn_ != nullptr; }
};

int main() {
    SmallBuffer<128> buf;

    // Store an int
    buf.emplace<int>(42);
    std::cout << "int: " << buf.get<int>() << "\n";

    // Replace with a string
    buf.emplace<std::string>("Hello, type-erased world!");
    std::cout << "string: " << buf.get<std::string>() << "\n";

    // Replace with a double
    buf.emplace<double>(3.14159);
    std::cout << "double: " << buf.get<double>() << "\n";

    // Size check
    std::cout << "\nsizeof(SmallBuffer<128>): " << sizeof(buf) << " bytes\n";

    return 0;
}
```

**Output:**

```text
int: 42
string: Hello, type-erased world!
double: 3.14159

sizeof(SmallBuffer<128>): 144 bytes
```

The 144 bytes come from the 128-byte storage plus two function pointers. Each `emplace` properly destroys the previous value before constructing the new one.

### Q2: Explain why `aligned_storage` was deprecated in C++23 and what replaces it

The comparison below makes the difference concrete. The old `aligned_storage` approach requires you to spell the size and alignment as template parameters, which can drift out of sync if `T` changes. The new approach derives both from `T` directly.

```cpp
#include <iostream>
#include <cstddef>
#include <string>
#include <memory>
#include <type_traits>

// ======== OLD WAY (deprecated in C++23) ========
// Problems with aligned_storage:

// 1. It's a DIFFERENT TYPE than what you store
//    aligned_storage_t<sizeof(int), alignof(int)> is NOT int
//    Accessing via reinterpret_cast may violate strict aliasing

// 2. The template parameters are error-prone
//    std::aligned_storage_t<sizeof(T), alignof(T)>  // easy to swap size/align
//    What if T changes size? Parameter doesn't auto-update.

// 3. aligned_union has the same problems, plus complexity
//    std::aligned_union_t<0, int, double, std::string>

// ======== NEW WAY (C++17/C++20/C++23) ========
// Simply use alignas + std::byte array
// + std::construct_at / std::destroy_at

template<typename T>
class ManualOptional {
    // NEW: direct, correct, simple
    alignas(T) std::byte storage_[sizeof(T)];
    bool engaged_ = false;

public:
    ManualOptional() = default;

    template<typename... Args>
    void emplace(Args&&... args) {
        reset();
        std::construct_at(reinterpret_cast<T*>(storage_),
                          std::forward<Args>(args)...);
        engaged_ = true;
    }

    void reset() {
        if (engaged_) {
            std::destroy_at(reinterpret_cast<T*>(storage_));
            engaged_ = false;
        }
    }

    T& value() { return *reinterpret_cast<T*>(storage_); }
    const T& value() const { return *reinterpret_cast<const T*>(storage_); }
    bool has_value() const { return engaged_; }

    ~ManualOptional() { reset(); }
};

int main() {
    std::cout << "=== Comparison ===\n\n";

    // Old way (deprecated):
    // std::aligned_storage_t<sizeof(std::string), alignof(std::string)> buf;
    // new (&buf) std::string("old");
    // reinterpret_cast<std::string*>(&buf)->~basic_string();

    // New way:
    ManualOptional<std::string> opt;
    opt.emplace("Modern C++23 approach");
    std::cout << "Value: " << opt.value() << "\n";
    std::cout << "Has value: " << opt.has_value() << "\n";
    opt.reset();
    std::cout << "After reset: " << opt.has_value() << "\n";

    // Works with any type
    ManualOptional<int> opt_int;
    opt_int.emplace(42);
    std::cout << "\nint value: " << opt_int.value() << "\n";

    std::cout << "\n=== Why the change ===\n";
    std::cout << "aligned_storage_t<S,A>: opaque type, strict aliasing risk\n";
    std::cout << "alignas(T) byte[sizeof(T)]: explicit, correct, simple\n";

    return 0;
}
```

### Q3: Show a variant-like storage that manually manages construction/destruction in aligned storage

This is the canonical example of why this feature exists - a discriminated union that stores different types in the same piece of memory. Notice how `destroy` only needs special work for types with non-trivial destructors like `std::string`; trivial types like `int` and `double` can be left alone.

```cpp
#include <iostream>
#include <cstddef>
#include <string>
#include <memory>
#include <algorithm>
#include <utility>
#include <new>
#include <cassert>

// A simple variant that can hold int, double, or std::string
class SimpleVariant {
    // Storage: large enough and aligned for any of our types
    static constexpr size_t max_size =
        std::max({sizeof(int), sizeof(double), sizeof(std::string)});
    static constexpr size_t max_align =
        std::max({alignof(int), alignof(double), alignof(std::string)});

    alignas(max_align) std::byte storage_[max_size];

    enum class Type { None, Int, Double, String } type_ = Type::None;

    void destroy() {
        switch (type_) {
            case Type::String:
                std::destroy_at(reinterpret_cast<std::string*>(storage_));
                break;
            default: break;  // trivial types need no destruction
        }
        type_ = Type::None;
    }

public:
    SimpleVariant() = default;
    ~SimpleVariant() { destroy(); }

    // No copy for simplicity (real variant would need it)
    SimpleVariant(const SimpleVariant&) = delete;
    SimpleVariant& operator=(const SimpleVariant&) = delete;

    // Set int
    void set(int v) {
        destroy();
        std::construct_at(reinterpret_cast<int*>(storage_), v);
        type_ = Type::Int;
    }

    // Set double
    void set(double v) {
        destroy();
        std::construct_at(reinterpret_cast<double*>(storage_), v);
        type_ = Type::Double;
    }

    // Set string
    void set(const std::string& v) {
        destroy();
        std::construct_at(reinterpret_cast<std::string*>(storage_), v);
        type_ = Type::String;
    }

    // Get
    template<typename T> T& get();

    // Visit
    void print() const {
        switch (type_) {
            case Type::Int:
                std::cout << "int: " << *reinterpret_cast<const int*>(storage_) << "\n";
                break;
            case Type::Double:
                std::cout << "double: " << *reinterpret_cast<const double*>(storage_) << "\n";
                break;
            case Type::String:
                std::cout << "string: \"" << *reinterpret_cast<const std::string*>(storage_) << "\"\n";
                break;
            case Type::None:
                std::cout << "(empty)\n";
                break;
        }
    }

    const char* type_name() const {
        switch (type_) {
            case Type::Int: return "int";
            case Type::Double: return "double";
            case Type::String: return "string";
            default: return "none";
        }
    }
};

template<> int& SimpleVariant::get<int>() {
    assert(type_ == Type::Int);
    return *reinterpret_cast<int*>(storage_);
}
template<> double& SimpleVariant::get<double>() {
    assert(type_ == Type::Double);
    return *reinterpret_cast<double*>(storage_);
}
template<> std::string& SimpleVariant::get<std::string>() {
    assert(type_ == Type::String);
    return *reinterpret_cast<std::string*>(storage_);
}

int main() {
    std::cout << "=== SimpleVariant with alignas + std::byte ===\n\n";

    std::cout << "Storage size: " << sizeof(SimpleVariant) << " bytes\n";
    std::cout << "(std::string is " << sizeof(std::string) << " bytes)\n\n";

    SimpleVariant v;
    v.print();  // (empty)

    v.set(42);
    v.print();  // int: 42

    v.set(3.14);
    v.print();  // double: 3.14

    v.set(std::string("Hello, variant!"));
    v.print();  // string: "Hello, variant!"

    // Modify in place
    v.get<std::string>() += " More text.";
    v.print();

    // Replace string with int (string destructor called automatically)
    v.set(99);
    v.print();  // int: 99

    return 0;
}
```

**Output:**

```text
=== SimpleVariant with alignas + std::byte ===

Storage size: 48 bytes
(std::string is 32 bytes)

(empty)
int: 42
double: 3.14
string: "Hello, variant!"
string: "Hello, variant! More text."
int: 99
```

When `set(99)` is called after storing a string, `destroy()` properly calls the `std::string` destructor before constructing the `int`. That is the whole lifecycle: `construct_at` - use - `destroy_at`.

---

## Notes

- `std::aligned_storage` and `std::aligned_union` are **deprecated in C++23** (P1413R3).
- The replacement is simply `alignas(T) std::byte storage[sizeof(T)]` - clearer and correct.
- Always use `std::construct_at` / `std::destroy_at` (C++20) for lifetime management in raw storage.
- For production variant-like types, use `std::variant` - it handles all the complexity.
- `alignas` on a `std::byte` array ensures proper alignment for any type you store.
