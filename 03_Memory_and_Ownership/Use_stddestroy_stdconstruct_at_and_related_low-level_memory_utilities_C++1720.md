# Use std::destroy, std::construct_at, and Related Low-Level Memory Utilities (C++17/20)

**Category:** Memory and Ownership  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/memory>  

---

## Topic Overview

### The Low-Level Memory Utilities

These are the building blocks for writing containers, allocators, and memory pools. Most application code never needs them directly, but if you are ever managing raw storage manually, these are the right tools.

| Function | Since | Purpose |
| --- | --- | --- |
| `std::construct_at(p, args...)` | C++20 | Construct object in raw memory |
| `std::destroy_at(p)` | C++17 | Destroy single object |
| `std::destroy(first, last)` | C++17 | Destroy range of objects |
| `std::destroy_n(first, n)` | C++17 | Destroy `n` objects from `first` |
| `std::uninitialized_copy` | C++11 | Copy-construct into raw memory |
| `std::uninitialized_move` | C++17 | Move-construct into raw memory |
| `std::uninitialized_fill` | C++11 | Fill raw memory with copies |
| `std::uninitialized_default_construct` | C++17 | Default-construct in raw memory |
| `std::uninitialized_value_construct` | C++17 | Value-construct (zero-init) in raw memory |

### Why Not Just Use `memcpy`

The reason `memcpy` is not enough for non-trivial types is that it copies raw bytes without invoking constructors or destructors. If an object owns heap memory (like `std::string`), you end up with two objects that think they own the same heap block, and the second destructor is a double-free.

```cpp
memcpy:               Copies raw bytes. OK for trivial types ONLY.
uninitialized_copy:   Calls copy constructors. Works for ALL types.
                      Exception-safe (destroys already-constructed on throw).
```

---

## Self-Assessment

### Q1: Use `std::construct_at` to construct an object in uninitialized storage without placement new

`std::construct_at` (C++20) does exactly what placement `new` does, but it works in `constexpr` contexts and pairs naturally with `std::destroy_at`. In generic container code, this cleaner pairing is the main advantage over raw placement `new`.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <new>

struct Widget {
    std::string name;
    int value;

    Widget(std::string n, int v) : name(std::move(n)), value(v) {
        std::cout << "  Constructed: " << name << "=" << value << "\n";
    }
    ~Widget() {
        std::cout << "  Destroyed: " << name << "\n";
    }
};

int main() {
    std::cout << "=== std::construct_at vs placement new ===\n\n";

    // Raw storage — no constructors called
    alignas(Widget) unsigned char buf1[sizeof(Widget)];
    alignas(Widget) unsigned char buf2[sizeof(Widget)];

    // OLD WAY: placement new
    std::cout << "Placement new:\n";
    Widget* w1 = new (buf1) Widget("PlacementNew", 1);

    // NEW WAY: std::construct_at (C++20)
    std::cout << "construct_at:\n";
    Widget* w2 = std::construct_at(
        reinterpret_cast<Widget*>(buf2), "ConstructAt", 2);

    // Both work identically — but construct_at is:
    // 1. constexpr-compatible (works in consteval/constexpr contexts)
    // 2. Cleaner syntax in generic code
    // 3. Pairs naturally with destroy_at

    std::cout << "\nUsing objects:\n";
    std::cout << "  w1: " << w1->name << "=" << w1->value << "\n";
    std::cout << "  w2: " << w2->name << "=" << w2->value << "\n";

    // Clean up
    std::cout << "\nDestruction:\n";
    std::destroy_at(w1);
    std::destroy_at(w2);

    // construct_at in generic code:
    std::cout << "\n=== Generic factory ===\n";
    alignas(int) unsigned char ibuf[sizeof(int)];
    int* ip = std::construct_at(reinterpret_cast<int*>(ibuf), 42);
    std::cout << "int: " << *ip << "\n";
    std::destroy_at(ip);

    return 0;
}
// Expected output:
// Placement new:
//   Constructed: PlacementNew=1
// construct_at:
//   Constructed: ConstructAt=2
//
// Using objects:
//   w1: PlacementNew=1
//   w2: ConstructAt=2
//
// Destruction:
//   Destroyed: PlacementNew
//   Destroyed: ConstructAt
//
// === Generic factory ===
// int: 42
```

### Q2: Use `std::destroy_at` and `std::destroy_n` for RAII cleanup of placement-new'd objects

Whenever you construct objects manually into raw storage, you are responsible for explicitly destroying them. `std::destroy_at` handles a single object, and `std::destroy_n` handles a contiguous range. The `SensorArray` RAII wrapper at the bottom shows how to bake this into a destructor so cleanup is automatic - the same pattern you will find in the internals of `std::vector`.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <new>
#include <utility>

struct Sensor {
    std::string id;
    double reading;

    Sensor(std::string i, double r) : id(std::move(i)), reading(r) {
        std::cout << "  [+] Sensor " << id << "\n";
    }
    ~Sensor() {
        std::cout << "  [-] Sensor " << id << "\n";
    }
};

int main() {
    constexpr size_t N = 4;

    std::cout << "=== destroy_at (single) and destroy_n (range) ===\n\n";

    // Allocate raw storage for N sensors
    alignas(Sensor) unsigned char storage[N * sizeof(Sensor)];
    Sensor* sensors = reinterpret_cast<Sensor*>(storage);

    // Construct N sensors manually
    std::cout << "Constructing:\n";
    std::construct_at(&sensors[0], "TEMP", 22.5);
    std::construct_at(&sensors[1], "PRES", 1013.25);
    std::construct_at(&sensors[2], "HUM", 45.0);
    std::construct_at(&sensors[3], "WIND", 12.3);

    // Use them
    std::cout << "\nReadings:\n";
    for (size_t i = 0; i < N; ++i) {
        std::cout << "  " << sensors[i].id << ": " << sensors[i].reading << "\n";
    }

    // destroy_at: destroy ONE object
    std::cout << "\ndestroy_at (single):\n";
    std::destroy_at(&sensors[0]);

    // destroy_n: destroy N objects starting from pointer
    std::cout << "\ndestroy_n (remaining 3):\n";
    std::destroy_n(&sensors[1], 3);

    // Also available:
    // std::destroy(first, last) — range version
    // These are exception-safe and work in constexpr too

    std::cout << "\n=== RAII wrapper using destroy_n ===\n";

    // A simple fixed-capacity container using these utilities
    struct SensorArray {
        alignas(Sensor) unsigned char data[4 * sizeof(Sensor)];
        size_t count = 0;

        Sensor* at(size_t i) { return reinterpret_cast<Sensor*>(data) + i; }

        template<typename... Args>
        void emplace_back(Args&&... args) {
            std::construct_at(at(count), std::forward<Args>(args)...);
            ++count;
        }

        ~SensorArray() {
            std::destroy_n(at(0), count);  // Clean up all live objects
            std::cout << "  SensorArray destroyed (" << count << " sensors)\n";
        }
    };

    {
        SensorArray arr;
        arr.emplace_back("GPS", 51.5);
        arr.emplace_back("ALT", 150.0);
    }  // Destructor calls destroy_n automatically

    return 0;
}
```

### Q3: Explain why `std::uninitialized_copy` is preferable to a raw `memcpy` for non-trivial types

The reason `memcpy` is dangerous for non-trivial types comes down to object lifetimes. `memcpy` produces a byte-for-byte duplicate of the source object's state, but the copy's constructor was never called. For something like `std::string`, that means two objects with internal pointers that both believe they own the same heap buffer. Both destructors will then call `free` on the same address - a double-free crash. `std::uninitialized_copy` calls the copy constructor for each element, which is the only correct way to do this. It is also exception-safe: if one constructor throws, the elements already constructed are properly destroyed before the exception propagates.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <algorithm>
#include <cstring>

struct Entry {
    std::string key;
    std::string value;

    Entry(std::string k, std::string v) : key(std::move(k)), value(std::move(v)) {
        std::cout << "  Constructed: " << key << "=" << value << "\n";
    }
    Entry(const Entry& o) : key(o.key), value(o.value) {
        std::cout << "  Copy-constructed: " << key << "\n";
    }
    ~Entry() {
        std::cout << "  Destroyed: " << key << "\n";
    }
};

int main() {
    std::cout << "=== memcpy vs uninitialized_copy ===\n\n";

    // Source data
    Entry source[2] = {{"name", "Alice"}, {"age", "30"}};

    // ---- BAD: memcpy for non-trivial types = UB ----
    // alignas(Entry) unsigned char bad_buf[2 * sizeof(Entry)];
    // std::memcpy(bad_buf, source, 2 * sizeof(Entry));
    //
    // WHY UB:
    // 1. Copies raw bytes — bypasses copy constructor
    // 2. std::string has internal pointers/SSO state — raw byte copy
    //    creates TWO objects "owning" the same heap buffer
    // 3. When both are destroyed -> double-free -> crash/corruption
    // 4. No Entry objects were constructed — no valid object lifetime

    // ---- GOOD: std::uninitialized_copy ----
    std::cout << "std::uninitialized_copy:\n";
    alignas(Entry) unsigned char good_buf[2 * sizeof(Entry)];
    Entry* dest = reinterpret_cast<Entry*>(good_buf);

    // Properly copy-constructs each element in uninitialized memory
    std::uninitialized_copy(source, source + 2, dest);

    std::cout << "\nUsing copied entries:\n";
    for (int i = 0; i < 2; ++i) {
        std::cout << "  " << dest[i].key << "=" << dest[i].value << "\n";
    }

    // Clean up properly constructed objects
    std::cout << "\nDestroying copies:\n";
    std::destroy_n(dest, 2);

    // ---- Summary ----
    std::cout << "\n=== When is memcpy OK? ===\n";
    std::cout << "memcpy is ONLY safe for trivially copyable types:\n";
    std::cout << "  int, float, POD structs, std::array<int,N>, etc.\n";
    std::cout << "  Check: std::is_trivially_copyable_v<T>\n\n";
    std::cout << "For non-trivial types (std::string, std::vector, etc.):\n";
    std::cout << "  Use std::uninitialized_copy (into raw memory)\n";
    std::cout << "  Use std::copy (into already-constructed range)\n";
    std::cout << "  Use std::uninitialized_move (move semantics)\n\n";
    std::cout << "uninitialized_copy is also EXCEPTION-SAFE:\n";
    std::cout << "  If copy ctor throws on element 5 of 10,\n";
    std::cout << "  it destroys elements 0-4 automatically.\n";

    return 0;
}
```

---

## Notes

- `std::construct_at` (C++20) replaces placement `new` and works in `constexpr` contexts.
- `std::destroy_at/destroy/destroy_n` (C++17) are the canonical way to end object lifetime in raw storage.
- `std::uninitialized_copy/move/fill` are exception-safe - they roll back on throw.
- `memcpy` is only legal for trivially copyable types; use `std::uninitialized_copy` for the rest.
- All these utilities are building blocks for implementing containers, allocators, and memory pools.
