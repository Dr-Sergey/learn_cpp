# Understand what makes a type trivially copyable and why it matters for serialization

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/named_req/TriviallyCopyable>  

---

## Topic Overview

A **trivially copyable** type can be safely copied byte-by-byte with `memcpy` or `memmove`. This is critical for serialization, networking, and interop with C APIs. If you ever need to blast a struct into a byte buffer and back, this is the property you're relying on.

### Conditions for Trivially Copyable

A type `T` is trivially copyable if **all** of these hold:

| Condition | Meaning |
| --- | --- |
| Trivial copy constructor | Copy ctor is not user-provided (compiler-generated or `= default`) |
| Trivial move constructor | Move ctor is not user-provided |
| Trivial copy assignment | Copy assign is not user-provided |
| Trivial move assignment | Move assign is not user-provided |
| Trivial destructor | Destructor is not user-provided |
| At least one non-deleted copy/move | The type must have at least one eligible copy/move operation |

Here's a quick check of what passes and what doesn't:

```cpp
#include <type_traits>

struct Trivial {
    int x;
    double y;
    char data[10];
};
static_assert(std::is_trivially_copyable_v<Trivial>);  // Yes

struct NonTrivial {
    std::string name;  // std::string has non-trivial copy/move/dtor
};
static_assert(!std::is_trivially_copyable_v<NonTrivial>);  // No
```

### What Breaks Trivial Copyability

Notice that even an empty user-provided special member function is enough to break trivial copyability - the compiler can't assume the body is a no-op even when it literally is:

```cpp
struct UserCopy {
    int x;
    UserCopy(const UserCopy& other) : x(other.x) {}  // User-provided copy ctor
};
static_assert(!std::is_trivially_copyable_v<UserCopy>);  // No

struct VirtualFunc {
    virtual void foo() {}  // vtable pointer added -> not trivially copyable
};
static_assert(!std::is_trivially_copyable_v<VirtualFunc>);  // No

struct DefaultedOK {
    int x;
    DefaultedOK(const DefaultedOK&) = default;  // Defaulted = trivial
    DefaultedOK& operator=(const DefaultedOK&) = default;
    ~DefaultedOK() = default;
};
static_assert(std::is_trivially_copyable_v<DefaultedOK>);  // Yes
```

### Trivially Copyable vs Standard-Layout

These two properties are often confused. They answer different questions - one is about copy behavior, the other is about memory layout:

| Property | Trivially Copyable | Standard-Layout |
| --- | --- | --- |
| Focus | Copy/move operations | Memory layout predictability |
| Virtual functions | No | No |
| Non-trivial members | No | Yes |
| Multiple access specifiers | Yes | No |
| Private base + derived data | Yes | No |
| POD type | Yes | Yes (both required) |

A type can be trivially copyable but NOT standard-layout - here's a concrete example:

```cpp
struct TriviallyCopyableNotStandardLayout {
private:
    int x;     // private access specifier
public:
    int y;     // public access specifier - two specifiers with data members!
};
// Trivially copyable: yes (all operations are trivial)
// Standard-layout: No (data members in multiple access specifiers)
static_assert(std::is_trivially_copyable_v<TriviallyCopyableNotStandardLayout>);
static_assert(!std::is_standard_layout_v<TriviallyCopyableNotStandardLayout>);
```

---

## Self-Assessment

### Q1: List the conditions that make a type trivially copyable and check with `std::is_trivially_copyable_v`

Let's go through the cases one by one so you can see what each rule actually catches in practice:

```cpp
#include <iostream>
#include <type_traits>
#include <cstring>

// 1. All special members trivial (compiler-generated or = default)
struct AllTrivial {
    int a;
    double b;
    char c[4];
};

// 2. Defaulted members still count as trivial
struct ExplicitDefault {
    int x;
    ExplicitDefault() = default;
    ExplicitDefault(const ExplicitDefault&) = default;
    ExplicitDefault(ExplicitDefault&&) = default;
    ExplicitDefault& operator=(const ExplicitDefault&) = default;
    ExplicitDefault& operator=(ExplicitDefault&&) = default;
    ~ExplicitDefault() = default;
};

// 3. User-provided destructor breaks it
struct UserDtor {
    int x;
    ~UserDtor() { /* nothing */ }  // Still non-trivial!
};

// 4. Virtual functions break it (vtable pointer)
struct HasVirtual {
    int x;
    virtual ~HasVirtual() = default;
};

// 5. Member with non-trivial type breaks it
struct HasString {
    std::string s;  // std::string is not trivially copyable
};

// 6. Inheritance is OK if base is trivially copyable
struct Base { int x; };
struct Derived : Base { int y; };

int main() {
    std::cout << std::boolalpha;
    std::cout << "AllTrivial:       " << std::is_trivially_copyable_v<AllTrivial> << "\n";       // true
    std::cout << "ExplicitDefault:  " << std::is_trivially_copyable_v<ExplicitDefault> << "\n";  // true
    std::cout << "UserDtor:         " << std::is_trivially_copyable_v<UserDtor> << "\n";         // false
    std::cout << "HasVirtual:       " << std::is_trivially_copyable_v<HasVirtual> << "\n";       // false
    std::cout << "HasString:        " << std::is_trivially_copyable_v<HasString> << "\n";        // false
    std::cout << "Derived:          " << std::is_trivially_copyable_v<Derived> << "\n";          // true
    std::cout << "int:              " << std::is_trivially_copyable_v<int> << "\n";              // true
    std::cout << "int*:             " << std::is_trivially_copyable_v<int*> << "\n";             // true
}
```

**How this works:**

- The compiler generates trivial copy/move/destructor only if no user-provided versions exist and all members are trivially copyable themselves.
- `= default` counts as "not user-provided" - it's the compiler's default implementation.
- An empty user-provided destructor `~T() {}` is still non-trivial - the compiler can't prove it's a no-op in all cases.
- Virtual functions add a vtable pointer, which needs special handling during copy - making the type non-trivially copyable.

### Q2: Show that trivially copyable types can be safely serialized with `memcpy`

Once you've verified a type is trivially copyable, you can treat it as a blob of bytes. Here's the full pattern for binary serialization - including the important caveats about portability:

```cpp
#include <iostream>
#include <cstring>
#include <type_traits>
#include <fstream>

struct Packet {
    uint32_t id;
    float x, y, z;
    uint8_t flags;
    // Padding bytes may exist here!
};
static_assert(std::is_trivially_copyable_v<Packet>);

int main() {
    // 1. memcpy between instances
    Packet src{42, 1.0f, 2.0f, 3.0f, 0xFF};
    Packet dst;
    std::memcpy(&dst, &src, sizeof(Packet));
    std::cout << "Copied: id=" << dst.id
              << " pos=(" << dst.x << "," << dst.y << "," << dst.z << ")"
              << " flags=" << (int)dst.flags << "\n";

    // 2. Serialize to byte buffer
    unsigned char buffer[sizeof(Packet)];
    std::memcpy(buffer, &src, sizeof(Packet));

    // 3. Deserialize from byte buffer
    Packet restored;
    std::memcpy(&restored, buffer, sizeof(Packet));
    std::cout << "Restored: id=" << restored.id << "\n";

    // 4. Write to file (binary serialization)
    {
        std::ofstream out("packet.bin", std::ios::binary);
        out.write(reinterpret_cast<const char*>(&src), sizeof(Packet));
    }

    // 5. Read from file
    {
        Packet loaded;
        std::ifstream in("packet.bin", std::ios::binary);
        in.read(reinterpret_cast<char*>(&loaded), sizeof(Packet));
        std::cout << "Loaded from file: id=" << loaded.id << "\n";
    }

    // WARNING: This is NOT portable across:
    // - Different endianness (x86 vs ARM big-endian)
    // - Different compilers (different padding/alignment)
    // - Different struct versions (adding/removing fields)
    // For cross-platform serialization, use a proper format (protobuf, flatbuffers, etc.)
}
```

**How this works:**

- `memcpy` copies raw bytes - it works correctly only if the type's representation IS its value (trivially copyable).
- For `std::string`, `memcpy` would copy the internal pointer, not the string data - resulting in double-free.
- Binary serialization is fast but not portable - use for same-machine IPC, shared memory, or network protocols with defined endianness.

### Q3: Demonstrate a type that is trivially copyable but not standard-layout

This distinction matters because "safe to memcpy" and "has predictable C-compatible layout" are separate guarantees. You can have one without the other:

```cpp
#include <iostream>
#include <type_traits>
#include <cstring>

// Trivially copyable but NOT standard-layout
// Reason: data members in BOTH private and public access specifiers
struct MixedAccess {
private:
    int secret = 10;
public:
    int visible = 20;

    int get_secret() const { return secret; }
};

// Verify:
static_assert(std::is_trivially_copyable_v<MixedAccess>);   // Yes
static_assert(!std::is_standard_layout_v<MixedAccess>);      // No

// Another example: base class AND derived class both have data
struct Base {
    int base_data;
};
struct DerivedWithData : Base {
    int derived_data;
};
// Standard-layout requires data in only ONE class in the hierarchy
static_assert(std::is_trivially_copyable_v<DerivedWithData>);  // Yes
static_assert(!std::is_standard_layout_v<DerivedWithData>);    // No

// For comparison - a POD type (both trivially copyable AND standard-layout):
struct POD {
    int x;
    double y;
    char z;
};
static_assert(std::is_trivially_copyable_v<POD>);  // Yes
static_assert(std::is_standard_layout_v<POD>);     // Yes

int main() {
    std::cout << std::boolalpha;
    std::cout << "MixedAccess - trivially copyable: "
              << std::is_trivially_copyable_v<MixedAccess>
              << ", standard-layout: "
              << std::is_standard_layout_v<MixedAccess> << "\n";

    std::cout << "DerivedWithData - trivially copyable: "
              << std::is_trivially_copyable_v<DerivedWithData>
              << ", standard-layout: "
              << std::is_standard_layout_v<DerivedWithData> << "\n";

    std::cout << "POD - trivially copyable: "
              << std::is_trivially_copyable_v<POD>
              << ", standard-layout: "
              << std::is_standard_layout_v<POD> << "\n";

    // memcpy works on MixedAccess even though it's not standard-layout:
    MixedAccess a;
    MixedAccess b;
    std::memcpy(&b, &a, sizeof(MixedAccess));
    std::cout << "Copied: secret=" << b.get_secret()
              << " visible=" << b.visible << "\n";
}
```

**Key distinction:**

- **Trivially copyable** = safe to `memcpy` (about copy semantics)
- **Standard-layout** = predictable memory layout, compatible with C (about layout)
- **POD** = trivially copyable + standard-layout + trivial default constructor (deprecated concept in C++20)

---

## Notes

- Use `static_assert(std::is_trivially_copyable_v<T>)` in serialization code to prevent accidental breakage when someone adds a `std::string` member.
- `std::atomic<T>` requires `T` to be trivially copyable.
- Padding bytes in trivially copyable types may contain indeterminate values - comparing with `memcmp` can give false negatives.
- C++20 added `std::bit_cast<To>(from)` as a type-safe alternative to `memcpy` for trivially copyable types of the same size.
- Being trivially copyable is a prerequisite for placement in `std::aligned_storage` (deprecated in C++23) and for use with `std::memcpy`-based optimizations in containers.
