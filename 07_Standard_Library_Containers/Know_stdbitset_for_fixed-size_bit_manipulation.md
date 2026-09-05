# Know std::bitset for fixed-size bit manipulation

**Category:** Standard Library Containers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/bitset>  

---

## Topic Overview

`std::bitset<N>` is a fixed-size sequence of `N` bits with a rich API for bitwise operations. Unlike `std::vector<bool>`, the size is a compile-time constant and the interface provides named operations (`set`, `reset`, `test`, `flip`) alongside bitwise operators. If you've ever worked with raw integer bitmasks and thought "there has to be a safer way to do this," `std::bitset` is usually the answer when the number of flags is known at compile time.

### Key API

One thing to note before you dive in: bit indexing is **LSB = 0**, meaning the rightmost bit is index 0. So `bitset<8>("10000000")` has its set bit at index 7, not index 0. This trips people up constantly.

| Operation | Description | Example |
| --- | --- | --- |
| `bitset<N>()` | Default: all bits 0 | `bitset<8> b;` -> `00000000` |
| `bitset<N>(val)` | From unsigned long long | `bitset<8> b(0b1010);` -> `00001010` |
| `bitset<N>(str)` | From string of '0'/'1' | `bitset<8>("11001100")` |
| `b.set(pos)` | Set bit at `pos` to 1 | `b.set(3);` |
| `b.set()` | Set all bits to 1 | `b.set();` |
| `b.reset(pos)` | Clear bit at `pos` | `b.reset(3);` |
| `b.reset()` | Clear all bits | `b.reset();` |
| `b.flip(pos)` | Toggle bit at `pos` | `b.flip(3);` |
| `b.flip()` | Toggle all bits | `b.flip();` |
| `b.test(pos)` | Returns `true` if bit at `pos` is 1 | `if (b.test(3)) ...` |
| `b[pos]` | Access bit (no bounds check) | `b[3] = 1;` |
| `b.count()` | Number of set bits (popcount) | `b.count()` -> `4` |
| `b.size()` | Returns `N` | `b.size()` -> `8` |
| `b.any()` | True if any bit is set | |
| `b.none()` | True if no bits are set | |
| `b.all()` | True if all bits are set | |
| `b.to_ulong()` | Convert to `unsigned long` | `b.to_ulong()` -> `204` |
| `b.to_ullong()` | Convert to `unsigned long long` | |
| `b.to_string()` | Convert to string of '0'/'1' | `b.to_string()` -> `"11001100"` |
| `b & c`, `b \| c`, `b ^ c` | Bitwise AND, OR, XOR | `auto r = a & b;` |
| `~b` | Bitwise NOT | `auto r = ~b;` |
| `b << n`, `b >> n` | Shift left/right | `auto r = b << 2;` |

### Core Example

Let's see the main operations together. Notice that `a` starts as `0b11001010`, which is decimal 202 - you can verify that with `to_ulong()` at the end.

```cpp
#include <iostream>
#include <bitset>
#include <string>

int main() {
    // Construction
    std::bitset<8> a(0b11001010);        // From integer
    std::bitset<8> b("01010101");        // From string
    std::bitset<8> c;                    // All zeros

    std::cout << "a = " << a << "\n";    // Output: a = 11001010
    std::cout << "b = " << b << "\n";    // Output: b = 01010101

    // Bitwise operations
    std::cout << "a & b = " << (a & b) << "\n";   // Output: a & b = 01000000
    std::cout << "a | b = " << (a | b) << "\n";   // Output: a | b = 11011111
    std::cout << "a ^ b = " << (a ^ b) << "\n";   // Output: a ^ b = 10011111
    std::cout << "~a    = " << ~a << "\n";         // Output: ~a    = 00110101

    // Query
    std::cout << "a.count() = " << a.count() << "\n";   // Output: a.count() = 4
    std::cout << "a.test(7) = " << a.test(7) << "\n";   // Output: a.test(7) = 1 (MSB)
    std::cout << "a.any()   = " << a.any() << "\n";     // Output: a.any()   = 1

    // Conversion
    std::cout << "a.to_ulong()  = " << a.to_ulong() << "\n";   // Output: 202
    std::cout << "a.to_string() = " << a.to_string() << "\n";   // Output: 11001010

    // Modification
    a.set(0);     // Set bit 0
    a.reset(7);   // Clear bit 7
    a.flip(1);    // Toggle bit 1
    std::cout << "Modified a = " << a << "\n";  // Output: Modified a = 01001001

    return 0;
}
```

The `operator<<` on a `bitset` prints bits from MSB (leftmost, highest index) to LSB (rightmost, index 0). This matches how you'd write a binary number on paper, which makes the output feel natural.

### Important Notes

- `N` must be a **compile-time constant** - `std::bitset` cannot be resized at runtime.
- Bit indexing is **LSB = 0** (rightmost bit). `bitset<8>("10000000").test(7)` is `true`.
- `to_ulong()` throws `std::overflow_error` if the value doesn't fit in `unsigned long`.
- `test(pos)` throws `std::out_of_range` if `pos >= N`. `operator[]` does **not** check bounds.
- `std::bitset` is not a container - it has no iterators and cannot be used with `<algorithm>`.

---

## Self-Assessment

### Q1: Use std::bitset to implement a simple permission flags system

This is the classic motivating example for `std::bitset`. Each permission maps to a bit position, and the bitset gives you clean named operations instead of raw `|=`, `&=`, and shift arithmetic.

```cpp
#include <iostream>
#include <bitset>
#include <string>

// Permission flags
enum Permission : std::size_t {
    Read    = 0,  // Bit 0
    Write   = 1,  // Bit 1
    Execute = 2,  // Bit 2
    Delete  = 3,  // Bit 3
    Admin   = 4,  // Bit 4
    NUM_PERMISSIONS = 5
};

using Permissions = std::bitset<NUM_PERMISSIONS>;

void print_permissions(const std::string& name, const Permissions& p) {
    std::cout << name << ": " << p << " [";
    if (p.test(Read))    std::cout << " Read";
    if (p.test(Write))   std::cout << " Write";
    if (p.test(Execute)) std::cout << " Execute";
    if (p.test(Delete))  std::cout << " Delete";
    if (p.test(Admin))   std::cout << " Admin";
    std::cout << " ]\n";
}

int main() {
    // Create user with Read + Write
    Permissions user;
    user.set(Read);
    user.set(Write);
    print_permissions("User", user);
    // Output: User: 00011 [ Read Write ]

    // Create admin with all permissions
    Permissions admin;
    admin.set();  // All bits on
    print_permissions("Admin", admin);
    // Output: Admin: 11111 [ Read Write Execute Delete Admin ]

    // Grant Execute to user
    user.set(Execute);
    print_permissions("User+Exec", user);
    // Output: User+Exec: 00111 [ Read Write Execute ]

    // Revoke Write
    user.reset(Write);
    print_permissions("User-Write", user);
    // Output: User-Write: 00101 [ Read Execute ]

    // Check if user can delete
    std::cout << "Can delete? " << std::boolalpha << user.test(Delete) << "\n";
    // Output: Can delete? false

    // Intersection: what permissions do user and admin share?
    Permissions shared = user & admin;
    print_permissions("Shared", shared);
    // Output: Shared: 00101 [ Read Execute ]

    // Union: combine permissions
    Permissions combined = user | Permissions().set(Delete);
    print_permissions("Combined", combined);
    // Output: Combined: 01101 [ Read Execute Delete ]

    return 0;
}
```

**How it works:**

- Each permission is mapped to a bit position via an `enum`.
- `std::bitset<5>` holds exactly 5 permission flags.
- `set(pos)` and `reset(pos)` grant/revoke specific permissions.
- `test(pos)` checks if a permission is active - with bounds checking.
- Bitwise `&` computes intersection (both have the permission), `|` computes union.
- This is type-safe and readable compared to raw integer bitmasks.

### Q2: Show bitwise operations on bitsets and how to convert to/from ulong and string

Pay attention to the round-trip at the end: integer -> bitset -> string -> bitset -> integer. This pattern is useful for serialization and debugging. The `to_string` overload with custom characters is a fun touch for visual output.

```cpp
#include <iostream>
#include <bitset>
#include <string>

int main() {
    // --- Construction from different sources ---
    std::bitset<16> from_int(0xCAFE);
    std::bitset<16> from_str("1111000011110000");
    std::bitset<16> from_partial("1010", 4, '0', '1');  // Custom zero/one chars

    std::cout << "from_int:     " << from_int << "\n";
    // Output: from_int:     1100101011111110
    std::cout << "from_str:     " << from_str << "\n";
    // Output: from_str:     1111000011110000
    std::cout << "from_partial: " << from_partial << "\n";
    // Output: from_partial: 0000000000001010

    // --- Bitwise operations ---
    auto and_result = from_int & from_str;
    auto or_result  = from_int | from_str;
    auto xor_result = from_int ^ from_str;
    auto not_result = ~from_int;

    std::cout << "\nBitwise operations:\n";
    std::cout << "AND: " << and_result << " (count=" << and_result.count() << ")\n";
    // Output: AND: 1100000011110000 (count=6)
    std::cout << "OR:  " << or_result  << " (count=" << or_result.count() << ")\n";
    // Output: OR:  1111101011111110 (count=13)
    std::cout << "XOR: " << xor_result << " (count=" << xor_result.count() << ")\n";
    // Output: XOR: 0011101000001110 (count=7)
    std::cout << "NOT: " << not_result << "\n";
    // Output: NOT: 0011010100000001

    // --- Shift operations ---
    std::cout << "\nShifts:\n";
    std::cout << "Left  <<4: " << (from_int << 4) << "\n";
    // Output: Left  <<4: 0101111111100000
    std::cout << "Right >>4: " << (from_int >> 4) << "\n";
    // Output: Right >>4: 0000110010101111

    // --- Conversion ---
    std::bitset<8> b(0b10101010);
    std::cout << "\nConversions:\n";
    std::cout << "to_ulong():  " << b.to_ulong() << "\n";
    // Output: to_ulong():  170
    std::cout << "to_ullong(): " << b.to_ullong() << "\n";
    // Output: to_ullong(): 170
    std::cout << "to_string(): " << b.to_string() << "\n";
    // Output: to_string(): 10101010

    // Custom characters in to_string
    std::cout << "to_string('.','#'): " << b.to_string('.', '#') << "\n";
    // Output: to_string('.','#'): #.#.#.#.

    // Round-trip: int -> bitset -> string -> bitset -> int
    unsigned long original = 42;
    std::bitset<8> bs(original);
    std::string s = bs.to_string();
    std::bitset<8> bs2(s);
    unsigned long back = bs2.to_ulong();
    std::cout << "\nRound-trip: " << original << " -> \"" << s
              << "\" -> " << back << "\n";
    // Output: Round-trip: 42 -> "00101010" -> 42

    return 0;
}
```

**How it works:**

- `std::bitset` supports all bitwise operators: `&`, `|`, `^`, `~`, `<<`, `>>` plus compound variants.
- `to_ulong()` / `to_ullong()` extract the numeric value. Throws if the bitset is too large.
- `to_string()` produces a string of '0' and '1' characters. Custom characters can be specified.
- Construction from `std::string` parses '0'/'1' characters (leftmost char = highest bit).

### Q3: Compare std::bitset with a hand-rolled bitmask approach for readability and safety

The comparison table embedded in this code is worth reading carefully. The short version is: hand-rolled bitmasks are faster to write, `std::bitset` is safer to maintain. For flags whose count is known at compile time, `std::bitset` is almost always the better choice.

```cpp
#include <iostream>
#include <bitset>
#include <stdexcept>
#include <cstdint>

// === Approach 1: Hand-rolled bitmask ===
namespace bitmask {
    constexpr uint32_t FLAG_A = 1u << 0;
    constexpr uint32_t FLAG_B = 1u << 1;
    constexpr uint32_t FLAG_C = 1u << 2;
    constexpr uint32_t FLAG_D = 1u << 3;

    void set(uint32_t& flags, uint32_t f)   { flags |= f; }
    void clear(uint32_t& flags, uint32_t f) { flags &= ~f; }
    bool test(uint32_t flags, uint32_t f)   { return (flags & f) != 0; }
    void toggle(uint32_t& flags, uint32_t f){ flags ^= f; }
}

// === Approach 2: std::bitset ===
namespace bitset_approach {
    enum Flag : std::size_t { A = 0, B = 1, C = 2, D = 3, COUNT = 4 };
    using Flags = std::bitset<COUNT>;
}

int main() {
    // --- Hand-rolled bitmask ---
    uint32_t flags = 0;
    bitmask::set(flags, bitmask::FLAG_A | bitmask::FLAG_C);
    std::cout << "Bitmask: test A=" << bitmask::test(flags, bitmask::FLAG_A)
              << ", test B=" << bitmask::test(flags, bitmask::FLAG_B) << "\n";
    // Output: Bitmask: test A=1, test B=0

    // BUG: Easy to make mistakes with raw integers
    uint32_t wrong = 7;  // Accidentally use wrong value - compiles fine!
    bitmask::set(flags, wrong);  // Sets bits 0, 1, 2 - unintended
    // No bounds checking, no type safety

    // --- std::bitset approach ---
    using namespace bitset_approach;
    Flags bf;
    bf.set(A);
    bf.set(C);
    std::cout << "Bitset:  test A=" << bf.test(A)
              << ", test B=" << bf.test(B) << "\n";
    // Output: Bitset:  test A=1, test B=0

    // Safety: bounds checking with test()
    try {
        bf.test(100);  // throws std::out_of_range
    } catch (const std::out_of_range& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }
    // Output: Caught: bitset::test: __pos (which is 100) >= _Nb (which is 4)

    // Readability: easy printing
    std::cout << "Bitset value: " << bf << "\n";
    // Output: Bitset value: 0101

    std::cout << "Popcount: " << bf.count() << "\n";
    // Output: Popcount: 2

    // --- Comparison Table ---
    // Feature           | Hand-rolled bitmask   | std::bitset
    // ---------------------------------------------------------
    // Size              | Fixed to integer width | Any N (compile-time)
    // Bounds checking   | No                     | Yes (test/set/reset)
    // Type safety       | Weak (just uint32_t)   | Strong (distinct type)
    // Debug printing    | Manual                 | operator<< built-in
    // popcount          | Manual (__builtin_popcount) | .count()
    // Constexpr         | Yes (C++11)            | Limited (C++23 improves)
    // Performance       | Fastest (single int)   | Same or very close
    // Iterators         | No                     | No
    // Max size          | 64 bits                | Unlimited (compile-time)

    return 0;
}
```

**How it works:**

- **Hand-rolled bitmask:** Fast and simple, but error-prone. Any integer can be passed where flags are expected. No bounds checking. Debugging requires manual formatting.
- **`std::bitset`:** Provides named operations (`set`, `reset`, `test`, `flip`), bounds checking, and built-in `operator<<` for printing. The distinct type prevents accidental misuse.
- **Performance:** Both compile down to the same bitwise instructions. `std::bitset` has zero overhead for sizes <= 64 bits (stored in a single word). For larger sizes, it uses an array of words.
- **Recommendation:** Use `std::bitset` when the number of flags is known at compile time and exceeds a few bits. Use raw bitmasks only for ultra-performance-critical hot paths or when interfacing with C APIs.

---

## Notes

- **`std::bitset` is NOT a container:** No iterators, no `begin()`/`end()`, no range-based for loop. To iterate over set bits, loop with `test(i)` or use `_Find_first()`/`_Find_next()` (non-standard GCC extensions).
- **For dynamic sizes**, use `std::vector<bool>` (space-efficient but proxy-reference issues) or `boost::dynamic_bitset`.
- **C++23 improvements:** `constexpr` support for more `std::bitset` operations.
- **Popcount:** `b.count()` is often implemented using hardware popcount instructions (`POPCNT` on x86).
- **Large bitsets:** `std::bitset<1'000'000>` is perfectly fine - stored on the stack if local (careful with stack overflow for very large N), or use `new` / static storage.
