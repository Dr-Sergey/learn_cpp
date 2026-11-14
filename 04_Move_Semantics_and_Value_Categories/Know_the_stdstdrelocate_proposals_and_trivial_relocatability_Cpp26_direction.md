# Know the std::relocate proposals and trivial relocatability (C++26 direction)

**Category:** Move Semantics and Value Categories  
**Standard:** Not in C++26 (adopted into the draft, then removed)  
**Reference:** <https://wg21.link/P2786>  

---

## Topic Overview

**Relocation** = move-construct + destroy-source, combined into one operation. For many types ("trivially relocatable" types), this combined operation can be optimized down to a simple `memcpy`. The payoff is huge for containers like `std::vector`: instead of calling N move constructors and N destructors during reallocation, you do a single bulk memory copy.

### Where the Standard Stands

This is the part most write-ups get wrong, so it comes first. Two competing designs went through the committee:

| Paper | Opt-in spelling | Library surface |
| --- | --- | --- |
| P1144 (O'Dwyer) | `[[trivially_relocatable]]` attribute | `std::relocate_at`, `std::uninitialized_relocate` |
| P2786 (Bloomberg) | `trivially_relocatable_if_eligible` contextual keyword after the class name | `std::trivially_relocate`, `std::relocate`, `std::is_trivially_relocatable` |

P2786 was voted into the C++26 working draft in February 2025. In November 2025 the committee found a bug it judged could not be fixed in time and **removed the feature from C++26**. The standard that shipped has no trivial relocation: no trait, no keyword, no `std::relocate`. Work continues for a later revision, and the final design may differ from both papers.

So everything below is either a library technique you can use today or a description of a proposal - not standard C++.

### What is Trivial Relocatability

The key question is: can we replace "move the object + destroy the old location" with a plain byte copy? For most types the answer is yes.

<!-- compile: fragment: illustrative type definitions -->
```cpp
#include <cstring>
#include <memory>

// A type is trivially relocatable if moving it + destroying the source
// is equivalent to memcpy.

struct TriviallyRelocatable {
    int x;
    double y;
    std::unique_ptr<int> ptr;  // unique_ptr is trivially relocatable!
    // Move = copy bytes, destroy source = no-op
    // So: relocate = memcpy
};

// NOT trivially relocatable:
struct SelfReferential {
    int data;
    int* self_ptr = &data;  // Points to itself!
    // After memcpy, self_ptr points to OLD location - broken!
};
```

The reason `unique_ptr` is trivially relocatable despite holding a pointer is that after the `memcpy` the pointer value is still correct - it points to the same heap allocation. The old location is simply abandoned without running a destructor, which is fine because the destructor would have been a no-op anyway (the moved-from `unique_ptr` holds null). Self-referential types break this because the pointer value itself encodes the old address.

Note that today's language gives this no blessing: `memcpy`-ing an object that is not trivially copyable and then using the copy is undefined behavior as far as the standard is concerned. Libraries that do it rely on the compiler behaving sensibly, which in practice it does.

### Why It Matters for vector

Right now, when `std::vector` needs more capacity, it pays a per-element cost in move constructors and destructors. With trivial relocatability, that entire cost collapses to one call.

<!-- compile: fragment: commentary only -->
```cpp
// vector::push_back with reallocation currently:
// 1. Allocate new buffer
// 2. Move-construct each element to new buffer  (N move constructors)
// 3. Destroy each element in old buffer          (N destructors)
// 4. Deallocate old buffer

// With trivial relocatability:
// 1. Allocate new buffer
// 2. memcpy(new_buf, old_buf, N * sizeof(T))   (ONE memory operation)
// 3. Deallocate old buffer (no destructors needed)
// The gain depends on the element type: large for types such as
// unique_ptr or string, nothing for types that were already trivially copyable.
```

### The Shape of the Proposed API

Both proposals pair a trait with a relocate operation that falls back to move-then-destroy. The sketch below uses a placeholder trait, because no standard one exists; substitute your library's (see Q3).

<!-- compile: fragment: sketch against a placeholder trait -->
```cpp
// Placeholder: stands in for folly::IsRelocatable, absl::is_trivially_relocatable,
// or a future standard trait.
template<typename T>
inline constexpr bool is_trivially_relocatable_v = /* library- or compiler-determined */;

// Relocate = move + destroy, optimized when possible:
template<typename T>
T* relocate_at(T* source, T* dest) {
    if constexpr (is_trivially_relocatable_v<T>) {
        std::memcpy(dest, source, sizeof(T));
    } else {
        std::construct_at(dest, std::move(*source));
        std::destroy_at(source);
    }
    return dest;
}
```

---

## Self-Assessment

### Q1: Which standard library types are trivially relocatable

In practice: `unique_ptr`, `shared_ptr`, `weak_ptr`, `vector`, and wrappers such as `optional`, `variant`, `pair` and `tuple` whenever everything they hold is. Anything whose representation points back into itself is not.

That makes the answer implementation-specific for some familiar types. libstdc++'s `std::string` keeps a pointer to its own small-string buffer, so it is **not** trivially relocatable, while the libc++ and MSVC strings store no self-pointer and are. libstdc++'s `std::list` embeds its sentinel node in the list object, with the first and last nodes pointing back at it, so it is not either. Never assume; ask the trait your library provides.

### Q2: Why can't the compiler auto-detect trivial relocatability

For a type that follows the Rule of Zero it can, and both proposals say so: if every base and member is trivially relocatable and the move operations and destructor are implicit or defaulted, the class is too.

The problem is a type with a user-provided move constructor or destructor. The compiler cannot tell whether that code merely transfers ownership (like `unique_ptr`, safe to `memcpy`) or depends on the object's address - storing `this` in a registry, keeping an interior pointer, holding a mutex other threads are waiting on. Only the author knows, so those types need an explicit opt-in. That opt-in is the spelling the two proposals disagreed on.

### Q3: How do existing libraries handle this

Folly uses the `folly::IsRelocatable<T>` trait, and `fbvector` reallocates with `memcpy` when it holds. Abseil has `absl::is_trivially_relocatable`. BDE (Bloomberg) uses `bslmf::IsBitwiseMoveable`. Qt uses `Q_DECLARE_TYPEINFO(T, Q_RELOCATABLE_TYPE)`. Clang provides the `__is_trivially_relocatable` builtin and the `[[clang::trivial_abi]]` attribute, and libc++ uses that knowledge inside `std::vector`. All of these pre-date the standard proposals and demonstrate that the optimization is real and valuable in production.

---

## Notes

- Trivial relocation is **not** part of C++26. P2786 was adopted into the working draft in February 2025 and removed in November 2025.
- Until it is standardized, use a library trait (Folly, Abseil, BDE, Qt) and treat `memcpy` relocation as a deliberate, tested reliance on implementation behavior.
- Rule-of-Zero types compose: they are trivially relocatable exactly when their members are.
- Trivial relocatability of standard types varies by implementation - `std::string` is the classic example.
- The main beneficiary is `std::vector` - reallocation becomes a single `memcpy` instead of N move + N destroy.
