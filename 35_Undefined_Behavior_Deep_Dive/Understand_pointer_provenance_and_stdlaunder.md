# Understand Pointer Provenance and std::launder

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17, C++20, C++23  
**Reference:** [cppreference - std::launder](https://en.cppreference.com/w/cpp/utility/launder)  

---

## Topic Overview

Pointer provenance is the concept that a pointer carries invisible metadata about **which object it was derived from**. The compiler uses this provenance information to perform optimizations: if a pointer was obtained from object A, the compiler assumes it cannot access object B, even if A and B happen to occupy the same memory address.

This becomes critical in two scenarios:

1. **Placement new** over an existing object: the old pointer's provenance points to the destroyed object, not the new one.
2. **`const` or reference members**: the compiler may cache values through the old pointer, not knowing the object was replaced.

`std::launder` (C++17) is a "provenance barrier" - it takes a pointer whose provenance is outdated and returns a new pointer with correct provenance to the object currently living at that address.

Here is a summary of when provenance issues arise and how to fix them:

| Scenario | Provenance Issue | Fix |
| --- | --- | --- |
| Placement new, no const/ref members | Compiler may cache old object's values | `std::launder` or use returned pointer |
| Placement new over `const` member | Compiler assumes `const` member unchanged | `std::launder` required |
| Placement new over reference member | Reference target assumed stable | `std::launder` required |
| `reinterpret_cast` round-trip | May lose provenance through cast chain | `std::launder` restores it |
| Aligned storage + placement new | Original storage has no object provenance | Use returned pointer from `new` |

The step-by-step picture below shows what happens to a pointer's provenance when placement new replaces an object:

```cpp
Step 1: Object A lives at address 0x1000
        ptr -> Object A (provenance: A)

Step 2: Placement new creates Object B at 0x1000
        ptr still has provenance: A (stale!)
        Compiler may optimize as if A still exists.

Step 3: std::launder(ptr) -> new_ptr
        new_ptr has provenance: B (correct!)
        Compiler now sees the current object.
```

One important detail: `std::launder` does NOT change the pointer value - it returns the same address. It is a compile-time directive that tells the optimizer to discard cached assumptions about what lives at that address.

---

## Self-Assessment

### Q1: Show why placement new over an object with const members requires std::launder

The reason `const` members make this especially tricky is that the compiler is explicitly allowed to cache a `const` member's value. After all, the standard says a `const` object cannot change - so why reload it? The problem is that placement new creates a *new* object at the same address, and the old pointer does not know that. Let's see this in action:

```cpp
#include <cstdio>
#include <memory>
#include <new>

struct Widget {
    const int id;     // const member - compiler may cache this
    double value;

    Widget(int i, double v) : id(i), value(v) {}
};

void demonstrate_provenance_issue() {
    // Allocate aligned storage
    alignas(Widget) unsigned char buf[sizeof(Widget)];

    // Construct Widget with id=1
    Widget* w1 = new (buf) Widget(1, 3.14);
    std::printf("Original: id=%d, value=%.2f\n", w1->id, w1->value);

    // Destroy and reconstruct with id=2
    w1->~Widget();
    Widget* w2 = new (buf) Widget(2, 2.71);

    // BUG: using w1 after placement new
    // The compiler may assume w1->id is still 1 (const member -> cached)
    // std::printf("Via old ptr: id=%d\n", w1->id);  // UB!

    // CORRECT: use the pointer returned by placement new
    std::printf("Via new ptr: id=%d, value=%.2f\n", w2->id, w2->value);

    // Alternative: std::launder refreshes provenance
    Widget* w3 = std::launder(reinterpret_cast<Widget*>(buf));
    std::printf("Via launder: id=%d, value=%.2f\n", w3->id, w3->value);

    w2->~Widget();  // Clean up
}

// Why does the compiler cache const members?
// Because the standard says modifying a const object is UB.
// If w1->id is const and w1 points to a Widget, the compiler
// is allowed to assume w1->id never changes for the lifetime of *w1.
// Placement new creates a NEW object, but w1's provenance
// still refers to the OLD object whose const member was 1.

int main() {
    demonstrate_provenance_issue();
}
```

When a `const` member exists, the compiler may cache its value through any pointer that was obtained from the original object. After placement new creates a new object at the same address, the old pointer's provenance refers to the destroyed object. `std::launder` or the pointer returned by `new` must be used to access the new object.

---

### Q2: Demonstrate std::launder with reference members and array storage

Reference members have the same problem as `const` members - the compiler assumes the reference target is stable for the lifetime of the object. When you replace the object via placement new, the old pointer still thinks it knows where the reference points. Here is how it plays out, along with a related array provenance issue:

```cpp
#include <cassert>
#include <cstdio>
#include <new>

struct RefHolder {
    int& ref;
    int data;

    RefHolder(int& r, int d) : ref(r), data(d) {}
};

void reference_member_provenance() {
    int target_a = 10;
    int target_b = 20;

    alignas(RefHolder) unsigned char buf[sizeof(RefHolder)];

    // Construct with reference to target_a
    RefHolder* h1 = new (buf) RefHolder(target_a, 100);
    std::printf("h1: ref=%d, data=%d\n", h1->ref, h1->data);

    h1->~RefHolder();

    // Reconstruct with reference to target_b
    RefHolder* h2 = new (buf) RefHolder(target_b, 200);

    // Using h1 here is UB: the reference member changed,
    // but h1's provenance refers to the old object.
    // The compiler may assume h1->ref still refers to target_a.

    // CORRECT: use h2 or std::launder
    std::printf("h2: ref=%d, data=%d\n", h2->ref, h2->data);

    RefHolder* h3 = std::launder(reinterpret_cast<RefHolder*>(buf));
    std::printf("h3: ref=%d, data=%d\n", h3->ref, h3->data);

    h2->~RefHolder();
}

// Array placement new and provenance
void array_provenance() {
    // When using aligned storage for arrays, be careful with indexing.
    alignas(int) unsigned char buf[sizeof(int) * 4];

    // Construct 4 ints
    for (int i = 0; i < 4; ++i) {
        new (buf + i * sizeof(int)) int(i * 10);
    }

    // Access via pointer arithmetic on the storage
    // This requires std::launder because buf's provenance
    // is unsigned char[], not int[].
    int* arr = std::launder(reinterpret_cast<int*>(buf));

    // Note: pointer arithmetic on arr is only valid if the ints
    // form an array. Individual placement new does NOT create an array.
    // arr[0] is OK (points to the first int).
    // arr[1] is technically UB - no array object exists!

    // Safe approach: use std::array or construct a real array
    std::printf("arr[0] = %d\n", *std::launder(reinterpret_cast<int*>(buf)));
    std::printf("arr[1] = %d\n",
                *std::launder(reinterpret_cast<int*>(buf + sizeof(int))));
}

// C++23 alternative: std::start_lifetime_as_array
// auto* arr = std::start_lifetime_as_array<int>(buf, 4);
// This implicitly creates the array object, making pointer arithmetic legal.

int main() {
    reference_member_provenance();
    array_provenance();
}
```

Reference members have the same provenance issue as `const` members - the compiler assumes the reference target does not change. Additionally, individual placement `new` calls do not create an array object, so pointer arithmetic between them is UB. `std::launder` fixes provenance for individual elements; `std::start_lifetime_as_array` (C++23) creates a proper array.

---

### Q3: Explain when std::launder is NOT needed and common misuses

`std::launder` is one of those tools that people sometimes reach for too eagerly once they learn it exists. It is quite narrow in what it actually does, and misusing it does not help and can give a false sense of safety. Here is a clear breakdown of when you need it and when you do not:

```cpp
#include <cstdio>
#include <memory>
#include <new>
#include <vector>

struct Simple {
    int x;
    double y;

    Simple(int a, double b) : x(a), y(b) {}
};

void launder_not_needed() {
    // Case 1: Using the pointer returned by placement new - launder NOT needed
    alignas(Simple) unsigned char buf[sizeof(Simple)];
    Simple* s = new (buf) Simple(1, 2.0);
    // s is already the correct pointer with proper provenance.
    std::printf("x=%d, y=%.1f\n", s->x, s->y);
    s->~Simple();

    // Case 2: No const or reference members, reusing same pointer
    Simple* s2 = new (buf) Simple(3, 4.0);
    // Accessing through s2 is fine - it's the returned pointer.
    std::printf("x=%d, y=%.1f\n", s2->x, s2->y);
    s2->~Simple();

    // Case 3: Regular new/delete - launder never needed
    Simple* heap = new Simple(5, 6.0);
    std::printf("x=%d, y=%.1f\n", heap->x, heap->y);
    delete heap;

    // Case 4: Containers manage provenance internally
    std::vector<Simple> vec;
    vec.emplace_back(7, 8.0);
    // vector handles placement new and provenance correctly.
}

void launder_IS_needed() {
    // Case 1: Replacing an object with const members
    struct ConstMember {
        const int id;
        int value;
    };

    alignas(ConstMember) unsigned char buf[sizeof(ConstMember)];
    auto* c1 = new (buf) ConstMember{1, 100};
    c1->~ConstMember();
    new (buf) ConstMember{2, 200};

    // Must launder to access the new object through the buffer
    auto* c2 = std::launder(reinterpret_cast<ConstMember*>(buf));
    std::printf("New id=%d\n", c2->id);
    c2->~ConstMember();

    // Case 2: Type-erased storage in custom containers
    // When you store objects in aligned_storage/byte arrays and
    // retrieve them via reinterpret_cast, std::launder is needed.
}

// Common MISUSES of std::launder:

// WRONG: using launder to "fix" strict aliasing violations
// float f = 3.14f;
// int* p = std::launder(reinterpret_cast<int*>(&f));
// int bits = *p;  // STILL UB - launder does not override aliasing rules!

// WRONG: using launder on a pointer to memory where no object exists
// unsigned char buf[sizeof(int)];
// int* p = std::launder(reinterpret_cast<int*>(buf));
// int val = *p;  // UB - no int object has been created in buf!

// launder ONLY works when a valid object of the target type
// already exists at the address. It updates provenance, not type.

int main() {
    launder_not_needed();
    launder_IS_needed();
}
```

The reason this trips people up is that `std::launder` sounds like a magic "fix this pointer" tool, but it is actually very specific: it only works when (1) an object has been replaced via placement new, AND (2) you access it through a pointer that predates the replacement, AND (3) the type has `const`/reference members or you are accessing through a reinterpreted buffer pointer. It is NOT a "cast fix" - it does not override strict aliasing, and it requires a valid object to already exist at the address.

---

## Notes

- `std::launder` was introduced in C++17 largely to formalize what `std::optional` and `std::variant` implementations need to do internally.
- The pointer provenance model is being formalized in both C (PNVI-ae-UD model) and C++ (P2318).
- `std::launder` is a no-op at runtime - it compiles to nothing. Its effect is purely on the optimizer's alias analysis.
- In C++23, `std::start_lifetime_as<T>` subsumes many use cases of `std::launder` by implicitly creating the object.
- When implementing type-erased storage (like `std::any`), `std::launder` is essential for correctness when retrieving stored objects.
- Compilers are increasingly strict about provenance in their optimizers; code that "worked" without `std::launder` may break with newer compiler versions.
