# Use `std::bit_width`, `std::countl_zero`, and Other Bit Utilities (C++20)

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/bit_width>  

---

## Topic Overview

### C++20 `<bit>` Header

Before C++20, if you needed low-level bit operations you had two options: write your own loop-based implementations, or reach for compiler-specific intrinsics like `__builtin_clz` or `_BitScanReverse`. Both approaches have real downsides - the first is slow, the second is not portable.

C++20 fixed this by introducing a comprehensive set of `constexpr` bit manipulation functions in the `<bit>` header. These functions map directly to hardware instructions on every major platform, so you get portability and performance at the same time. They also work at compile time, which means you can use them in `static_assert` checks and `constexpr` computations. That is a genuinely big deal - before this, any compile-time bit logic had to be expressed with awkward template metaprogramming tricks.

### Function Summary

Here is a quick reference for all the functions the header provides. Each one operates on an unsigned integer of any width. The examples column uses the `->` notation to mean "returns".

| Function | Description | Example |
| --- | --- | --- |
| `std::bit_width(x)` | Min bits to represent `x` (ceil(log2(x))+1) | `bit_width(8u)` -> 4 |
| `std::countl_zero(x)` | Count leading zeros | `countl_zero(uint8_t(1))` -> 7 |
| `std::countl_one(x)` | Count leading ones | `countl_one(uint8_t(0xF0))` -> 4 |
| `std::countr_zero(x)` | Count trailing zeros | `countr_zero(8u)` -> 3 |
| `std::countr_one(x)` | Count trailing ones | `countr_one(7u)` -> 3 |
| `std::popcount(x)` | Count set bits | `popcount(0b1011u)` -> 3 |
| `std::has_single_bit(x)` | Is power of 2? | `has_single_bit(8u)` -> true |
| `std::bit_ceil(x)` | Round up to next power of 2 | `bit_ceil(5u)` -> 8 |
| `std::bit_floor(x)` | Round down to power of 2 | `bit_floor(5u)` -> 4 |
| `std::rotl(x, n)` | Rotate bits left | `rotl(1u, 3)` -> 8 |
| `std::rotr(x, n)` | Rotate bits right | `rotr(8u, 3)` -> 1 |

**All functions are `constexpr`** - they work at compile time and runtime.

**Important:** All functions require **unsigned integer types**. If you have a signed value, cast it to unsigned first. Passing a signed type is undefined behavior (or a compilation error, depending on the implementation).

---

## Self-Assessment

### Q1: Use `std::bit_width(n)` to find the number of bits needed to represent `n`

`std::bit_width(x)` answers the question "how many bits do I need to store the value `x`?" That is exactly log2(x) rounded up, plus one to account for the highest set bit. You will often use this when sizing bit fields, allocating compact storage, or building compression schemes.

Take a look at the range of values and what `bit_width` returns for each of them - the pattern becomes obvious pretty quickly:

```cpp
#include <iostream>
#include <bit>
#include <cstdint>

int main() {
    std::cout << "=== std::bit_width ===\n";
    std::cout << "bit_width(0)   = " << std::bit_width(0u) << "\n";    // 0
    std::cout << "bit_width(1)   = " << std::bit_width(1u) << "\n";    // 1
    std::cout << "bit_width(2)   = " << std::bit_width(2u) << "\n";    // 2
    std::cout << "bit_width(3)   = " << std::bit_width(3u) << "\n";    // 2
    std::cout << "bit_width(4)   = " << std::bit_width(4u) << "\n";    // 3
    std::cout << "bit_width(7)   = " << std::bit_width(7u) << "\n";    // 3
    std::cout << "bit_width(8)   = " << std::bit_width(8u) << "\n";    // 4
    std::cout << "bit_width(255) = " << std::bit_width(255u) << "\n";  // 8
    std::cout << "bit_width(256) = " << std::bit_width(256u) << "\n";  // 9

    // Compile-time verification
    static_assert(std::bit_width(0u) == 0);
    static_assert(std::bit_width(1u) == 1);
    static_assert(std::bit_width(255u) == 8);
    static_assert(std::bit_width(256u) == 9);

    // === Practical use: determine minimum storage size ===
    constexpr unsigned max_value = 1000;
    constexpr auto bits_needed = std::bit_width(max_value);  // 10 bits
    static_assert(bits_needed == 10);

    std::cout << "\nTo represent values 0.." << max_value
              << " you need " << bits_needed << " bits\n";

    // === Other counting functions ===
    std::cout << "\n=== Counting Functions ===\n";
    uint8_t val = 0b00101100;
    std::cout << "Value: 0b00101100\n";
    std::cout << "countl_zero:  " << std::countl_zero(val) << "\n";   // 2
    std::cout << "countr_zero:  " << std::countr_zero(val) << "\n";   // 2
    std::cout << "popcount:     " << std::popcount(val) << "\n";      // 3
    std::cout << "has_single_bit: " << std::has_single_bit(val) << "\n"; // false

    return 0;
}
```

Notice how the `static_assert` lines prove the function is truly `constexpr` - the compiler evaluates those checks at compile time and the program would not compile if any of them were wrong. The `bits_needed` constant is also `constexpr`, which means you can use it as a template argument or in another `static_assert` downstream.

**Expected output:**

```text
=== std::bit_width ===
bit_width(0)   = 0
bit_width(1)   = 1
bit_width(2)   = 2
bit_width(3)   = 2
bit_width(4)   = 3
bit_width(7)   = 3
bit_width(8)   = 4
bit_width(255) = 8
bit_width(256) = 9

To represent values 0..1000 you need 10 bits

=== Counting Functions ===
Value: 0b00101100
countl_zero:  2
countr_zero:  2
popcount:     3
has_single_bit: 0
```

### Q2: Implement power-of-two rounding up using `std::bit_ceil`

Rounding up to the next power of two is a surprisingly common operation. Allocators often need alignment boundaries that are powers of two, and hash tables almost always size their bucket arrays at powers of two so that a bitmask can replace a modulo operation. `std::bit_ceil` gives you this rounding in a single call, and because it is `constexpr`, you can do it at compile time for fixed sizes.

The example also shows `std::bit_floor`, which rounds down instead of up - useful when you need the largest power of two that fits within a given value.

```cpp
#include <iostream>
#include <bit>
#include <cstdint>
#include <vector>

// === std::bit_ceil: round up to next power of 2 ===
// bit_ceil(n) returns the smallest power of 2 >= n

// Practical use: allocator alignment
constexpr std::size_t aligned_size(std::size_t requested) {
    if (requested == 0) return 1;
    return std::bit_ceil(requested);
}

// Practical use: hash table capacity
constexpr std::size_t next_capacity(std::size_t min_size) {
    return std::bit_ceil(min_size);
}

// Compile-time verification
static_assert(std::bit_ceil(1u) == 1);
static_assert(std::bit_ceil(2u) == 2);
static_assert(std::bit_ceil(3u) == 4);
static_assert(std::bit_ceil(5u) == 8);
static_assert(std::bit_ceil(9u) == 16);
static_assert(std::bit_ceil(255u) == 256);
static_assert(std::bit_ceil(256u) == 256);

static_assert(aligned_size(100) == 128);
static_assert(aligned_size(1024) == 1024);

// === bit_floor: round down to power of 2 ===
static_assert(std::bit_floor(1u) == 1);
static_assert(std::bit_floor(5u) == 4);
static_assert(std::bit_floor(7u) == 4);
static_assert(std::bit_floor(8u) == 8);

int main() {
    std::cout << "=== std::bit_ceil (round up to power of 2) ===\n";
    for (unsigned n : {1u, 2u, 3u, 5u, 7u, 8u, 9u, 100u, 255u, 256u, 1000u}) {
        std::cout << "bit_ceil(" << n << ") = " << std::bit_ceil(n) << "\n";
    }

    std::cout << "\n=== std::bit_floor (round down to power of 2) ===\n";
    for (unsigned n : {1u, 2u, 3u, 5u, 7u, 8u, 9u, 100u, 255u, 256u}) {
        std::cout << "bit_floor(" << n << ") = " << std::bit_floor(n) << "\n";
    }

    std::cout << "\n=== Practical: Allocator Alignment ===\n";
    for (std::size_t req : {50, 100, 200, 500, 1000, 4096}) {
        std::cout << "Requested " << req << " bytes -> aligned to "
                  << aligned_size(req) << " bytes\n";
    }

    std::cout << "\n=== Practical: Hash Table Capacity ===\n";
    for (std::size_t n : {10, 50, 100, 1000}) {
        std::cout << "Min " << n << " buckets -> capacity " << next_capacity(n) << "\n";
    }

    return 0;
}
```

The `aligned_size` and `next_capacity` helpers are both `constexpr`, so you can use them in template arguments or `static_assert` checks to verify sizing decisions at compile time. That is exactly the kind of thing that used to require platform-specific macros or manual counting.

**Expected output:**

```text
=== std::bit_ceil (round up to power of 2) ===
bit_ceil(1) = 1
bit_ceil(2) = 2
bit_ceil(3) = 4
bit_ceil(5) = 8
bit_ceil(7) = 8
bit_ceil(8) = 8
bit_ceil(9) = 16
bit_ceil(100) = 128
bit_ceil(255) = 256
bit_ceil(256) = 256
bit_ceil(1000) = 1024

=== std::bit_floor (round down to power of 2) ===
bit_floor(1) = 1
bit_floor(2) = 2
bit_floor(3) = 2
bit_floor(5) = 4
bit_floor(7) = 4
bit_floor(8) = 8
bit_floor(9) = 8
bit_floor(100) = 64
bit_floor(255) = 128
bit_floor(256) = 256

=== Practical: Allocator Alignment ===
Requested 50 bytes -> aligned to 64 bytes
Requested 100 bytes -> aligned to 128 bytes
Requested 200 bytes -> aligned to 256 bytes
Requested 500 bytes -> aligned to 512 bytes
Requested 1000 bytes -> aligned to 1024 bytes
Requested 4096 bytes -> aligned to 4096 bytes

=== Practical: Hash Table Capacity ===
Min 10 buckets -> capacity 16
Min 50 buckets -> capacity 64
Min 100 buckets -> capacity 128
Min 1000 buckets -> capacity 1024
```

### Q3: Use `countl_zero` to implement a fast `log2` floor for positive integers

The relationship between `countl_zero` and `log2` is one of those elegant bit-twiddling observations worth knowing. For a 32-bit value, the number of leading zeros tells you exactly where the highest set bit lives, which is the same as the floor of log2. What makes this especially useful is that the compiler maps `countl_zero` directly to a single hardware instruction - `BSR` or `LZCNT` on x86, `CLZ` on ARM - so the whole `log2_floor` call compiles down to one instruction.

Here is the implementation with both a clean version and a version that shows the `countl_zero` relationship explicitly:

```cpp
#include <iostream>
#include <bit>
#include <cstdint>
#include <limits>
#include <cassert>

// === log2_floor using countl_zero ===
// countl_zero counts leading zero bits
// For an N-bit type, log2_floor(x) = N - 1 - countl_zero(x)
// This compiles to a single BSR/LZCNT instruction on x86

constexpr int log2_floor(unsigned x) {
    assert(x > 0);
    return static_cast<int>(std::bit_width(x)) - 1;
    // Equivalent to: (sizeof(unsigned)*8 - 1) - std::countl_zero(x)
}

// Alternative implementation showing countl_zero directly
constexpr int log2_floor_v2(unsigned x) {
    assert(x > 0);
    constexpr int bits = std::numeric_limits<unsigned>::digits;
    return bits - 1 - std::countl_zero(x);
}

// log2_ceil: round up
constexpr int log2_ceil(unsigned x) {
    if (x <= 1) return 0;
    return static_cast<int>(std::bit_width(x - 1));
}

// Compile-time verification
static_assert(log2_floor(1) == 0);
static_assert(log2_floor(2) == 1);
static_assert(log2_floor(3) == 1);
static_assert(log2_floor(4) == 2);
static_assert(log2_floor(7) == 2);
static_assert(log2_floor(8) == 3);
static_assert(log2_floor(255) == 7);
static_assert(log2_floor(256) == 8);
static_assert(log2_floor(1024) == 10);

static_assert(log2_ceil(1) == 0);
static_assert(log2_ceil(2) == 1);
static_assert(log2_ceil(3) == 2);
static_assert(log2_ceil(4) == 2);
static_assert(log2_ceil(5) == 3);

int main() {
    std::cout << "=== log2_floor using countl_zero ===\n";
    for (unsigned n : {1u, 2u, 3u, 4u, 7u, 8u, 15u, 16u, 255u, 256u, 1023u, 1024u}) {
        std::cout << "log2_floor(" << n << ") = " << log2_floor(n)
                  << "  (countl_zero = " << std::countl_zero(n) << ")\n";
    }

    std::cout << "\n=== log2_ceil ===\n";
    for (unsigned n : {1u, 2u, 3u, 4u, 5u, 8u, 9u, 16u, 17u}) {
        std::cout << "log2_ceil(" << n << ") = " << log2_ceil(n) << "\n";
    }

    std::cout << "\n=== How countl_zero Works ===\n";
    std::cout << "countl_zero counts leading 0s in the binary representation.\n";
    std::cout << "For 32-bit unsigned:\n";
    for (unsigned n : {1u, 2u, 128u, 255u, 256u}) {
        std::cout << "  " << n << " (0x" << std::hex << n << std::dec << "): "
                  << std::countl_zero(n) << " leading zeros -> log2 = "
                  << log2_floor(n) << "\n";
    }

    std::cout << "\n=== Assembly (x86-64 with -O2) ===\n";
    std::cout << "log2_floor(x):         bsr eax, edi   (1 instruction!)\n";
    std::cout << "Manual loop counting:  10+ instructions\n";
    std::cout << "The compiler maps countl_zero to BSR/LZCNT hardware instruction.\n";

    return 0;
}
```

The key insight here is in the `log2_floor_v2` implementation: `bits - 1 - countl_zero(x)` gives you the position of the highest set bit, counting from zero. The two implementations give identical results; `log2_floor` just expresses it more readably via `bit_width`. Both compile to the same single hardware instruction.

**Expected output:**

```text
=== log2_floor using countl_zero ===
log2_floor(1) = 0  (countl_zero = 31)
log2_floor(2) = 1  (countl_zero = 30)
log2_floor(3) = 1  (countl_zero = 30)
log2_floor(4) = 2  (countl_zero = 29)
log2_floor(7) = 2  (countl_zero = 29)
log2_floor(8) = 3  (countl_zero = 28)
log2_floor(15) = 3  (countl_zero = 28)
log2_floor(16) = 4  (countl_zero = 27)
log2_floor(255) = 7  (countl_zero = 24)
log2_floor(256) = 8  (countl_zero = 23)
log2_floor(1023) = 9  (countl_zero = 22)
log2_floor(1024) = 10  (countl_zero = 21)

=== log2_ceil ===
log2_ceil(1) = 0
log2_ceil(2) = 1
log2_ceil(3) = 2
log2_ceil(4) = 2
log2_ceil(5) = 3
log2_ceil(8) = 3
log2_ceil(9) = 4
log2_ceil(16) = 4
log2_ceil(17) = 5

=== How countl_zero Works ===
countl_zero counts leading 0s in the binary representation.
For 32-bit unsigned:
  1 (0x1): 31 leading zeros -> log2 = 0
  2 (0x2): 30 leading zeros -> log2 = 1
  128 (0x80): 24 leading zeros -> log2 = 7
  255 (0xff): 24 leading zeros -> log2 = 7
  256 (0x100): 23 leading zeros -> log2 = 8

=== Assembly (x86-64 with -O2) ===
log2_floor(x):         bsr eax, edi   (1 instruction!)
Manual loop counting:  10+ instructions
The compiler maps countl_zero to BSR/LZCNT hardware instruction.
```

---

## Notes

- All `<bit>` functions are `constexpr` and work on **unsigned integer types** only.
- `std::bit_width(x)` = ceil(log2(x+1)) = number of bits needed to represent `x`.
- `std::bit_ceil(x)` rounds up to the nearest power of 2 - essential for allocators and hash tables.
- `std::countl_zero` maps to hardware instructions (`BSR`/`LZCNT` on x86, `CLZ` on ARM).
- `std::popcount` maps to `POPCNT` instruction - one cycle on modern CPUs.
- `std::has_single_bit(x)` is the standard way to check if `x` is a power of 2.
- For signed integers, cast to unsigned first: `std::bit_width(static_cast<unsigned>(x))`.
