# Use `std::is_trivially_copyable` to Validate memcpy-Safe Types

**Category:** Type System and Deduction  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/types/is_trivially_copyable>  

---

## Topic Overview

### What Is Trivially Copyable

A **trivially copyable** type can be safely copied byte-by-byte using `memcpy`, `memmove`, or written to disk/network as raw bytes. The C++ standard guarantees that for trivially copyable types, the object representation (its bytes in memory) fully defines the object's value.

This matters because copying a `std::string` byte-by-byte would be disastrous - the string owns heap memory and the copy would corrupt both objects. The trait gives you a compile-time way to verify before you do anything that dangerous.

### Requirements for Trivially Copyable

A type `T` is trivially copyable if **ALL** of these hold:

| Requirement | Meaning |
| --- | --- |
| Trivial copy constructor | Compiler-generated, no custom logic |
| Trivial move constructor | Compiler-generated, no custom logic |
| Trivial copy assignment | Compiler-generated, no custom logic |
| Trivial move assignment | Compiler-generated, no custom logic |
| Trivial destructor | No custom cleanup needed |
| At least one non-deleted copy/move operation | Type must be copyable or movable |

"Trivial" means the compiler can implement the operation as a simple `memcpy` - no user-defined logic runs.

### What Breaks Trivial Copyability

The most common culprit is a user-defined destructor. Even an empty one counts:

```cpp
#include <type_traits>
#include <string>
#include <vector>

// TRIVIALLY COPYABLE:
struct Point { float x, y, z; };
static_assert(std::is_trivially_copyable_v<Point>);

struct WithArray { int data[10]; double extra; };
static_assert(std::is_trivially_copyable_v<WithArray>);

// NOT TRIVIALLY COPYABLE:
struct HasVirtual { virtual void f() {} };         // vtable pointer
struct HasString  { std::string name; };            // string has custom dtor
struct HasVector  { std::vector<int> v; };          // vector has custom dtor

static_assert(!std::is_trivially_copyable_v<HasVirtual>);
static_assert(!std::is_trivially_copyable_v<HasString>);
static_assert(!std::is_trivially_copyable_v<HasVector>);
```

### Safe `memcpy` Pattern

The standard pattern is to guard your `memcpy` call with a `static_assert` so that if someone later adds a non-trivial member to `T`, the build breaks at the right place rather than producing silent memory corruption:

```cpp
#include <cstring>

template<typename T>
void safe_memcpy(T* dst, const T* src, size_t count) {
    static_assert(std::is_trivially_copyable_v<T>,
                  "memcpy requires trivially copyable types!");
    std::memcpy(dst, src, count * sizeof(T));
}
```

### Related Traits

| Trait | What It Checks |
| --- | --- |
| `is_trivially_copyable` | Safe to memcpy |
| `is_trivial` | Trivially copyable + trivial default constructor |
| `is_standard_layout` | Predictable memory layout (for C interop) |
| `is_pod` (deprecated C++20) | Was `is_trivial && is_standard_layout` |

---

## Self-Assessment

### Q1: Assert that a serializable struct is `is_trivially_copyable` before using `memcpy` on it

Here is a realistic serialization pattern with the static_assert acting as a built-in safety net:

```cpp
#include <iostream>
#include <type_traits>
#include <cstring>
#include <cstdint>
#include <array>

// A network packet header - must be memcpy-safe for serialization
struct PacketHeader {
    uint32_t magic;
    uint16_t version;
    uint16_t type;
    uint32_t payload_size;
    uint32_t checksum;
};

// Compile-time validation
static_assert(std::is_trivially_copyable_v<PacketHeader>,
              "PacketHeader must be trivially copyable for binary serialization");

// A sensor reading for binary file storage
struct SensorReading {
    double timestamp;
    float  temperature;
    float  humidity;
    int32_t sensor_id;
};

static_assert(std::is_trivially_copyable_v<SensorReading>,
              "SensorReading must be trivially copyable for binary I/O");

// Generic binary serializer
template<typename T>
std::array<std::byte, sizeof(T)> serialize(const T& obj) {
    static_assert(std::is_trivially_copyable_v<T>,
                  "Only trivially copyable types can be serialized as raw bytes");
    std::array<std::byte, sizeof(T)> buffer;
    std::memcpy(buffer.data(), &obj, sizeof(T));
    return buffer;
}

template<typename T>
T deserialize(const std::array<std::byte, sizeof(T)>& buffer) {
    static_assert(std::is_trivially_copyable_v<T>,
                  "Only trivially copyable types can be deserialized from raw bytes");
    T obj;
    std::memcpy(&obj, buffer.data(), sizeof(T));
    return obj;
}

int main() {
    // Serialize a packet header
    PacketHeader header{0xDEADBEEF, 1, 42, 1024, 0xABCD};
    auto bytes = serialize(header);

    std::cout << "Serialized PacketHeader: " << sizeof(header) << " bytes\n";

    // Deserialize it back
    auto restored = deserialize<PacketHeader>(bytes);
    std::cout << "Restored: magic=0x" << std::hex << restored.magic
              << " version=" << std::dec << restored.version
              << " type=" << restored.type
              << " payload=" << restored.payload_size
              << " checksum=0x" << std::hex << restored.checksum << "\n";

    // Batch copy sensor readings via memcpy
    SensorReading readings[3] = {
        {1.0, 22.5f, 45.0f, 1},
        {2.0, 23.1f, 44.8f, 2},
        {3.0, 21.9f, 46.2f, 3}
    };

    SensorReading copy[3];
    static_assert(std::is_trivially_copyable_v<SensorReading>);
    std::memcpy(copy, readings, sizeof(readings));  // Safe!

    std::cout << std::dec;
    for (const auto& r : copy) {
        std::cout << "Sensor " << r.sensor_id
                  << ": temp=" << r.temperature
                  << " humidity=" << r.humidity << "\n";
    }

    // This would NOT compile:
    // struct Bad { std::string name; int id; };
    // serialize(Bad{"test", 1});  // static_assert fails

    return 0;
}
```

**Output:**

```text
Serialized PacketHeader: 16 bytes
Restored: magic=0xdeadbeef version=1 type=42 payload=1024 checksum=0xabcd
Sensor 1: temp=22.5 humidity=45
Sensor 2: temp=23.1 humidity=44.8
Sensor 3: temp=21.9 humidity=46.2
```

### Q2: Show how adding a non-trivial destructor removes trivial copyability

This is the result that surprises most people: an empty destructor still breaks trivial copyability. The fix is `= default`:

```cpp
#include <iostream>
#include <type_traits>
#include <cstring>

// Start with a trivially copyable type
struct ResourceA {
    int handle;
    double data;
};

static_assert(std::is_trivially_copyable_v<ResourceA>);         // trivially copyable
static_assert(std::is_trivially_destructible_v<ResourceA>);     // trivial dtor
static_assert(std::is_trivially_copy_constructible_v<ResourceA>);

// Now add a user-defined destructor:
struct ResourceB {
    int handle;
    double data;
    ~ResourceB() { /* cleanup, e.g., close(handle) */ }  // non-trivial!
};

static_assert(!std::is_trivially_copyable_v<ResourceB>);        // NOT trivially copyable
static_assert(!std::is_trivially_destructible_v<ResourceB>);    // non-trivial dtor
// Note: copy constructor may still be trivial, but the TYPE is not trivially copyable
// because trivial copyability requires ALL special members to be trivial

// Even an EMPTY destructor breaks it:
struct ResourceC {
    int handle;
    double data;
    ~ResourceC() {}  // empty but user-defined -> still non-trivial!
};

static_assert(!std::is_trivially_copyable_v<ResourceC>);

// The fix: use = default
struct ResourceD {
    int handle;
    double data;
    ~ResourceD() = default;  // explicitly defaulted -> trivial again
};

static_assert(std::is_trivially_copyable_v<ResourceD>);   // back to trivially copyable

// What about other non-trivial members?
struct HasCustomCopy {
    int x;
    HasCustomCopy(const HasCustomCopy& o) : x(o.x) {}  // user-defined copy ctor
    HasCustomCopy& operator=(const HasCustomCopy&) = default;
};

static_assert(!std::is_trivially_copyable_v<HasCustomCopy>);  // copy ctor is non-trivial

struct HasCustomAssign {
    int x;
    HasCustomAssign& operator=(const HasCustomAssign& o) {
        x = o.x;
        return *this;
    }
};

static_assert(!std::is_trivially_copyable_v<HasCustomAssign>);  // copy assign is non-trivial

int main() {
    std::cout << std::boolalpha;

    std::cout << "=== Trivially Copyable Status ===\n";
    std::cout << "ResourceA (plain):           " << std::is_trivially_copyable_v<ResourceA> << "\n";
    std::cout << "ResourceB (custom dtor):     " << std::is_trivially_copyable_v<ResourceB> << "\n";
    std::cout << "ResourceC (empty dtor):      " << std::is_trivially_copyable_v<ResourceC> << "\n";
    std::cout << "ResourceD (defaulted dtor):  " << std::is_trivially_copyable_v<ResourceD> << "\n";
    std::cout << "HasCustomCopy:               " << std::is_trivially_copyable_v<HasCustomCopy> << "\n";
    std::cout << "HasCustomAssign:             " << std::is_trivially_copyable_v<HasCustomAssign> << "\n";

    std::cout << "\n=== Breakdown of ResourceB ===\n";
    std::cout << "trivially_copy_constructible: " << std::is_trivially_copy_constructible_v<ResourceB> << "\n";
    std::cout << "trivially_move_constructible: " << std::is_trivially_move_constructible_v<ResourceB> << "\n";
    std::cout << "trivially_copy_assignable:    " << std::is_trivially_copy_assignable_v<ResourceB> << "\n";
    std::cout << "trivially_move_assignable:    " << std::is_trivially_move_assignable_v<ResourceB> << "\n";
    std::cout << "trivially_destructible:       " << std::is_trivially_destructible_v<ResourceB> << "\n";
    std::cout << "-> trivially_copyable:        " << std::is_trivially_copyable_v<ResourceB> << "\n";

    return 0;
}
```

**Output:**

```text
=== Trivially Copyable Status ===
ResourceA (plain):           true
ResourceB (custom dtor):     false
ResourceC (empty dtor):      false
ResourceD (defaulted dtor):  true
HasCustomCopy:               false
HasCustomAssign:             false

=== Breakdown of ResourceB ===
trivially_copy_constructible: true
trivially_move_constructible: true
trivially_copy_assignable:    true
trivially_move_assignable:    true
trivially_destructible:       false
-> trivially_copyable:        false
```

Even though `ResourceB`'s copy/move operations are all trivial, the non-trivial destructor alone is enough to make the entire type non-trivially-copyable. An empty `~T() {}` is still user-defined and thus non-trivial. Use `~T() = default;` to keep triviality.

### Q3: Explain the full set of requirements: trivial copy/move ctors, dtor, and assignment

A type `T` is *trivially copyable* if and only if all six requirements in the table below hold. Pay special attention to Case 2 and Case 3 in the code - they are the ones that surprise people most:

| # | Requirement | Details |
| --- | --- | --- |
| 1 | **Trivial destructor** | `~T()` is implicitly defined or `= default`, no virtual base classes |
| 2 | **Trivial copy constructor** | Not user-provided (implicit or `= default`), no virtual bases |
| 3 | **Trivial move constructor** | Not user-provided (implicit or `= default`), no virtual bases, OR deleted |
| 4 | **Trivial copy assignment** | Not user-provided (implicit or `= default`), no virtual bases |
| 5 | **Trivial move assignment** | Not user-provided (implicit or `= default`), no virtual bases, OR deleted |
| 6 | **At least one non-deleted** | At least one of copy ctor, move ctor, copy assign, or move assign is non-deleted |

```cpp
#include <iostream>
#include <type_traits>

// Comprehensive test: all trivially copyable requirements
template<typename T>
void diagnose_trivial_copyability() {
    std::cout << std::boolalpha;
    std::cout << "  trivially_destructible:       " << std::is_trivially_destructible_v<T> << "\n";
    std::cout << "  trivially_copy_constructible: " << std::is_trivially_copy_constructible_v<T> << "\n";
    std::cout << "  trivially_move_constructible: " << std::is_trivially_move_constructible_v<T> << "\n";
    std::cout << "  trivially_copy_assignable:    " << std::is_trivially_copy_assignable_v<T> << "\n";
    std::cout << "  trivially_move_assignable:    " << std::is_trivially_move_assignable_v<T> << "\n";
    std::cout << "  -> trivially_copyable:        " << std::is_trivially_copyable_v<T> << "\n";
}

// Case 1: All trivial (plain aggregate)
struct Case1 { int x; double y; };

// Case 2: Deleted move, but copy is trivial -> still trivially copyable!
struct Case2 {
    int x;
    Case2(const Case2&) = default;
    Case2(Case2&&) = delete;       // deleted move
    Case2& operator=(const Case2&) = default;
    Case2& operator=(Case2&&) = delete;
};

// Case 3: Virtual function -> vtable pointer breaks layout guarantees
struct Case3 { virtual void f() {} int x; };

// Case 4: Virtual inheritance
struct Base { int x; };
struct Case4 : virtual Base { int y; };

// Case 5: All operations deleted -> not trivially copyable (rule 6)
struct Case5 {
    Case5(const Case5&) = delete;
    Case5(Case5&&) = delete;
    Case5& operator=(const Case5&) = delete;
    Case5& operator=(Case5&&) = delete;
};

int main() {
    std::cout << "=== Case 1: Plain aggregate ===\n";
    diagnose_trivial_copyability<Case1>();

    std::cout << "\n=== Case 2: Deleted move, trivial copy ===\n";
    diagnose_trivial_copyability<Case2>();

    std::cout << "\n=== Case 3: Virtual function ===\n";
    diagnose_trivial_copyability<Case3>();

    std::cout << "\n=== Case 4: Virtual inheritance ===\n";
    diagnose_trivial_copyability<Case4>();

    std::cout << "\n=== Case 5: All ops deleted ===\n";
    diagnose_trivial_copyability<Case5>();

    // Built-in types
    std::cout << "\n=== Built-in types ===\n";
    std::cout << "int:    " << std::is_trivially_copyable_v<int> << "\n";
    std::cout << "double: " << std::is_trivially_copyable_v<double> << "\n";
    std::cout << "int*:   " << std::is_trivially_copyable_v<int*> << "\n";
    std::cout << "int[5]: " << std::is_trivially_copyable_v<int[5]> << "\n";

    return 0;
}
```

**Output:**

```text
=== Case 1: Plain aggregate ===
  trivially_destructible:       true
  trivially_copy_constructible: true
  trivially_move_constructible: true
  trivially_copy_assignable:    true
  trivially_move_assignable:    true
  -> trivially_copyable:        true

=== Case 2: Deleted move, trivial copy ===
  trivially_destructible:       true
  trivially_copy_constructible: true
  trivially_move_constructible: false
  trivially_copy_assignable:    true
  trivially_move_assignable:    false
  -> trivially_copyable:        true

=== Case 3: Virtual function ===
  trivially_destructible:       true
  trivially_copy_constructible: true
  trivially_move_constructible: true
  trivially_copy_assignable:    true
  trivially_move_assignable:    true
  -> trivially_copyable:        false

=== Case 4: Virtual inheritance ===
  trivially_destructible:       true
  trivially_copy_constructible: false
  trivially_move_constructible: false
  trivially_copy_assignable:    false
  trivially_move_assignable:    false
  -> trivially_copyable:        false

=== Case 5: All ops deleted ===
  trivially_destructible:       true
  trivially_copy_constructible: false
  trivially_move_constructible: false
  trivially_copy_assignable:    false
  trivially_move_assignable:    false
  -> trivially_copyable:        false
```

Two surprises worth flagging. **Case 2:** deleted move doesn't break trivial copyability - only the non-deleted operations need to be trivial. **Case 3:** even though all individual traits report `true`, virtual functions have hidden state (vtable pointer) that makes memcpy unsafe - `is_trivially_copyable` correctly reports `false`.

---

## Notes

- **Use `static_assert`** at the point of use (before `memcpy`, before binary I/O) - not just at type definition. This catches problems when types evolve.
- **`std::bit_cast<T>` (C++20)** also requires trivially copyable types. It's a type-safe alternative to `memcpy` for reinterpretation.
- Standard containers (`vector`, `string`, `map`, etc.) are NEVER trivially copyable - they manage heap resources.
- **Padding bytes** are part of the object representation but may have indeterminate values. Two trivially copyable objects that compare equal may have different padding bits - be careful when hashing raw bytes.
