# Understand `std::is_nothrow_move_constructible` and Its Effect on `std::vector`

**Category:** Type System and Deduction  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/types/is_move_constructible>  

---

## Topic Overview

### What Is `std::is_nothrow_move_constructible`

`std::is_nothrow_move_constructible<T>` is a type trait that checks whether type `T` can be move-constructed without throwing exceptions. It's `true` when the move constructor is declared `noexcept` (or is trivial).

```cpp
#include <type_traits>
#include <string>
#include <vector>

static_assert(std::is_nothrow_move_constructible_v<int>);           // true: trivial
static_assert(std::is_nothrow_move_constructible_v<std::string>);   // true: noexcept move
static_assert(std::is_nothrow_move_constructible_v<std::vector<int>>); // true
```

### Why Does `std::vector` Care

When `std::vector` needs to **reallocate** (e.g., during `push_back` when capacity is exceeded), it must transfer elements from the old buffer to the new one. It has two options:

```cpp
// Option A: MOVE elements (fast, but if a move throws mid-transfer,
//           old buffer is already partially destroyed -> data loss!)

// Option B: COPY elements (slow, but if a copy throws,
//           old buffer is intact -> strong exception guarantee preserved)
```

`std::vector` uses **move** (Option A) only if `is_nothrow_move_constructible_v<T>` is `true`. Otherwise, it falls back to **copy** (Option B). This is one of the most impactful places that `noexcept` on a move constructor has a real performance effect.

### The Strong Exception Guarantee

The C++ standard requires that `push_back` provides the **strong exception guarantee**:

> If an exception is thrown, the container is left in its previous valid state.

Here's why this forces the conditional behavior:

```cpp
// Reallocation with MOVE:
//   [old buffer: A B C D]
//   Transfer: A->new, B->new, C fails! (throws)
//   Problem: A and B are already moved-from in old buffer
//            -> old buffer is corrupted -> CANNOT restore previous state!
//            -> strong guarantee VIOLATED

// Reallocation with COPY:
//   [old buffer: A B C D]
//   Transfer: A->new, B->new, C fails! (throws)
//   Old buffer still has: A B C D (untouched, they were copied)
//   -> Destroy new buffer, keep old -> strong guarantee PRESERVED
```

### The Decision Logic

The actual implementation uses `std::move_if_noexcept`, which neatly encodes the three-way decision:

```cpp
// Simplified logic inside vector::push_back / resize:
if constexpr (std::is_nothrow_move_constructible_v<T>) {
    // Safe to move — no exception possible
    std::uninitialized_move(old_begin, old_end, new_begin);
} else if constexpr (std::is_copy_constructible_v<T>) {
    // Fall back to copy — preserves strong guarantee
    std::uninitialized_copy(old_begin, old_end, new_begin);
} else {
    // Move-only type that might throw — move anyway (best effort)
    std::uninitialized_move(old_begin, old_end, new_begin);
}
```

The actual implementation uses `std::move_if_noexcept`:

```cpp
// std::move_if_noexcept<T>(x):
//   If T is nothrow-move-constructible -> return std::move(x)  (rvalue)
//   Else if T is copy-constructible    -> return x              (lvalue -> copy)
//   Else                               -> return std::move(x)  (last resort)
```

### Performance Impact

| Type's move constructor | vector reallocation | Performance |
| --- | --- | --- |
| `noexcept` (e.g., `std::string`) | Moves elements | Fast: O(n) pointer/size copies |
| Might throw + copyable | Copies elements | Slow: O(n) deep copies |
| Might throw + move-only | Moves (no choice) | Fast but unsafe if it throws |
| No move + copyable | Copies elements | Slow |

### How to Ensure `noexcept` Move

The rule is simple: if your move constructor does nothing that can throw, mark it `noexcept`. The compiler won't infer it from the body:

```cpp
struct Good {
    std::vector<int> data;
    std::string name;
    // Implicitly generated move ctor is noexcept
    // because all members have noexcept moves
};
static_assert(std::is_nothrow_move_constructible_v<Good>);

struct Bad {
    Bad(Bad&&) {}  // Move ctor NOT marked noexcept!
};
static_assert(!std::is_nothrow_move_constructible_v<Bad>);  // false!

struct Fixed {
    Fixed(Fixed&&) noexcept {}  // Explicitly noexcept
};
static_assert(std::is_nothrow_move_constructible_v<Fixed>);
```

---

## Self-Assessment

### Q1: Show that `std::vector` uses `is_nothrow_move_constructible` to choose move vs copy during realloc

This is a live demonstration - you can see the move/copy counts directly. The only difference between `FastType` and `SlowType` is the `noexcept` annotation on the move constructor:

```cpp
#include <iostream>
#include <vector>
#include <type_traits>
#include <string>

// Type with noexcept move — vector will MOVE during realloc
struct FastType {
    int data;
    static int move_count;
    static int copy_count;

    FastType(int d = 0) : data(d) {}
    FastType(const FastType& o) : data(o.data) { ++copy_count; }
    FastType(FastType&& o) noexcept : data(o.data) { ++move_count; o.data = 0; }
    FastType& operator=(const FastType&) = default;
    FastType& operator=(FastType&&) noexcept = default;
};
int FastType::move_count = 0;
int FastType::copy_count = 0;

// Type WITHOUT noexcept move — vector will COPY during realloc
struct SlowType {
    int data;
    static int move_count;
    static int copy_count;

    SlowType(int d = 0) : data(d) {}
    SlowType(const SlowType& o) : data(o.data) { ++copy_count; }
    SlowType(SlowType&& o) : data(o.data) { ++move_count; o.data = 0; }  // NO noexcept!
    SlowType& operator=(const SlowType&) = default;
    SlowType& operator=(SlowType&&) = default;
};
int SlowType::move_count = 0;
int SlowType::copy_count = 0;

int main() {
    static_assert(std::is_nothrow_move_constructible_v<FastType>,
                  "FastType should be nothrow moveable");
    static_assert(!std::is_nothrow_move_constructible_v<SlowType>,
                  "SlowType should NOT be nothrow moveable");

    // Force reallocation by exceeding capacity
    std::cout << "=== FastType (noexcept move) ===\n";
    {
        std::vector<FastType> v;
        v.reserve(2);  // Start with capacity 2
        v.emplace_back(1);
        v.emplace_back(2);
        FastType::move_count = 0;
        FastType::copy_count = 0;

        v.emplace_back(3);  // Triggers reallocation!
        std::cout << "After realloc: moves=" << FastType::move_count
                  << ", copies=" << FastType::copy_count << "\n";
        // Expected: moves=2, copies=0 (existing elements MOVED to new buffer)
    }

    std::cout << "\n=== SlowType (throwing move) ===\n";
    {
        std::vector<SlowType> v;
        v.reserve(2);
        v.emplace_back(1);
        v.emplace_back(2);
        SlowType::move_count = 0;
        SlowType::copy_count = 0;

        v.emplace_back(3);  // Triggers reallocation!
        std::cout << "After realloc: moves=" << SlowType::move_count
                  << ", copies=" << SlowType::copy_count << "\n";
        // Expected: moves=0, copies=2 (existing elements COPIED for safety)
    }

    return 0;
}
```

**Output:**

```text
=== FastType (noexcept move) ===
After realloc: moves=2, copies=0

=== SlowType (throwing move) ===
After realloc: moves=0, copies=2
```

**How this works:**

- `FastType` has `noexcept` move - vector moves existing elements during reallocation (0 copies)
- `SlowType` has potentially-throwing move - vector falls back to copying (0 moves)
- The difference is entirely determined by `is_nothrow_move_constructible_v<T>`
- This can have **massive** performance impact for types with expensive copies (e.g., types containing `vector`, `string`, etc.)

### Q2: Write a move constructor that is conditionally `noexcept` based on member nothrow properties

The `noexcept(expression)` syntax lets you propagate `noexcept` status automatically rather than hard-coding it. If any member's move might throw, the whole class correctly reports it might throw too:

```cpp
#include <iostream>
#include <type_traits>
#include <vector>
#include <string>
#include <memory>

// Generic container with conditionally noexcept move
template<typename T, typename Alloc = std::allocator<T>>
struct Container {
    T* data = nullptr;
    std::size_t size = 0;
    Alloc alloc;

    Container() = default;

    // Conditionally noexcept: noexcept if BOTH T's move AND Alloc's move are noexcept
    Container(Container&& other) noexcept(
        std::is_nothrow_move_constructible_v<Alloc>
        // We move the pointer, not T objects, so T's move doesn't matter here
        // But for illustration:
        // && std::is_nothrow_move_constructible_v<T>
    )
        : data(std::exchange(other.data, nullptr))
        , size(std::exchange(other.size, 0))
        , alloc(std::move(other.alloc))
    {}

    Container& operator=(Container&& other) noexcept(
        std::is_nothrow_move_assignable_v<Alloc>
    ) {
        if (this != &other) {
            // Deallocate current...
            data = std::exchange(other.data, nullptr);
            size = std::exchange(other.size, 0);
            alloc = std::move(other.alloc);
        }
        return *this;
    }

    ~Container() {
        // Deallocate...
    }
};

// A wrapper that propagates noexcept from all members
template<typename A, typename B>
struct Pair {
    A first;
    B second;

    Pair(Pair&& other) noexcept(
        std::is_nothrow_move_constructible_v<A> &&
        std::is_nothrow_move_constructible_v<B>
    )
        : first(std::move(other.first))
        , second(std::move(other.second))
    {}

    Pair& operator=(Pair&&) = default;
    Pair() = default;
    Pair(A a, B b) : first(std::move(a)), second(std::move(b)) {}
};

// Struct with potentially throwing member
struct MayThrow {
    MayThrow() = default;
    MayThrow(MayThrow&&) {}  // NOT noexcept
};

int main() {
    // Container: noexcept if allocator is nothrow-moveable
    static_assert(std::is_nothrow_move_constructible_v<Container<int>>,
                  "Container<int> should be nothrow moveable with default allocator");

    // Pair<int, string>: both members are nothrow moveable
    static_assert(std::is_nothrow_move_constructible_v<Pair<int, std::string>>,
                  "Pair<int, string> should be nothrow moveable");

    // Pair<int, MayThrow>: MayThrow's move may throw
    static_assert(!std::is_nothrow_move_constructible_v<Pair<int, MayThrow>>,
                  "Pair<int, MayThrow> should NOT be nothrow moveable");

    // This affects vector performance:
    std::cout << "Pair<int, string> -> vector will MOVE during realloc: "
              << std::boolalpha
              << std::is_nothrow_move_constructible_v<Pair<int, std::string>> << "\n";

    std::cout << "Pair<int, MayThrow> -> vector will COPY during realloc: "
              << !std::is_nothrow_move_constructible_v<Pair<int, MayThrow>> << "\n";

    return 0;
}
```

**Output:**

```text
Pair<int, string> -> vector will MOVE during realloc: true
Pair<int, MayThrow> -> vector will COPY during realloc: true
```

**How this works:**

- `noexcept(expression)` computes whether all sub-expressions are noexcept
- The move constructor is `noexcept` only if ALL member moves are `noexcept`
- Use `std::is_nothrow_move_constructible_v<MemberType>` in the `noexcept` specifier
- This propagation ensures `std::vector` can safely move your type during reallocation
- Standard library types like `std::string`, `std::vector` all provide `noexcept` moves

### Q3: Explain why the strong exception safety guarantee forces this conditional behavior

The **strong exception guarantee** promises:

> If an operation throws, the program state is unchanged - as if the operation never happened.

**Why this forces conditional move/copy:**

**Scenario:** `vector::push_back` triggers reallocation (size == capacity):

```cpp
// Step 1: Allocate new buffer (bigger)
// Step 2: Transfer elements from old buffer to new buffer
// Step 3: Destroy old buffer elements
// Step 4: Deallocate old buffer
```

**If Step 2 uses MOVE and element N's move throws:**

```cpp
// Old buffer:  [A_moved  B_moved  C_moved  D ... ]  <- A,B,C are moved-from (destroyed)
// New buffer:  [A        B        C        FAIL  ]  <- exception during D's move

// Problem: We can't restore old buffer because A,B,C are moved-from!
//          Strong guarantee VIOLATED.
//          Even if we catch the exception, old data is lost.
```

**If Step 2 uses COPY and element N's copy throws:**

```cpp
// Old buffer:  [A  B  C  D  E  F]  <- ALL original objects still intact (we copied, not moved)
// New buffer:  [A  B  C  FAIL   ]  <- exception during D's copy

// Solution: Destroy partial new buffer, deallocate it, and keep old buffer.
//           Container state is exactly as before push_back.
//           Strong guarantee PRESERVED
```

**The conditional decision:**

```cpp
#include <iostream>
#include <type_traits>
#include <utility>

template<typename T>
void demonstrate_move_if_noexcept(T& obj) {
    // This is what vector internally uses:
    // std::move_if_noexcept(x) returns:
    //   - T&&         if T is nothrow-move-constructible (move it, safe)
    //   - const T&    if T is copy-constructible but move may throw (copy it, safe)
    //   - T&&         if T is move-only and may throw (no choice, move anyway)

    using result_type = decltype(std::move_if_noexcept(obj));

    if constexpr (std::is_rvalue_reference_v<result_type>) {
        std::cout << "  -> Will MOVE (noexcept or move-only)\n";
    } else {
        std::cout << "  -> Will COPY (for safety)\n";
    }
}

struct NoexceptMove {
    NoexceptMove() = default;
    NoexceptMove(NoexceptMove&&) noexcept = default;
    NoexceptMove(const NoexceptMove&) = default;
};

struct ThrowingMove {
    ThrowingMove() = default;
    ThrowingMove(ThrowingMove&&) {}          // may throw
    ThrowingMove(const ThrowingMove&) = default;
};

struct MoveOnly {
    MoveOnly() = default;
    MoveOnly(MoveOnly&&) {}                  // may throw
    MoveOnly(const MoveOnly&) = delete;      // can't copy!
};

int main() {
    NoexceptMove a;
    ThrowingMove b;
    MoveOnly c;

    std::cout << "NoexceptMove (safe to move):";
    demonstrate_move_if_noexcept(a);  // -> MOVE

    std::cout << "ThrowingMove (unsafe, can copy):";
    demonstrate_move_if_noexcept(b);  // -> COPY (for strong guarantee)

    std::cout << "MoveOnly (unsafe, must move):";
    demonstrate_move_if_noexcept(c);  // -> MOVE (no choice)

    std::cout << "\nConclusion:\n";
    std::cout << "  The strong exception guarantee requires copying when move\n";
    std::cout << "  may throw, because a failed move corrupts the source.\n";
    std::cout << "  Mark your move constructors noexcept to get move performance!\n";

    return 0;
}
```

**Output:**

```text
NoexceptMove (safe to move):  -> Will MOVE (noexcept or move-only)
ThrowingMove (unsafe, can copy):  -> Will COPY (for safety)
MoveOnly (unsafe, must move):  -> Will MOVE (noexcept or move-only)

Conclusion:
  The strong exception guarantee requires copying when move
  may throw, because a failed move corrupts the source.
  Mark your move constructors noexcept to get move performance!
```

---

## Notes

- **Always mark move constructors and move assignment operators `noexcept`** unless they genuinely might throw. This is the single most impactful `noexcept` annotation for performance.
- The implicitly-generated move constructor is `noexcept` if all members and bases have `noexcept` moves. If even ONE member has a potentially-throwing move, the whole thing loses `noexcept`.
- `std::move_if_noexcept` is the utility function that implements this logic. It returns an rvalue reference for noexcept types, or a const lvalue reference otherwise.
- Other containers (`deque`, `vector`-based containers) and algorithms (`std::swap`, `std::sort`) also benefit from `noexcept` moves.
- Use `static_assert(std::is_nothrow_move_constructible_v<MyType>)` in your code to catch accidental `noexcept` loss early.
