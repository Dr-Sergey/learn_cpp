# Understand Trivially Relocatable Types and memcpy-Based Relocation

**Category:** Move Semantics and Value Categories  
**Standard:** Not version-specific  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p1144r9.html>  

---

## Topic Overview

### What Is Trivial Relocation

**Relocation** = move-construct at new location + destroy at old location. A type is **trivially relocatable** if this relocation is equivalent to a `memcpy` - no custom move/destroy logic needed. The practical payoff is that `std::vector` can reallocate its buffer with a single `memcpy` instead of calling a move constructor and destructor on every element.

```cpp
Normal relocation:          Trivial relocation:

1. Move-construct at dest   1. memcpy from src to dest
2. Destroy at source        (done - no destroy needed)
```

| Type | Trivially relocatable? | Why |
| --- | :---: | --- |
| `int`, `double` | Yes | Trivially copyable |
| `std::string` | Yes* | Move = pointer swap, destroy = check null (*implementation-dependent) |
| `std::vector<T>` | Yes* | Move = pointer swap (*not yet standardized) |
| `std::unique_ptr<T>` | Yes* | Move = pointer copy, destroy = check null |
| `std::list<T>` | **No** | Nodes have back-pointers to the list head |
| Self-referential types | **No** | Internal pointers become invalid after memcpy |

### Current Status

Trivial relocation is **not yet standardized** (P1144 is proposed). However:

- Most standard library implementations already optimize for it internally
- Clang has `[[clang::trivially_relocatable]]`
- Many user types are trivially relocatable in practice

---

## Self-Assessment

### Q1: Explain what trivially relocatable means - move + destroy is equivalent to memcpy

The key insight is what happens when you move a `std::string`: the move steals the internal pointer, then the destructor on the source checks if the pointer is null and does nothing. The net effect is identical to just copying the raw bytes. That's why `memcpy` is sufficient.

```cpp
#include <iostream>
#include <cstring>
#include <string>
#include <memory>
#include <type_traits>

// A trivially relocatable type: move + destroy == memcpy
struct TriviallyRelocatable {
    std::string name;    // string internals: pointer + size + capacity
    int value;

    TriviallyRelocatable(std::string n, int v)
        : name(std::move(n)), value(v) {}

    // Move: steals name's pointer -> source.name = {nullptr, 0, 0}
    // Destroy source: ~string on {nullptr, 0, 0} -> no-op
    // The net effect is just: copy bytes from old to new location
    // This means memcpy achieves the same result!
};

// A NON-trivially-relocatable type: has self-referencing pointer
struct SelfReferential {
    int data;
    int* self_ptr;  // Points to &data - becomes dangling after memcpy!

    SelfReferential(int d) : data(d), self_ptr(&data) {}

    SelfReferential(SelfReferential&& other) noexcept
        : data(other.data)
        , self_ptr(&data)  // Must re-point to OUR data, not other's
    {}

    // memcpy would copy the old self_ptr value -> points to old (freed) location!
};

int main() {
    std::cout << "=== Trivially Relocatable ===\n\n";

    std::cout << "Trivially relocatable means:\n";
    std::cout << "  move_construct(dest, src) + destroy(src)\n";
    std::cout << "  == memcpy(dest, src, sizeof(T))\n\n";

    // Show that memcpy produces the correct result for TriviallyRelocatable
    TriviallyRelocatable original("hello", 42);
    std::cout << "Original: name=\"" << original.name << "\", value=" << original.value << "\n";
    std::cout << "Original addr: " << &original << "\n\n";

    // Simulate memcpy-based relocation
    alignas(TriviallyRelocatable) char buffer[sizeof(TriviallyRelocatable)];
    std::memcpy(buffer, &original, sizeof(TriviallyRelocatable));
    // WARNING: This is for demonstration only - not valid C++ without start_lifetime_as

    auto* relocated = reinterpret_cast<TriviallyRelocatable*>(buffer);
    std::cout << "Relocated: name=\"" << relocated->name << "\", value=" << relocated->value << "\n";
    std::cout << "Relocated addr: " << relocated << "\n\n";

    // Show why SelfReferential is NOT trivially relocatable
    std::cout << "=== Non-trivially relocatable (self-referential) ===\n";
    SelfReferential sr(99);
    std::cout << "sr.data = " << sr.data << "\n";
    std::cout << "sr.self_ptr points to: " << sr.self_ptr << " (&data = " << &sr.data << ")\n";
    std::cout << "After memcpy, self_ptr would still point to old address -> DANGLING!\n";

    // Clean up: destroy original manually since we memcpy'd its guts
    // (In real code, use proper move + destroy)
    std::memset(&original, 0, sizeof(original));

    // Properly destroy the relocated object
    relocated->~TriviallyRelocatable();

    return 0;
}
```

The `SelfReferential` example shows exactly why this property matters: `self_ptr` stores the address of `data` within the same object. After a `memcpy` to a new location, `self_ptr` still holds the old address - a dangling pointer. The move constructor has to fix it up manually, so `memcpy` alone isn't safe.

### Q2: Show why `std::vector` can use `memmove` for trivially relocatable types during reallocation

When `std::vector` runs out of capacity and has to move all elements to a new buffer, each element normally goes through move-construct + destroy. For trivially relocatable types, that entire loop collapses into a single `memcpy` call.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <chrono>
#include <cstring>

struct Tracked {
    int id;
    static int move_count;
    static int destroy_count;

    Tracked(int i) : id(i) {}
    Tracked(Tracked&& other) noexcept : id(other.id) { ++move_count; }
    ~Tracked() { ++destroy_count; }

    static void reset() { move_count = 0; destroy_count = 0; }
};
int Tracked::move_count = 0;
int Tracked::destroy_count = 0;

int main() {
    std::cout << "=== Vector Reallocation: move+destroy vs memcpy ===\n\n";

    // Normal reallocation: move each element + destroy old
    std::cout << "1. Normal reallocation (non-trivial type):\n";
    {
        Tracked::reset();
        std::vector<Tracked> v;
        v.reserve(4);
        for (int i = 0; i < 4; ++i) v.emplace_back(i);

        Tracked::reset();
        v.emplace_back(4);  // Triggers reallocation

        std::cout << "  Elements: 5, Moves: " << Tracked::move_count
                  << ", Destroys: " << Tracked::destroy_count << "\n";
        std::cout << "  Cost: O(n) moves + O(n) destroys per reallocation\n";
    }

    // For trivially relocatable types, vector CAN do:
    //   1. Allocate new buffer
    //   2. memcpy(new_buf, old_buf, n * sizeof(T))  <- single bulk operation!
    //   3. Free old buffer (no element-wise destruction needed)
    // This is dramatically faster for large vectors

    std::cout << "\n2. Trivially relocatable optimization:\n";
    std::cout << "  Instead of: for each element { move(new, old); destroy(old); }\n";
    std::cout << "  Just:       memcpy(new_buf, old_buf, n * sizeof(T))\n";
    std::cout << "  Speedup:    ~10x for large vectors (one memcpy vs n operations)\n";

    // Benchmark: vector<int> (trivially copyable, trivially relocatable)
    constexpr int N = 10'000'000;
    {
        auto t1 = std::chrono::high_resolution_clock::now();
        std::vector<int> v;
        for (int i = 0; i < N; ++i) v.push_back(i);
        auto t2 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
        std::cout << "\n3. vector<int> push_back " << N << ": " << ms << " ms\n";
        std::cout << "  (Compiler can use memcpy for reallocation - very fast)\n";
    }

    std::cout << "\n=== How libraries detect trivial relocation ===\n";
    std::cout << "  - is_trivially_copyable -> always trivially relocatable\n";
    std::cout << "  - Custom trait: is_trivially_relocatable<T>\n";
    std::cout << "  - Clang: [[clang::trivially_relocatable]] attribute\n";
    std::cout << "  - P1144 proposal: std::is_trivially_relocatable type trait\n";

    return 0;
}
```

### Q3: List standard types that are trivially relocatable and why user types often are too

The good news is that most types you write day-to-day are trivially relocatable, because they're composed entirely of types that are themselves trivially relocatable.

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <vector>
#include <memory>
#include <map>
#include <unordered_map>
#include <list>
#include <deque>

int main() {
    std::cout << "=== Trivially Relocatable Standard Types ===\n\n";

    std::cout << "TRIVIALLY RELOCATABLE (in practice, most implementations):\n";
    std::cout << "  Fundamental types:  int, double, char, pointers\n";
    std::cout << "  Smart pointers:     unique_ptr<T>, shared_ptr<T>\n";
    std::cout << "  Containers:         vector<T>, string, deque<T>*\n";
    std::cout << "  Utilities:          optional<T>, variant<T...>, pair<A,B>\n";
    std::cout << "  Functional:         function<F> (most impls)\n";
    std::cout << "  * = implementation-dependent\n\n";

    std::cout << "NOT TRIVIALLY RELOCATABLE:\n";
    std::cout << "  list<T>:         sentinel node has pointers to itself\n";
    std::cout << "  Any type with self-referencing pointers\n";
    std::cout << "  Types registered in global registries (ctor/dtor do registration)\n\n";

    // Why user types are OFTEN trivially relocatable:
    std::cout << "=== Why user types are usually trivially relocatable ===\n\n";

    std::cout << "A type is trivially relocatable if ALL its members are.\n\n";

    struct UserType {
        std::string name;         // Trivially relocatable
        std::vector<int> data;    // Trivially relocatable
        std::unique_ptr<int> ptr; // Trivially relocatable
        int flags;                // Trivially copyable
        // -> UserType is trivially relocatable!
    };

    struct AlsoRelocatable {
        std::map<std::string, int> config;  // Relocatable
        double value;                        // Trivially copyable
    };

    struct NotRelocatable {
        int data;
        int* self_ptr;  // Points to &data -> NOT relocatable

        NotRelocatable() : data(0), self_ptr(&data) {}
        NotRelocatable(NotRelocatable&& o) noexcept
            : data(o.data), self_ptr(&data) {}  // Must fix pointer!
    };

    std::cout << "Common user type pattern:\n";
    std::cout << "  struct Config {\n";
    std::cout << "      string name;     // relocatable\n";
    std::cout << "      vector<T> items; // relocatable\n";
    std::cout << "      int count;       // trivially copyable\n";
    std::cout << "  };\n";
    std::cout << "  -> Trivially relocatable (all members are)\n\n";

    std::cout << "Rare non-relocatable pattern:\n";
    std::cout << "  struct Node {\n";
    std::cout << "      Data data;\n";
    std::cout << "      Node* self;  // points to &data -> breaks after memcpy\n";
    std::cout << "  };\n\n";

    std::cout << "Estimate: >95% of user-defined types are trivially relocatable\n";
    std::cout << "because most types are just aggregates of strings, vectors,\n";
    std::cout << "smart pointers, and scalars - all of which are relocatable.\n";

    return 0;
}
```

The takeaway: if your type's members are all standard containers, smart pointers, or scalars, you're almost certainly trivially relocatable already. The rare exception is a type that stores a pointer to one of its own members.

---

## Notes

- Trivially relocatable = `move + destroy == memcpy`. Not yet standardized (P1144).
- Most standard types (string, vector, unique_ptr, shared_ptr) are trivially relocatable in practice.
- `std::vector` can use `memcpy`/`memmove` for bulk reallocation of trivially relocatable types - ~10x faster.
- Self-referential types (internal pointers to own members) are **not** trivially relocatable.
- Clang provides `[[clang::trivially_relocatable]]`; MSVC and GCC optimize internally without an attribute.
- `is_trivially_copyable` is a subset of `is_trivially_relocatable` - all trivially copyable types are trivially relocatable.
