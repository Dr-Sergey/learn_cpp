# Understand Itanium ABI Layout Rules for Vtables, RTTI, and Exceptions

**Category:** ABI and Binary Compatibility  
**Standard:** C++17, C++20 (ABI defined by Itanium C++ ABI specification)  
**Reference:** https://itanium-cxx-abi.github.io/cxx-abi/abi.html  

---

## Topic Overview

The Itanium C++ ABI is the de facto standard ABI used by GCC, Clang, and most non-MSVC compilers on Linux, macOS, and many other platforms. It defines the binary layout of vtables, RTTI structures, exception handling tables, and name mangling - ensuring that object files compiled by different conforming compilers can link together. Understanding this ABI is critical when building shared libraries, debugging corrupted vtables, or performing binary analysis.

Here is the part that surprises people: a vtable in the Itanium ABI is not simply an array of function pointers. Each vtable contains an **offset-to-top** value (used for `dynamic_cast` and virtual base adjustments), a **pointer to the typeinfo object**, and then the virtual function pointers in declaration order. When a class has multiple bases, secondary vtables are emitted for each base subobject, forming a **vtable group**. The primary base class shares the primary vtable, while secondary bases get their own vtable entries with thunks to adjust the `this` pointer.

If the table below feels like a lot, just remember: the vtable slots at negative indices are metadata, and the non-negative indices are the actual function pointers.

| Vtable Component         | Offset (slots) | Purpose                                      |
| --- | --- | --- |
| offset-to-top            | -2              | Distance from subobject to complete object    |
| typeinfo pointer         | -1              | Points to `std::type_info`-derived struct     |
| virtual function ptr [0] | 0               | First virtual function in declaration order   |
| virtual function ptr [1] | 1               | Second virtual function                       |
| ...                      | ...             | Continues for all virtual functions            |

RTTI structures follow a specific hierarchy: `__class_type_info` for classes without bases, `__si_class_type_info` for single non-virtual public inheritance, and `__vmi_class_type_info` for everything else (multiple or virtual inheritance). Each includes the mangled name and base class descriptors with offset/flag fields. The reason this matters in practice is that `dynamic_cast` and `typeid` both walk these structures at runtime, so their correctness depends on the whole chain being consistent.

Exception handling uses a two-phase model: phase 1 searches for a handler (personality routine + LSDA), phase 2 performs the actual unwinding. The Language-Specific Data Area (LSDA) encodes call-site ranges, action tables linking to type filters, and cleanup actions. The `.gcc_except_table` section holds this data, while `.eh_frame` holds the CFI (Call Frame Information) for stack unwinding.

---

## Self-Assessment

### Q1: Given a diamond inheritance hierarchy, predict the vtable group layout including offset-to-top and thunks

Diamond inheritance is one of the harder layout scenarios to reason about, because virtual bases mean the compiler has to place the shared `Base` subobject in exactly one location and give every path to it a way to find it. The `offset-to-top` field is the key - each subobject's vtable records how far back you have to go to reach the start of the complete object, so `dynamic_cast` can reconstruct the full picture. Let's look at how this plays out in code.

```cpp
#include <cstdio>
#include <cstdint>

struct Base {
    virtual void identify() { std::printf("Base\n"); }
    virtual ~Base() = default;
    int base_data = 0xBA;
};

struct Left : virtual Base {
    virtual void left_func() { std::printf("Left\n"); }
    int left_data = 0x1E;
};

struct Right : virtual Base {
    virtual void right_func() { std::printf("Right\n"); }
    int right_data = 0x21;
};

struct Diamond : Left, Right {
    void identify() override { std::printf("Diamond\n"); }
    void left_func() override { std::printf("Diamond::left\n"); }
    void right_func() override { std::printf("Diamond::right\n"); }
    int diamond_data = 0xDD;
};

// Dump vtable pointers and inspect layout at runtime
void inspect_layout() {
    Diamond d;

    // The vptr is typically the first member of the object
    auto obj_addr = reinterpret_cast<std::uintptr_t>(&d);

    // Primary vtable (Left subobject) - first pointer in Diamond
    auto primary_vptr = *reinterpret_cast<std::uintptr_t*>(obj_addr);

    // Read offset-to-top: located at vptr[-2] (in pointer-sized units)
    auto vtable_start = reinterpret_cast<std::intptr_t*>(primary_vptr);
    std::printf("Primary vtable offset-to-top: %lld\n",
                static_cast<long long>(vtable_start[-2]));
    std::printf("Primary vtable typeinfo ptr:  %p\n",
                reinterpret_cast<void*>(vtable_start[-1]));

    // Call through base pointer to verify thunk dispatch
    Base* bp = &d;
    bp->identify();  // Should print "Diamond" - possibly through thunk

    Left* lp = &d;
    lp->left_func(); // "Diamond::left"

    Right* rp = &d;
    rp->right_func(); // "Diamond::right" - uses secondary vtable + thunk
}

int main() {
    inspect_layout();
    return 0;
}
```

Notice that when you call through a `Right*`, the compiler uses the secondary vtable and a thunk to adjust `this` before jumping to the real `Diamond::right_func` implementation. That adjustment is what the `offset-to-top` makes possible.

**Expected Layout (Itanium ABI, 64-bit):**

```cpp
Diamond object layout:
  [Left  vptr]         -> primary vtable (offset-to-top = 0)
  [left_data]
  [Right vptr]         -> secondary vtable (offset-to-top = -sizeof(Left subobj))
  [right_data]
  [diamond_data]
  [padding]
  [Base  vptr]         -> virtual base vtable (offset-to-top = -offset to Diamond start)
  [base_data]
```

### Q2: Examine the RTTI structure generated for a class with virtual multiple inheritance and verify typeinfo chaining

RTTI is what powers `dynamic_cast` and `typeid` at runtime. For a class like `Duck` that inherits from multiple bases, the ABI generates a `__vmi_class_type_info` structure that lists every base along with its offset and flags. When you attempt a cross-cast (say, going from an `Animal*` to a `Swimmable*`), the runtime walks this structure to find the right offset. The key insight is that `typeinfo` objects have identity - two references to `typeid(Duck)` must compare equal, which requires both to point to the same address. That guarantee breaks if you use `-fvisibility=hidden` carelessly across shared library boundaries.

<!-- compile: needs POSIX header `cxxabi.h` -->
```cpp
#include <typeinfo>
#include <cstdio>
#include <cxxabi.h>  // GCC/Clang extension

struct Animal {
    virtual ~Animal() = default;
};

struct Flyable {
    virtual ~Flyable() = default;
    virtual void fly() = 0;
};

struct Swimmable {
    virtual ~Swimmable() = default;
    virtual void swim() = 0;
};

struct Duck : Animal, Flyable, Swimmable {
    void fly() override {}
    void swim() override {}
};

void inspect_rtti() {
    Duck d;
    const std::type_info& ti = typeid(d);

    // Demangle the name
    int status = 0;
    char* demangled = abi::__cxa_demangle(ti.name(), nullptr, nullptr, &status);
    std::printf("Type: %s (mangled: %s)\n", demangled, ti.name());
    std::free(demangled);

    // dynamic_cast exercises the RTTI path
    Animal* ap = &d;

    // Cross-cast: Animal* -> Swimmable* requires RTTI traversal
    Swimmable* sp = dynamic_cast<Swimmable*>(ap);
    std::printf("Cross-cast Animal* -> Swimmable*: %s\n",
                sp ? "success" : "failure");

    // Verify typeinfo comparison across shared library boundaries
    // (same type must have same typeinfo address with Itanium ABI
    //  when using -rdynamic or default visibility)
    const std::type_info& ti2 = typeid(Duck);
    std::printf("typeinfo identity: %s\n",
                (&ti == &ti2) ? "same address (correct)" : "different (visibility issue)");
}

// RTTI structure for Duck (conceptually):
// __vmi_class_type_info {
//     name: "4Duck"
//     __flags: __non_diamond_repeat_mask = 0
//     __base_count: 3
//     __base_info[0]: { Animal,  offset=0,  __public_mask }
//     __base_info[1]: { Flyable, offset=8,  __public_mask }
//     __base_info[2]: { Swimmable, offset=16, __public_mask }
// }

int main() {
    inspect_rtti();
    return 0;
}
```

### Q3: Decode the LSDA (Language-Specific Data Area) for a function with multiple catch handlers and understand exception dispatch

The LSDA is the data structure the runtime reads to figure out which catch handler - if any - matches a thrown exception. The reason this trips people up is that the type matching does not happen in simple top-to-bottom order in the source; it uses a pre-built action table that encodes the type filters in reverse order from how the type table is indexed. Phase 1 reads the LSDA to decide whether this stack frame can handle the exception; phase 2 actually runs the landing pad code. The comment block inside the code shows the conceptual layout the compiler generates for this exact function.

```cpp
#include <cstdio>
#include <stdexcept>
#include <typeinfo>

// Compile with: g++ -S -fverbose-asm -o - example.cpp | grep -A50 ".LLSDA"
// to see the actual LSDA tables generated

struct NetworkError : std::runtime_error {
    using std::runtime_error::runtime_error;
    int error_code;
};

struct TimeoutError : NetworkError {
    using NetworkError::NetworkError;
};

// This function has a complex LSDA due to multiple catch types
void process_request(int type) {
    try {
        if (type == 1)
            throw std::bad_alloc();
        if (type == 2)
            throw TimeoutError("timeout");
        if (type == 3)
            throw NetworkError("connection refused");
        if (type == 4)
            throw std::runtime_error("generic");
    }
    catch (const TimeoutError& e) {
        // LSDA action: type filter for TimeoutError typeinfo
        std::printf("Timeout: %s\n", e.what());
    }
    catch (const NetworkError& e) {
        // LSDA action: type filter for NetworkError typeinfo
        std::printf("Network: %s\n", e.what());
    }
    catch (const std::exception& e) {
        // LSDA action: type filter for std::exception typeinfo
        std::printf("Generic: %s\n", e.what());
    }
}

// LSDA structure (conceptual for above function):
//
// Header:
//   @LPStart encoding, @TType encoding, @TType base offset
//   Call-site encoding
//
// Call Site Table:
//   | Region Start | Region Length | Landing Pad | Action |
//   | throw site 1 | ...          | catch block | act 1  |
//   | throw site 2 | ...          | catch block | act 1  |
//   ...
//
// Action Table:
//   Action 1: filter=3 (TimeoutError), next -> Action 2
//   Action 2: filter=2 (NetworkError), next -> Action 3
//   Action 3: filter=1 (std::exception), next -> 0 (end)
//
// Type Table (reverse order):
//   [1] -> typeinfo for std::exception
//   [2] -> typeinfo for NetworkError
//   [3] -> typeinfo for TimeoutError

int main() {
    for (int i = 1; i <= 4; ++i) {
        std::printf("Request type %d: ", i);
        process_request(i);
    }
    return 0;
}
```

Watch what happens: even though `TimeoutError` derives from `NetworkError`, the runtime correctly dispatches to the most-derived handler first, because the action table checks `TimeoutError` before `NetworkError`. If you ever see a base-class handler catching a derived exception unexpectedly, the issue is usually in the catch ordering in source, not in the LSDA itself.

**Output:**

```text
Request type 1: Generic: std::bad_alloc
Request type 2: Timeout: timeout
Request type 3: Network: connection refused
Request type 4: Generic: generic
```

---

## Notes

- The **offset-to-top** field is critical for `dynamic_cast` - it tells the runtime where the complete object starts relative to the current subobject.
- Itanium ABI guarantees **typeinfo uniqueness by address** for types with default visibility - this breaks if `-fvisibility=hidden` is used across shared library boundaries.
- Virtual thunks adjust the `this` pointer using the **vcall offset** stored in the vtable, not a fixed offset - this handles virtual inheritance correctly.
- The LSDA type filter table is indexed in **reverse order** - filter index 1 corresponds to the last entry in the type table.
- Exception handling has two phases: phase 1 (search) calls `__gxx_personality_v0` with `_UA_SEARCH_PHASE`, phase 2 (cleanup) calls it with `_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME`.
- When debugging vtable corruption, compare the vtable pointer against `nm -C library.so | grep "vtable for ClassName"` to find the expected address.
- Adding a virtual function to a base class **shifts all subsequent vtable entries**, breaking ABI for all derived classes in other translation units.
