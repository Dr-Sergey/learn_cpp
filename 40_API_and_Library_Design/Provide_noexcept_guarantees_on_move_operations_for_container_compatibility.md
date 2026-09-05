# Provide noexcept guarantees on move operations for container compatibility

**Category:** API and Library Design  
**Standard:** C++11, C++17  
**Reference:** <https://en.cppreference.com/w/cpp/types/is_nothrow_move_constructible>  

---

## Topic Overview

`std::vector` and other containers use `std::move_if_noexcept` during reallocation. The reason this matters so much is the strong exception guarantee: if a reallocation throws partway through, the container must be able to restore itself to its original state. Moves can't be undone - once you've moved an element, the source is in a "valid but unspecified" state. Copies can be undone by simply discarding the partially-filled new buffer. So if your type's move constructor is not `noexcept`, vector plays it safe and copies everything instead. That can be a massive performance penalty when your type holds heap-allocated data.

### The Problem

Here's a concrete demonstration of how a missing `noexcept` forces copies when you expected moves:

```cpp
#include <vector>
#include <iostream>
#include <string>
#include <type_traits>
#include <utility>

struct Slow {
    std::string data;
    // Move constructor NOT noexcept
    Slow(Slow&& other) : data(std::move(other.data)) {
        // Missing noexcept!
    }
};

struct Fast {
    std::string data;
    // Move constructor IS noexcept
    Fast(Fast&& other) noexcept : data(std::move(other.data)) {}
};

int main() {
    // Slow: vector COPIES during reallocation (for strong exception guarantee)
    std::vector<Slow> v1;
    for (int i = 0; i < 10000; ++i) v1.push_back(Slow{"data"});
    // Many unnecessary copies!

    // Fast: vector MOVES during reallocation
    std::vector<Fast> v2;
    for (int i = 0; i < 10000; ++i) v2.push_back(Fast{"data"});
    // Efficient moves!

    std::cout << "Slow is nothrow move: "
              << std::is_nothrow_move_constructible_v<Slow> << "\n";  // 0
    std::cout << "Fast is nothrow move: "
              << std::is_nothrow_move_constructible_v<Fast> << "\n";  // 1
}
```

The only difference between `Slow` and `Fast` is the `noexcept` keyword on the move constructor. That one word determines whether vector reallocates by moving (cheap pointer swaps) or by copying (deep heap allocation clones). For types that hold anything heap-allocated, this is the difference between O(n) pointer operations and O(n) full copies.

### Conditional noexcept

When you write a generic wrapper type, you want its move operations to be `noexcept` if and only if the wrapped type's move operations are `noexcept`. You should propagate the guarantee rather than either unconditionally promising it (which could be lying) or unconditionally omitting it (which loses the performance benefit for all the types that do qualify):

```cpp
#include <type_traits>

template<typename T>
class Wrapper {
    T value_;
public:
    // Propagate noexcept from T's move constructor
    Wrapper(Wrapper&& other) noexcept(std::is_nothrow_move_constructible_v<T>)
        : value_(std::move(other.value_)) {}

    Wrapper& operator=(Wrapper&& other)
        noexcept(std::is_nothrow_move_assignable_v<T>) {
        value_ = std::move(other.value_);
        return *this;
    }
};
```

This is the correct approach for any generic container or wrapper: let the `noexcept` status flow through from the element type automatically.

---

## Self-Assessment

### Q1: How does std::move_if_noexcept work

`std::move_if_noexcept` is how the standard library implements the "move only if safe" decision. The logic is straightforward - if the type can be moved without throwing, move it; if it might throw, copy it instead:

```cpp
// Simplified implementation:
template<typename T>
auto move_if_noexcept(T& x) noexcept {
    if constexpr (std::is_nothrow_move_constructible_v<T>)
        return std::move(x);  // Move - safe, no exceptions
    else
        return x;             // Copy - safe, strong guarantee
}
```

`std::vector` calls something equivalent to this on each element during reallocation. The `noexcept` status of your type's move constructor is what flips that branch.

### Q2: Show the performance impact with a benchmark

The performance gap comes entirely from the cost difference between moving and copying heap-allocated data:

```cpp
// With noexcept move: vector reallocation is O(n) moves ~= O(n) pointer swaps
// Without noexcept move: vector reallocation is O(n) copies ~= O(n) string copies
// For std::string of 1KB: moves are ~100x faster than copies
// For a vector growing to 10000 elements: total difference is ~10ms vs ~1s
```

For small types that fit in a few registers, the difference is negligible. For types with heap-allocated members (strings, vectors, unique_ptr-wrapped data), the gap can be enormous.

### Q3: What about swap

`swap` should always be `noexcept`. A throwing swap is almost never useful - the copy-and-swap idiom and many generic algorithms rely on swap being safe to call unconditionally:

```cpp
class MyType {
    std::vector<int> data_;
public:
    // Swap should ALWAYS be noexcept
    friend void swap(MyType& a, MyType& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);  // vector::swap is noexcept
    }

    // Copy-and-swap assignment leverages noexcept swap
    MyType& operator=(MyType other) noexcept {
        swap(*this, other);
        return *this;
    }
};
```

Since `std::vector::swap` is already `noexcept`, delegating to it lets you mark your own `swap` `noexcept` without any risk. The copy-and-swap assignment then gets the `noexcept` for free.

---

## Notes

- As a rule, move constructors and move assignment operators should always be `noexcept` if you can make them so - the performance implications are real.
- Use `static_assert(std::is_nothrow_move_constructible_v<MyType>)` to lock in the guarantee and get an immediate compile error if a future change breaks it.
- If your type wraps only standard library types with defaulted move operations, those are almost certainly already `noexcept` - the compiler-generated defaults propagate `noexcept` from member types automatically.
- Use `noexcept(noexcept(...))` for conditional propagation in generic code when you want to be precise about exactly which expression you're querying.
