# Know the difference between standard-layout types and their serialization guarantees

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/named_req/StandardLayoutType>  

---

## Topic Overview

### What Is a Standard-Layout Type

A **standard-layout type** is one whose memory layout is predictable and laid out the way a plain C struct would be. That predictability is the whole point - it's what lets you do the low-level things C++ otherwise frowns on:

- Serialization (writing and reading raw bytes).
- Talking to C code and hardware.
- Using `offsetof()` safely.
- Passing data across DLL/ABI boundaries.

### Conditions For Standard-Layout

A class or struct earns "standard-layout" only if **every one** of these is true. Don't memorize them cold - the theme is "nothing the compiler would need to lay out in a surprising way":

1. **No virtual functions** and no virtual base classes.
2. **All non-static data members share the same access** (all `public`, or all `private`, etc.).
3. **No non-static data members of reference type.**
4. **All non-static members and base classes** are themselves standard-layout.
5. **No two base subobjects of the same type** (no non-virtual diamond).
6. **All non-static data members are declared in one class** of the hierarchy - either only the most-derived class, or only one base.
7. **No base class has the same type as the first non-static data member** - otherwise two subobjects of one type would need the same address.

### Standard-Layout vs. Trivially Copyable vs. POD

These four properties get confused constantly. They're related but distinct, and each has its own trait you can check:

| Property | Meaning | Trait |
| --- | --- | --- |
| **Trivially copyable** | Can be `memcpy`'d safely | `std::is_trivially_copyable_v` |
| **Standard-layout** | Predictable C-compatible layout | `std::is_standard_layout_v` |
| **Trivial** | Default ctor/dtor/copy/move are all trivial | `std::is_trivial_v` |
| **POD** (deprecated C++20) | Both trivial AND standard-layout | `std::is_pod_v` (deprecated) |

A handy way to keep them straight: *trivially copyable* is about how you may **copy** the bytes; *standard-layout* is about how the bytes are **arranged**. Serialization usually wants both.

### Why Standard-Layout Matters

The three concrete guarantees you get, and the reasons each one is useful:

1. **`offsetof()` is well-defined** - it reliably gives you the byte offset of any member.
2. **A pointer to the first member equals a pointer to the object** - so `reinterpret_cast<int*>(&obj)` points at the first member when it's an `int`.
3. **C interoperability** - the layout matches what a C compiler would have produced.

---

## Self-Assessment

### Q1: List the conditions and verify with `std::is_standard_layout_v`

The fastest way to internalize the rules is to see the line that *breaks* each one. Every `static_assert` below is a mini-lesson:

```cpp
#include <type_traits>
#include <iostream>
#include <string>
#include <cstddef>

// Standard-layout: YES
struct Point {
    double x, y, z;
    // All public, no virtual, no references, trivial members
};
static_assert(std::is_standard_layout_v<Point>);

// Standard-layout: YES (even with methods!)
struct Rect {
    double x, y, w, h;
    double area() const { return w * h; }  // Non-virtual functions are fine
};
static_assert(std::is_standard_layout_v<Rect>);

// Standard-layout: NO - mixed access control
struct MixedAccess {
    int pub_member;
private:
    int priv_member;  // Different access -> not standard-layout
};
static_assert(!std::is_standard_layout_v<MixedAccess>);

// Standard-layout: NO - virtual function
struct WithVirtual {
    int data;
    virtual void foo() {}
};
static_assert(!std::is_standard_layout_v<WithVirtual>);

// Standard-layout: NO - reference member
struct WithRef {
    int& ref;
};
static_assert(!std::is_standard_layout_v<WithRef>);

// Standard-layout: NO - a member whose own type is not standard-layout
struct HasVirtualMember {
    WithVirtual inner;  // WithVirtual is not standard-layout, so neither is this
};
static_assert(!std::is_standard_layout_v<HasVirtualMember>);

// Standard-layout: NO - data in both base and derived
struct Base { int x; };
struct Derived : Base { int y; };  // Data in BOTH levels
static_assert(!std::is_standard_layout_v<Derived>);

// Standard-layout: YES - data only in derived
struct EmptyBase {};
struct DerivedOk : EmptyBase { int x, y; };  // Data only in DerivedOk
static_assert(std::is_standard_layout_v<DerivedOk>);

int main() {
    std::cout << std::boolalpha;
    std::cout << "Point:       " << std::is_standard_layout_v<Point> << "\n";       // true
    std::cout << "Rect:        " << std::is_standard_layout_v<Rect> << "\n";        // true
    std::cout << "MixedAccess: " << std::is_standard_layout_v<MixedAccess> << "\n"; // false
    std::cout << "WithVirtual: " << std::is_standard_layout_v<WithVirtual> << "\n"; // false
    std::cout << "WithRef:     " << std::is_standard_layout_v<WithRef> << "\n";     // false
    std::cout << "HasVirtualMember: " << std::is_standard_layout_v<HasVirtualMember> << "\n"; // false
    std::cout << "Derived:     " << std::is_standard_layout_v<Derived> << "\n";     // false
    std::cout << "DerivedOk:   " << std::is_standard_layout_v<DerivedOk> << "\n";   // true
    return 0;
}
```

Notice `Rect` is still standard-layout even though it has a method - it's *virtual* functions that break the property, not member functions in general.

### Q2: Explain why standard-layout guarantees `offsetof()` is well-defined

`offsetof(Type, member)` gives you the byte offset of `member` inside `Type`. It's only meaningful for standard-layout types, and the three reasons map directly onto the conditions above:

1. The compiler can't sneak hidden members (like a vtable pointer) in front of your data.
2. Members sit in declaration order, with no reordering across access sections.
3. The first member lives at offset 0 - it's pointer-interconvertible with the object.

```cpp
#include <cstddef>
#include <cstdint>
#include <iostream>
#include <type_traits>

struct Packet {  // Standard-layout
    uint32_t header;
    uint16_t length;
    uint8_t  flags;
    uint8_t  checksum;
};
static_assert(std::is_standard_layout_v<Packet>);

int main() {
    std::cout << "offsetof(header):   " << offsetof(Packet, header)   << "\n"; // 0
    std::cout << "offsetof(length):   " << offsetof(Packet, length)   << "\n"; // 4
    std::cout << "offsetof(flags):    " << offsetof(Packet, flags)    << "\n"; // 6
    std::cout << "offsetof(checksum): " << offsetof(Packet, checksum) << "\n"; // 7
    std::cout << "sizeof(Packet):     " << sizeof(Packet)             << "\n"; // 8

    // Pointer to first member == pointer to object:
    Packet pkt{0x12345678, 100, 0xFF, 0xAB};
    uint32_t* first = reinterpret_cast<uint32_t*>(&pkt);
    std::cout << "First member via cast: 0x" << std::hex << *first << "\n"; // 0x12345678

    return 0;
}
```

For *non*-standard-layout types, `offsetof()` was undefined behavior until C++17 and is only *conditionally supported* since: a compiler may accept it, warn, or reject it. The layout is the implementation's business - it is free to put a vtable pointer before the first member or arrange base subobjects as it likes - so portable code cannot rely on those offsets.

### Q3: Show a struct that breaks standard-layout by adding a virtual function

Adding a single `virtual` function is the most dramatic way to break the property - and you can literally watch `sizeof` balloon as the vtable pointer appears:

```cpp
#include <type_traits>
#include <cstddef>
#include <cstdint>
#include <cstring>
#include <iostream>

// Standard-layout version:
struct SensorDataSL {
    float temperature;
    float humidity;
    uint32_t timestamp;
};
static_assert(std::is_standard_layout_v<SensorDataSL>);

// Non-standard-layout version (virtual function added):
struct SensorDataVirtual {
    float temperature;
    float humidity;
    uint32_t timestamp;

    virtual void process() {}  // This BREAKS standard-layout!
};
static_assert(!std::is_standard_layout_v<SensorDataVirtual>);

int main() {
    std::cout << "sizeof(SL):      " << sizeof(SensorDataSL) << "\n";      // 12
    std::cout << "sizeof(Virtual):  " << sizeof(SensorDataVirtual) << "\n"; // 24 (8 bytes vtable ptr + padding)

    // The virtual function causes:
    // 1. A vtable pointer (typically 8 bytes on 64-bit) is inserted BEFORE the first member
    // 2. sizeof increases by at least pointer size + potential padding
    // 3. offsetof() is no longer guaranteed to work (conditionally supported)
    // 4. You can't memcpy/serialize the object safely
    // 5. C interoperability is broken

    // Byte-wise serialization needs BOTH properties:
    //   trivially copyable -> memcpy is allowed at all
    //   standard-layout    -> the bytes are arranged the way the other side expects
    static_assert(std::is_trivially_copyable_v<SensorDataSL>);
    SensorDataSL data{23.5f, 65.0f, 1000};
    unsigned char buffer[sizeof(SensorDataSL)];
    std::memcpy(buffer, &data, sizeof(data));

    SensorDataSL restored;
    std::memcpy(&restored, buffer, sizeof(restored));
    std::cout << "Restored temp: " << restored.temperature << "\n";  // 23.5

    return 0;
}
```

**What breaking standard-layout costs you:**

- `sizeof` grows (the vtable pointer is now part of the object).
- Members shift, so their offsets are no longer predictable.
- `offsetof()` is no longer guaranteed (conditionally supported since C++17, undefined before).
- `memcpy` serialization is unsafe - you'd be copying an internal vtable pointer that means nothing on the other end.
- C interoperability breaks.
- You can no longer cast pointer-to-first-member back to pointer-to-object.

### Common Pitfall: Standard-Layout Does Not Mean Serializable

The trap that catches almost everyone: a struct that's "obviously simple" but contains a `std::string`. Checking `is_standard_layout` will not save you here. On libstdc++, libc++ and the MSVC library `std::string` *is* standard-layout (the standard does not promise either way), so the struct passes the check - and is still impossible to serialize as bytes, because the string owns heap memory and is not trivially copyable:

```cpp
#include <string>
#include <type_traits>

struct Config {
    int version;
    std::string name;     // owns a heap buffer: NOT trivially copyable
    double threshold;
};
static_assert(!std::is_trivially_copyable_v<Config>);  // memcpy would copy a pointer, not the text
// std::is_standard_layout_v<Config> is typically TRUE - and tells you nothing useful here

// Fix: use char arrays for C-compatible structs
struct ConfigC {
    int version;
    char name[64];        // Fixed-size char array: the bytes live in the struct
    double threshold;
};
static_assert(std::is_standard_layout_v<ConfigC>);
static_assert(std::is_trivially_copyable_v<ConfigC>);
```

If you need a C-compatible, serializable struct, swap the `std::string` for a fixed-size `char` array, and assert **both** traits. You lose convenience but regain a layout you can hand to C and bytes you are allowed to copy.

---

## Notes

- Standard-layout types have a C-compatible arrangement of members, so they can be handed to/from C code and hardware.
- Copying the bytes with `memcpy` additionally requires the type to be trivially copyable - check both traits before serializing.
- Virtual functions, reference members, and mixed access control each break standard-layout.
- Use `std::is_standard_layout_v<T>` to check at compile time.
- `offsetof()` is only guaranteed for standard-layout types; for others it is conditionally supported.
- `std::string` is standard-layout on the major implementations, yet never trivially copyable - layout alone says nothing about ownership.
- POD = trivial + standard-layout (the term was deprecated in C++20, but the idea is still a useful mental model).
