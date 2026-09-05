# Use `std::is_nothrow_move_constructible` to Guide Safe Generic Code

**Category:** Type System and Deduction  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/types/is_move_constructible>  

---

## Topic Overview

### Why Nothrow Moves Matter in Generic Code

When writing generic containers or algorithms, you face a critical decision: **use move or copy** to transfer elements? If a move constructor throws mid-way through a batch operation (e.g., vector reallocation), you've corrupted the original data and can't recover. Move semantics only give you a strong exception guarantee when the move itself can't throw.

`std::is_nothrow_move_constructible_v<T>` lets you query at compile time whether moving `T` is safe, and choose your strategy accordingly.

### The `std::vector` Problem

During `push_back` when capacity is exceeded, `std::vector` must transfer all elements to a new buffer. The dilemma is real:

| Strategy | Safe if move throws? | Performance |
| --- | --- | --- |
| Copy all elements | Yes - original still intact | Slow (deep copies) |
| Move all elements | No - can't restore originals | Fast |
| Move if noexcept, else copy | Yes - best of both | Optimal |

This is exactly what `std::vector` does internally with `std::move_if_noexcept`. The standard library literally branches on this trait to give you both safety and performance.

### Key Traits and Utilities

Here are the tools for working with this pattern:

```cpp
#include <type_traits>
#include <utility>

// Type traits
std::is_nothrow_move_constructible_v<T>   // true if T(T&&) is noexcept
std::is_nothrow_move_assignable_v<T>      // true if T::operator=(T&&) is noexcept

// Conditional move utility
std::move_if_noexcept(x)
// Returns: T&& if T is nothrow move constructible
//          const T& otherwise (forces copy)
```

### Making Your Types Noexcept-Move

The fix is straightforward - annotate move operations with `noexcept` when they can't throw, which for most types that just move members is always the case:

```cpp
struct Good {
    std::vector<int> data;
    // Rule: Mark move operations noexcept when they can't throw
    Good(Good&& other) noexcept : data(std::move(other.data)) {}
    Good& operator=(Good&& other) noexcept {
        data = std::move(other.data);
        return *this;
    }
};

static_assert(std::is_nothrow_move_constructible_v<Good>);
```

### Using in Generic Code: `if constexpr` Dispatch

You can make your own generic code branch on this trait the same way the standard library does:

```cpp
template<typename T>
void transfer(T& dst, T& src) {
    if constexpr (std::is_nothrow_move_constructible_v<T>) {
        dst = std::move(src);  // safe - won't throw
    } else {
        dst = src;  // fallback to copy - strong exception guarantee
    }
}
```

---

## Self-Assessment

### Q1: Show how `std::vector` uses `is_nothrow_move_constructible` to choose between move and copy on reallocation

The key here is to actually watch the behavior differ. One type has `noexcept` on its move constructor; the other does not. Watch which operations fire during vector reallocation:

```cpp
#include <iostream>
#include <type_traits>
#include <vector>
#include <utility>
#include <cstring>

// A type WITH noexcept move
struct FastWidget {
    int id;
    FastWidget(int i) : id(i) {}
    FastWidget(const FastWidget& o) : id(o.id) {
        std::cout << "  FastWidget COPY " << id << "\n";
    }
    FastWidget(FastWidget&& o) noexcept : id(o.id) {
        std::cout << "  FastWidget MOVE " << id << "\n";
        o.id = -1;
    }
    FastWidget& operator=(const FastWidget&) = default;
    FastWidget& operator=(FastWidget&&) noexcept = default;
};

// A type WITHOUT noexcept move
struct SlowWidget {
    int id;
    SlowWidget(int i) : id(i) {}
    SlowWidget(const SlowWidget& o) : id(o.id) {
        std::cout << "  SlowWidget COPY " << id << "\n";
    }
    SlowWidget(SlowWidget&& o) : id(o.id) {  // NOT noexcept!
        std::cout << "  SlowWidget MOVE " << id << "\n";
        o.id = -1;
    }
    SlowWidget& operator=(const SlowWidget&) = default;
    SlowWidget& operator=(SlowWidget&&) = default;
};

// Simulates what vector does internally
template<typename T>
void simulate_realloc(T* old_buf, T* new_buf, size_t count) {
    std::cout << "  move_if_noexcept strategy for "
              << (std::is_nothrow_move_constructible_v<T> ? "MOVE" : "COPY") << ":\n";
    for (size_t i = 0; i < count; ++i) {
        // This is the key: move_if_noexcept returns T&& or const T&
        ::new (new_buf + i) T(std::move_if_noexcept(old_buf[i]));
    }
}

int main() {
    std::cout << std::boolalpha;
    std::cout << "FastWidget nothrow move: "
              << std::is_nothrow_move_constructible_v<FastWidget> << "\n";
    std::cout << "SlowWidget nothrow move: "
              << std::is_nothrow_move_constructible_v<SlowWidget> << "\n\n";

    // Real vector demonstration
    std::cout << "=== std::vector<FastWidget> reallocation ===\n";
    std::vector<FastWidget> fast;
    fast.reserve(2);
    fast.emplace_back(1);
    fast.emplace_back(2);
    std::cout << "Triggering reallocation (push_back):\n";
    fast.emplace_back(3);  // Will MOVE existing elements

    std::cout << "\n=== std::vector<SlowWidget> reallocation ===\n";
    std::vector<SlowWidget> slow;
    slow.reserve(2);
    slow.emplace_back(10);
    slow.emplace_back(20);
    std::cout << "Triggering reallocation (push_back):\n";
    slow.emplace_back(30);  // Will COPY existing elements (move not noexcept)

    return 0;
}
```

**Output:**

```text
FastWidget nothrow move: true
SlowWidget nothrow move: false

=== std::vector<FastWidget> reallocation ===
Triggering reallocation (push_back):
  FastWidget MOVE 1
  FastWidget MOVE 2

=== std::vector<SlowWidget> reallocation ===
Triggering reallocation (push_back):
  SlowWidget COPY 10
  SlowWidget COPY 20
```

`SlowWidget` elements are **copied** despite having a move constructor, because the move is not `noexcept`. The vector can't risk a throwing move corrupting the source buffer during reallocation.

### Q2: Write a generic swap that uses move only when `is_nothrow_move_constructible` is true

Notice that the `noexcept` on `safe_swap` is itself conditional - the function is only nothrow when the underlying type supports nothrow moves. That's the right way to propagate exception guarantees in generic code:

```cpp
#include <iostream>
#include <type_traits>
#include <utility>
#include <string>

// Generic safe_swap: uses move when nothrow, copy otherwise
template<typename T>
void safe_swap(T& a, T& b) noexcept(std::is_nothrow_move_constructible_v<T> &&
                                      std::is_nothrow_move_assignable_v<T>)
{
    if constexpr (std::is_nothrow_move_constructible_v<T> &&
                  std::is_nothrow_move_assignable_v<T>) {
        // Safe to move - no exceptions possible
        T temp(std::move(a));
        a = std::move(b);
        b = std::move(temp);
    } else {
        // Fallback to copy - strong exception guarantee
        T temp(a);       // copy
        a = b;           // copy
        b = temp;        // copy
    }
}

// Type with noexcept move
struct NothrowType {
    int value;
    NothrowType(int v) : value(v) {}
    NothrowType(NothrowType&& o) noexcept : value(o.value) {
        std::cout << "  [NothrowType] MOVE\n";
    }
    NothrowType(const NothrowType& o) : value(o.value) {
        std::cout << "  [NothrowType] COPY\n";
    }
    NothrowType& operator=(NothrowType&& o) noexcept {
        value = o.value;
        std::cout << "  [NothrowType] MOVE-ASSIGN\n";
        return *this;
    }
    NothrowType& operator=(const NothrowType& o) {
        value = o.value;
        std::cout << "  [NothrowType] COPY-ASSIGN\n";
        return *this;
    }
};

// Type with potentially-throwing move
struct ThrowingType {
    int value;
    ThrowingType(int v) : value(v) {}
    ThrowingType(ThrowingType&& o) : value(o.value) {  // No noexcept
        std::cout << "  [ThrowingType] MOVE\n";
    }
    ThrowingType(const ThrowingType& o) : value(o.value) {
        std::cout << "  [ThrowingType] COPY\n";
    }
    ThrowingType& operator=(ThrowingType&& o) {
        value = o.value;
        std::cout << "  [ThrowingType] MOVE-ASSIGN\n";
        return *this;
    }
    ThrowingType& operator=(const ThrowingType& o) {
        value = o.value;
        std::cout << "  [ThrowingType] COPY-ASSIGN\n";
        return *this;
    }
};

int main() {
    std::cout << "=== safe_swap with NothrowType (uses MOVE) ===\n";
    NothrowType a{1}, b{2};
    safe_swap(a, b);
    std::cout << "a=" << a.value << ", b=" << b.value << "\n";

    std::cout << "\n=== safe_swap with ThrowingType (uses COPY) ===\n";
    ThrowingType c{10}, d{20};
    safe_swap(c, d);
    std::cout << "c=" << c.value << ", d=" << d.value << "\n";

    // Verify noexcept propagation
    static_assert(noexcept(safe_swap(a, b)),  "NothrowType swap should be noexcept");
    static_assert(!noexcept(safe_swap(c, d)), "ThrowingType swap should NOT be noexcept");

    return 0;
}
```

**Output:**

```text
=== safe_swap with NothrowType (uses MOVE) ===
  [NothrowType] MOVE
  [NothrowType] MOVE-ASSIGN
  [NothrowType] MOVE-ASSIGN
a=2, b=1

=== safe_swap with ThrowingType (uses COPY) ===
  [ThrowingType] COPY
  [ThrowingType] COPY-ASSIGN
  [ThrowingType] COPY-ASSIGN
c=20, d=10
```

### Q3: Trace why `std::string` may not be nothrow-move-constructible on all implementations

This is a subtlety worth knowing: `std::string` itself is guaranteed nothrow-move by the standard, but the more general `std::basic_string` with a custom allocator is not always:

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <memory>
#include <new>
#include <vector>

int main() {
    std::cout << std::boolalpha;

    // On most modern implementations (GCC, Clang, MSVC), this is true:
    std::cout << "std::string nothrow-move: "
              << std::is_nothrow_move_constructible_v<std::string> << "\n";

    // But historically, some implementations had issues:
    // 1. COW (Copy-on-Write) implementations (pre-C++11 GCC libstdc++):
    //    - Move was essentially a pointer swap, but the reference count
    //      manipulation might have involved synchronization that could throw
    //
    // 2. Custom allocators with propagate_on_container_move_assignment:
    //    - If allocator doesn't propagate on move, string must allocate
    //      new buffer and copy characters -> can throw std::bad_alloc
    //
    // 3. Small-string optimization (SSO):
    //    - Short strings stored inline -> move = memcpy-like (noexcept)
    //    - Long strings stored on heap -> move = pointer swap (noexcept)
    //    - But: if allocators differ and don't propagate, still must copy

    // Demonstrate with custom allocator scenarios:
    using DefaultString = std::basic_string<char>;
    std::cout << "basic_string<char> nothrow-move: "
              << std::is_nothrow_move_constructible_v<DefaultString> << "\n";

    // The standard GUARANTEES noexcept move for std::string since C++11:
    //   string(string&& str) noexcept;   [basic.string] / [string.cons]
    //
    // BUT: basic_string<char, traits, Alloc> with a custom Alloc
    // that has POCMA = false may NOT be nothrow-move-constructible:

    // Illustration:
    // template<typename T>
    // struct ThrowingAllocator : std::allocator<T> {
    //     using propagate_on_container_move_assignment = std::false_type;
    //     // Move construction of basic_string with this allocator
    //     // MIGHT need to allocate if source allocator != dest allocator
    // };
    // using CustomString = std::basic_string<char, std::char_traits<char>,
    //                                         ThrowingAllocator<char>>;
    // This might have is_nothrow_move_constructible = false

    // Summary table:
    std::cout << "\n=== nothrow-move status of common types ===\n";
    std::cout << "int:                    " << std::is_nothrow_move_constructible_v<int> << "\n";
    std::cout << "double:                 " << std::is_nothrow_move_constructible_v<double> << "\n";
    std::cout << "std::string:            " << std::is_nothrow_move_constructible_v<std::string> << "\n";
    std::cout << "std::vector<int>:       " << std::is_nothrow_move_constructible_v<std::vector<int>> << "\n";
    std::cout << "std::unique_ptr<int>:   " << std::is_nothrow_move_constructible_v<std::unique_ptr<int>> << "\n";
    std::cout << "std::shared_ptr<int>:   " << std::is_nothrow_move_constructible_v<std::shared_ptr<int>> << "\n";

    // Practical lesson: Always check the trait for your ACTUAL type.
    // Don't assume all string-like types are nothrow movable.

    return 0;
}
```

**Output (typical modern compiler):**

```text
std::string nothrow-move: true
basic_string<char> nothrow-move: true

=== nothrow-move status of common types ===
int:                    true
double:                 true
std::string:            true
std::vector<int>:       true
std::unique_ptr<int>:   true
std::shared_ptr<int>:   true
```

`std::string` with the default allocator IS noexcept-move on all conforming C++11+ implementations. The risk arises with **`basic_string` using custom allocators** where `propagate_on_container_move_assignment` is `false_type`. In that case, move construction may need to allocate and copy, potentially throwing `std::bad_alloc`.

---

## Notes

- **Always mark your move constructors `noexcept`** when they don't allocate or call potentially-throwing operations. This single keyword can dramatically improve performance in `std::vector` and other containers.
- Use `static_assert(std::is_nothrow_move_constructible_v<MyType>)` as a regression test in your code.
- `std::move_if_noexcept(x)` is the standard utility that implements the "move-if-safe, copy-if-not" pattern.
- The `noexcept` specifier is part of the type system since C++17 - it affects overload resolution and function pointer compatibility.
