# Understand Conditional `noexcept` and Its Impact on `std::vector` Reallocation

**Category:** Move Semantics and Value Categories  
**Standard:** C++11 and later  
**Reference:** <https://en.cppreference.com/w/cpp/utility/move_if_noexcept>  

---

## Topic Overview

When `std::vector` needs to reallocate (due to `push_back`, `emplace_back`, `resize`, etc.), it must transfer existing elements from the old buffer to the new buffer. The vector has two choices: **copy** or **move** each element. Moving is faster, but if a move constructor throws an exception midway, the vector is left in an unrecoverable state - some elements have been moved out of the old buffer, the new buffer is incomplete, and the strong exception guarantee is violated.

This is the core tension: moving is fast but risky if it can throw; copying is slow but safe. The vector resolves this by asking the type's move constructor whether it promises not to throw.

```cpp
┌────────────────────────────────────────────────────────────────────┐
│         vector Reallocation: Copy vs Move Decision                 │
│                                                                    │
│   if (is_nothrow_move_constructible<T> ||                          │
│       !is_copy_constructible<T>)                                   │
│   {                                                                │
│       // MOVE elements - fast, safe (can't throw) or only option   │
│   }                                                                │
│   else                                                             │
│   {                                                                │
│       // COPY elements - slow but provides strong exception safety │
│   }                                                                │
│                                                                    │
│   Implemented internally via std::move_if_noexcept(element)        │
└────────────────────────────────────────────────────────────────────┘
```

The vector uses `std::move_if_noexcept` internally, which returns an rvalue reference if the type's move constructor is `noexcept`, or a const lvalue reference otherwise. The effect is:

```cpp
┌────────────────────────────────────┬─────────────────────────────┐
│  Move ctor is...                   │  vector reallocation does..│
├────────────────────────────────────┼─────────────────────────────┤
│  noexcept                          │  MOVE (fast)               │
│  potentially throwing + copyable   │  COPY (slow but safe)      │
│  potentially throwing + !copyable  │  MOVE (only option, risky) │
│  not declared at all               │  uses copy ctor            │
└────────────────────────────────────┴─────────────────────────────┘
```

This means that **forgetting `noexcept` on your move constructor** silently degrades `std::vector` performance from O(1) amortized moves to O(n) copies per reallocation. The fix is simple: always mark move constructors and move assignment operators as `noexcept` when they truly cannot throw - which is almost always the case, since move operations that only reassign pointers, sizes, and built-in types never throw.

For conditional cases, use `noexcept(expression)` to propagate noexcept-ness from member types:

```cpp
MyClass(MyClass&& o) noexcept(
    std::is_nothrow_move_constructible_v<Member1> &&
    std::is_nothrow_move_constructible_v<Member2>
) = default;
```

Or simply `= default` the move constructor - the compiler will compute the correct `noexcept` specification automatically based on all members. This is the preferred approach when you can use it.

---

## Self-Assessment

### Q1: How does forgetting `noexcept` on a move constructor affect `std::vector` performance

The benchmark below shows the same work done by two types that differ only in one keyword: `noexcept`. Watch the performance difference.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <chrono>

struct BadMove {
    std::string data;
    int id;

    BadMove(int id, std::string s) : data(std::move(s)), id(id) {}

    // Move constructor WITHOUT noexcept - vector will COPY instead
    BadMove(BadMove&& o) : data(std::move(o.data)), id(o.id) {
        // No throw here, but compiler doesn't know that!
    }

    BadMove(const BadMove& o) : data(o.data), id(o.id) {}
    BadMove& operator=(BadMove&&) = default;
    BadMove& operator=(const BadMove&) = default;
};

struct GoodMove {
    std::string data;
    int id;

    GoodMove(int id, std::string s) : data(std::move(s)), id(id) {}

    // Move constructor WITH noexcept - vector will MOVE (fast)
    GoodMove(GoodMove&& o) noexcept : data(std::move(o.data)), id(o.id) {}

    GoodMove(const GoodMove& o) : data(o.data), id(o.id) {}
    GoodMove& operator=(GoodMove&&) noexcept = default;
    GoodMove& operator=(const GoodMove&) = default;
};

template<typename T>
long long benchmark_pushback(int n, const std::string& label) {
    auto start = std::chrono::high_resolution_clock::now();
    std::vector<T> v;
    for (int i = 0; i < n; ++i) {
        v.emplace_back(i, std::string(200, 'x'));
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
    std::cout << label << ": " << us.count() << " us\n";
    return us.count();
}

int main() {
    const int N = 100'000;

    std::cout << "is_nothrow_move_constructible:\n";
    std::cout << "  BadMove:  "
              << std::is_nothrow_move_constructible_v<BadMove> << "\n";
    std::cout << "  GoodMove: "
              << std::is_nothrow_move_constructible_v<GoodMove> << "\n\n";

    auto bad_time  = benchmark_pushback<BadMove>(N, "BadMove  (copies on realloc)");
    auto good_time = benchmark_pushback<GoodMove>(N, "GoodMove (moves on realloc)");

    if (bad_time > 0 && good_time > 0) {
        std::cout << "\nRatio (bad/good): ~"
                  << (double)bad_time / good_time << "x\n";
    }

    return 0;
}
```

The `is_nothrow_move_constructible` printout confirms the trait value before the benchmark runs. If `BadMove` had `noexcept`, both rows would show `1` and the timings would be similar.

### Q2: How does `std::move_if_noexcept` work, and when would you use it directly

You rarely call `std::move_if_noexcept` yourself - it is mainly used inside container implementations. But understanding it directly shows you exactly what decision `std::vector` is making on your behalf.

```cpp
#include <iostream>
#include <utility>
#include <type_traits>
#include <string>

struct ThrowingMove {
    std::string data;

    ThrowingMove(std::string s) : data(std::move(s)) {}

    // Potentially throwing move
    ThrowingMove(ThrowingMove&& o) : data(std::move(o.data)) {}
    ThrowingMove(const ThrowingMove& o) : data(o.data) {}
    ThrowingMove& operator=(ThrowingMove&&) = default;
    ThrowingMove& operator=(const ThrowingMove&) = default;
};

struct SafeMove {
    std::string data;

    SafeMove(std::string s) : data(std::move(s)) {}

    SafeMove(SafeMove&& o) noexcept : data(std::move(o.data)) {}
    SafeMove(const SafeMove& o) : data(o.data) {}
    SafeMove& operator=(SafeMove&&) noexcept = default;
    SafeMove& operator=(const SafeMove&) = default;
};

struct MoveOnly {
    std::string data;

    MoveOnly(std::string s) : data(std::move(s)) {}
    MoveOnly(MoveOnly&& o) : data(std::move(o.data)) {}  // throws? maybe

    // Not copyable!
    MoveOnly(const MoveOnly&) = delete;
    MoveOnly& operator=(MoveOnly&&) = default;
    MoveOnly& operator=(const MoveOnly&) = delete;
};

int main() {
    // std::move_if_noexcept returns:
    //   T&&        if noexcept move OR not copyable
    //   const T&   if potentially throwing move AND copyable

    ThrowingMove tm{"throwing"};
    SafeMove sm{"safe"};
    MoveOnly mo{"moveonly"};

    // ThrowingMove: has throwing move + is copyable => const T&
    auto&& ref1 = std::move_if_noexcept(tm);
    std::cout << "ThrowingMove: "
              << (std::is_lvalue_reference_v<decltype(ref1)>
                  ? "const T& (will copy)" : "T&& (will move)")
              << "\n";

    // SafeMove: has noexcept move => T&&
    auto&& ref2 = std::move_if_noexcept(sm);
    std::cout << "SafeMove:     "
              << (std::is_lvalue_reference_v<decltype(ref2)>
                  ? "const T& (will copy)" : "T&& (will move)")
              << "\n";

    // MoveOnly: throwing move but NOT copyable => T&& (only option)
    auto&& ref3 = std::move_if_noexcept(mo);
    std::cout << "MoveOnly:     "
              << (std::is_lvalue_reference_v<decltype(ref3)>
                  ? "const T& (will copy)" : "T&& (will move)")
              << "\n";

    return 0;
}
```

The `MoveOnly` case shows the "only option" escape hatch: if the type cannot be copied, the vector moves it even without `noexcept`, accepting the weaker exception guarantee. There is no other choice.

### Q3: How do you write a conditionally `noexcept` move constructor for a class with multiple members

When you have a template class whose `T` might or might not have a `noexcept` move, you want your move constructor's `noexcept`-ness to track `T`'s. The cleanest way is `= default`; the explicit way is `noexcept(is_nothrow_move_constructible_v<T>)`.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <type_traits>
#include <utility>
#include <memory>

// A type with conditional noexcept based on template parameter
template<typename T>
class Container {
    T value_;
    std::size_t size_;

public:
    Container() : value_{}, size_{0} {}
    explicit Container(T val) : value_(std::move(val)), size_{1} {}

    // Option 1: Manually conditional noexcept
    Container(Container&& o)
        noexcept(std::is_nothrow_move_constructible_v<T>)
        : value_(std::move(o.value_))
        , size_(std::exchange(o.size_, 0))
    {}

    Container& operator=(Container&& o)
        noexcept(std::is_nothrow_move_assignable_v<T>)
    {
        value_ = std::move(o.value_);
        size_  = std::exchange(o.size_, 0);
        return *this;
    }

    Container(const Container&) = default;
    Container& operator=(const Container&) = default;

    const T& get() const { return value_; }
};

// Option 2: Let the compiler compute it via = default
// This is the preferred approach when possible.
template<typename T>
struct AutoContainer {
    T value;
    std::size_t size = 0;

    // The compiler computes noexcept based on all members:
    //   noexcept if T's move ctor is noexcept AND size_t move is noexcept
    AutoContainer(AutoContainer&&) = default;
    AutoContainer& operator=(AutoContainer&&) = default;
    AutoContainer(const AutoContainer&) = default;
    AutoContainer& operator=(const AutoContainer&) = default;
    AutoContainer() = default;
};

int main() {
    // Container<std::string>: string move is noexcept => Container move is noexcept
    std::cout << "Container<string> nothrow move: "
              << std::is_nothrow_move_constructible_v<Container<std::string>>
              << "\n";

    // Verify vector uses move for noexcept types
    {
        std::vector<Container<std::string>> v;
        v.emplace_back("hello");
        v.emplace_back("world");
        v.emplace_back("test");
        std::cout << "vector of Container<string>: size=" << v.size()
                  << " (moved during realloc)\n";
    }

    // AutoContainer: compiler-deduced noexcept
    std::cout << "AutoContainer<string> nothrow move: "
              << std::is_nothrow_move_constructible_v<AutoContainer<std::string>>
              << "\n";
    std::cout << "AutoContainer<int> nothrow move: "
              << std::is_nothrow_move_constructible_v<AutoContainer<int>>
              << "\n";

    // Practical check: does YOUR type pass the vector optimization?
    struct MyType {
        std::vector<int> data;
        std::string name;
        int id;

        // If you = default, the compiler does it right:
        MyType(MyType&&) = default;
        MyType& operator=(MyType&&) = default;
        MyType(const MyType&) = default;
        MyType& operator=(const MyType&) = default;
        MyType() = default;
    };

    std::cout << "\nMyType nothrow move: "
              << std::is_nothrow_move_constructible_v<MyType> << "\n";

    static_assert(std::is_nothrow_move_constructible_v<MyType>,
                  "MyType should have noexcept move for vector efficiency!");

    return 0;
}
```

The `static_assert` at the bottom is a good habit: it turns a silent performance regression into a compile error. If someone later adds a member with a throwing move, the assert fires and reminds them to fix it.

---

## Notes

- `std::vector` uses `std::move_if_noexcept` during reallocation: it moves elements only if the move constructor is `noexcept`; otherwise it copies for strong exception safety.
- **Always mark move constructors and move assignment operators `noexcept`** when they cannot throw. Omitting it is a silent performance bug.
- Prefer `= default` for move operations - the compiler automatically computes the correct `noexcept` specification based on all data members.
- For template classes, use conditional `noexcept`: `noexcept(std::is_nothrow_move_constructible_v<T>)`.
- If your type is move-only (no copy constructor), vector will move even without `noexcept` - it has no other choice. But it means you lose the strong exception guarantee.
- Use `static_assert(std::is_nothrow_move_constructible_v<MyType>)` in your code to catch accidental `noexcept` regressions at compile time.
- This behavior also affects `std::vector::reserve`, `resize`, `insert`, and any operation that may trigger reallocation - not just `push_back`.
