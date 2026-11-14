# Understand Trivially Relocatable Types and memcpy-Based Relocation

**Category:** Move Semantics and Value Categories  
**Standard:** Not version-specific (library technique; not in C++26)  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p1144r9.html>  

---

## Topic Overview

### What Is Trivial Relocation

**Relocation** = move-construct at new location + destroy at old location. A type is **trivially relocatable** if this relocation is equivalent to a `memcpy` - no custom move/destroy logic needed. The practical payoff is that `std::vector` can reallocate its buffer with a single `memcpy` instead of calling a move constructor and destructor on every element.

```text
Normal relocation:          Trivial relocation:

1. Move-construct at dest   1. memcpy from src to dest
2. Destroy at source        (done - no destroy needed)
```

| Type | Trivially relocatable? | Why |
| --- | :---: | --- |
| `int`, `double` | Yes | Trivially copyable |
| `std::unique_ptr<T>` | Yes | Move = pointer copy, destroy = check null |
| `std::vector<T>` | Yes | Move = steal three pointers, destroy = check null |
| `std::string` | Depends | Yes in libc++ and MSVC; **no** in libstdc++, whose string points into its own small-string buffer |
| `std::list<T>` | Depends | No in libstdc++, where the end nodes point back at a sentinel stored inside the list object |
| Self-referential types | **No** | Internal pointers become invalid after memcpy |

"Yes" here means "in every mainstream implementation", not "guaranteed by the standard" - the standard has no such notion.

### Current Status

Trivial relocation is **not standardized**. P2786 was voted into the C++26 working draft in February 2025 and removed again in November 2025 after a late design bug, so C++26 shipped without it; P1144 is the competing design. In the meantime:

- libc++ and libstdc++ use internal traits to turn `std::vector` reallocation into `memcpy`/`memmove` for types they know are safe
- Clang has the `__is_trivially_relocatable` builtin and the `[[clang::trivial_abi]]` attribute
- Folly, Abseil, BDE and Qt each ship their own opt-in trait
- Many user types are trivially relocatable in practice

---

## Self-Assessment

### Q1: Explain what trivially relocatable means - move + destroy is equivalent to memcpy

The key insight is what happens when you move a `std::unique_ptr`: the move steals the pointer and nulls the source, then the destructor on the source sees null and does nothing. The net effect is identical to just copying the raw bytes and forgetting the source. That's why `memcpy` is sufficient.

The example deliberately holds its string through a `unique_ptr`. A bare `std::string` member would make the demo work on libc++ and MSVC but crash on libstdc++, whose small-string representation points into itself.

```cpp
#include <iostream>
#include <cstring>
#include <memory>
#include <new>
#include <string>

// A trivially relocatable type: move + destroy == memcpy
struct TriviallyRelocatable {
    std::unique_ptr<std::string> name;  // one pointer to the heap, none into itself
    int value;

    TriviallyRelocatable(std::string n, int v)
        : name(std::make_unique<std::string>(std::move(n))), value(v) {}

    // Move: steals name's pointer -> source.name = nullptr
    // Destroy source: ~unique_ptr on nullptr -> no-op
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

    // Build the object in raw storage so that WE decide when its destructor runs.
    alignas(TriviallyRelocatable) unsigned char old_storage[sizeof(TriviallyRelocatable)];
    alignas(TriviallyRelocatable) unsigned char new_storage[sizeof(TriviallyRelocatable)];

    auto* original = new (old_storage) TriviallyRelocatable("hello", 42);
    std::cout << "Original:  name=\"" << *original->name << "\", value=" << original->value << "\n";

    // Relocate by copying bytes. From here on old_storage is abandoned:
    // no destructor runs on it, exactly as a relocating vector would do.
    // WARNING: the standard does not bless this for a non-trivially-copyable
    // type - it is what relocation-aware libraries rely on in practice.
    std::memcpy(new_storage, old_storage, sizeof(TriviallyRelocatable));
    auto* relocated = std::launder(reinterpret_cast<TriviallyRelocatable*>(new_storage));

    std::cout << "Relocated: name=\"" << *relocated->name << "\", value=" << relocated->value << "\n";
    std::cout << "Storage moved: " << (static_cast<void*>(original) != static_cast<void*>(relocated)) << "\n\n";

    // Show why SelfReferential is NOT trivially relocatable
    std::cout << "=== Non-trivially relocatable (self-referential) ===\n";
    SelfReferential sr(99);
    std::cout << "sr.data = " << sr.data << "\n";
    std::cout << "self_ptr == &data: " << (sr.self_ptr == &sr.data) << "\n";
    std::cout << "After memcpy, self_ptr would still point to old address -> DANGLING!\n";

    // One live object, one destructor call: the string is freed exactly once.
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
    std::cout << "  Gain:       one bulk copy instead of n move + n destroy calls\n";

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
    std::cout << "  - Library traits: folly::IsRelocatable, absl::is_trivially_relocatable\n";
    std::cout << "  - Clang: __is_trivially_relocatable builtin, [[clang::trivial_abi]]\n";
    std::cout << "  - Standard trait: none yet (proposed in P1144 and P2786)\n";

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
    std::cout << "  Containers:         vector<T>, string*, deque<T>*\n";
    std::cout << "  Utilities:          optional<T>, variant<T...>, pair<A,B>\n";
    std::cout << "                      (when the types they hold are)\n";
    std::cout << "  * = implementation-dependent: libstdc++'s string is NOT\n\n";

    std::cout << "NOT TRIVIALLY RELOCATABLE:\n";
    std::cout << "  list<T>* (libstdc++): sentinel inside the object, nodes point at it\n";
    std::cout << "  function<F>*: small-buffer storage may hold a non-relocatable callable\n";
    std::cout << "  Any type with self-referencing pointers\n";
    std::cout << "  Types registered in global registries (ctor/dtor do registration)\n\n";

    // Why user types are OFTEN trivially relocatable:
    std::cout << "=== Why user types are usually trivially relocatable ===\n\n";

    std::cout << "A Rule-of-Zero type is trivially relocatable if ALL its members are.\n\n";

    struct UserType {
        std::string name;         // Trivially relocatable (not on libstdc++)
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

    std::cout << "Most user-defined types are trivially relocatable, because most\n";
    std::cout << "are just aggregates of containers, smart pointers, and scalars.\n";

    return 0;
}
```

The takeaway: if your type's members are all standard containers, smart pointers, or scalars, and it declares no move operations or destructor of its own, it is very likely trivially relocatable already. The exceptions are a type that stores a pointer to one of its own members, and a member whose implementation does that for you - `std::string` on libstdc++ being the one that surprises people.

---

## Notes

- Trivially relocatable = `move + destroy == memcpy`. Not standardized: P2786 was adopted for C++26 and then removed before publication.
- Many standard types (vector, unique_ptr, shared_ptr) are trivially relocatable in practice; `std::string` and `std::list` depend on the implementation.
- `std::vector` can use `memcpy`/`memmove` for bulk reallocation of trivially relocatable types instead of n moves and n destructor calls.
- Self-referential types (internal pointers to own members) are **not** trivially relocatable.
- Clang provides the `__is_trivially_relocatable` builtin and `[[clang::trivial_abi]]`; libc++ and libstdc++ optimize internally for types they can prove safe.
- Every trivially copyable type is trivially relocatable, but not the other way round.
- Byte-copying a non-trivially-copyable object is formally undefined behavior today; relocation-aware libraries rely on it working in practice.
