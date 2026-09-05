# Use std::bit_floor and bit_ceil (C++20) for power-of-two alignment

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/bit_ceil>  

---

## Topic Overview

C++20 provides a family of bit-manipulation utilities in `<bit>` that replace common manual bit tricks with clear, portable, `constexpr` functions.

If you have worked with hash tables, ring buffers, GPU textures, or memory allocators, you have almost certainly written or encountered hand-rolled power-of-two arithmetic. The old bit-twiddling versions work but are cryptic, easy to get wrong at edge cases, and not self-documenting. These C++20 functions do the same thing clearly and correctly.

### The Power-of-Two Functions

| Function | Returns | Undefined if |
| --- | --- | --- |
| `std::bit_ceil(x)` | Smallest power of 2 >= x | Result overflows the type |
| `std::bit_floor(x)` | Largest power of 2 <= x | Never (returns 0 if x==0) |
| `std::has_single_bit(x)` | `true` if x is a power of 2 | Never |
| `std::bit_width(x)` | Number of bits needed to represent x | Never |
| `std::countl_zero(x)` | Leading zeros | Never |
| `std::countr_zero(x)` | Trailing zeros | Never |
| `std::popcount(x)` | Number of 1-bits | Never |

All operate on **unsigned integer types** only.

### Old vs New

The old version rounds up to the next power of two by filling in all the lower bits, then adding one. It works, but good luck reading it at a glance.

```cpp
// Old: round up to next power of 2
unsigned next_pow2_old(unsigned n) {
    if (n == 0) return 1;
    --n;
    n |= n >> 1;
    n |= n >> 2;
    n |= n >> 4;
    n |= n >> 8;
    n |= n >> 16;
    return n + 1;
}

// New: one function call
#include <bit>
constexpr unsigned next_pow2_new(unsigned n) {
    return std::bit_ceil(n);
}
```

### Core Syntax

Here is a quick tour of all four main functions together, so you can see what they each produce.

```cpp
#include <bit>
#include <cstdint>
#include <iostream>

int main() {
    // bit_ceil: smallest power of 2 >= n
    std::cout << std::bit_ceil(1u)  << "\n";   // 1
    std::cout << std::bit_ceil(5u)  << "\n";   // 8
    std::cout << std::bit_ceil(8u)  << "\n";   // 8  (already power of 2)
    std::cout << std::bit_ceil(15u) << "\n";   // 16

    // bit_floor: largest power of 2 <= n
    std::cout << std::bit_floor(1u)  << "\n";  // 1
    std::cout << std::bit_floor(5u)  << "\n";  // 4
    std::cout << std::bit_floor(8u)  << "\n";  // 8
    std::cout << std::bit_floor(15u) << "\n";  // 8

    // has_single_bit: is power of 2?
    std::cout << std::boolalpha;
    std::cout << std::has_single_bit(8u)  << "\n";  // true
    std::cout << std::has_single_bit(12u) << "\n";  // false

    // bit_width: min bits needed
    std::cout << std::bit_width(0u)  << "\n";  // 0
    std::cout << std::bit_width(7u)  << "\n";  // 3
    std::cout << std::bit_width(8u)  << "\n";  // 4
}
```

---

## Self-Assessment

### Q1: Use bit_ceil(n) to find the smallest power of two >= n for a buffer size calculation

**Answer:**

Power-of-two buffer sizes come up constantly because they enable fast modulo via a bitmask. Notice how the allocator below makes that optimization explicit.

```cpp
#include <bit>
#include <cstddef>
#include <iostream>
#include <vector>
#include <cassert>

// Allocate a buffer whose size is always a power of 2
// (common in hash tables, circular buffers, GPU textures)
std::vector<char> allocate_aligned_buffer(std::size_t requested) {
    if (requested == 0) requested = 1;
    std::size_t actual = std::bit_ceil(requested);
    std::cout << "requested " << requested
              << " -> allocated " << actual << " bytes\n";
    return std::vector<char>(actual);
}

int main() {
    auto b1 = allocate_aligned_buffer(100);
    // Output: requested 100 -> allocated 128 bytes
    assert(b1.size() == 128);

    auto b2 = allocate_aligned_buffer(256);
    // Output: requested 256 -> allocated 256 bytes
    assert(b2.size() == 256);

    auto b3 = allocate_aligned_buffer(1);
    // Output: requested 1 -> allocated 1 bytes
    assert(b3.size() == 1);

    auto b4 = allocate_aligned_buffer(1000);
    // Output: requested 1000 -> allocated 1024 bytes

    // Power-of-2 sizes enable fast modulo with bitmask:
    std::size_t mask = b4.size() - 1;  // 1023
    std::size_t index = 5000;
    std::size_t wrapped = index & mask;  // same as index % 1024, but faster
    std::cout << "5000 & 1023 = " << wrapped << "\n";
    // Output: 5000 & 1023 = 904
}
```

**Why power-of-2 buffer sizes:**

- Modulo becomes a bitmask: `index % size` -> `index & (size - 1)` - single AND instruction.
- Memory allocators often work in power-of-2 blocks anyway.
- Hash table probing, ring buffers, and texture dimensions all benefit.

---

### Q2: Use bit_floor(n) to find the largest power of two <= n for a block-size calculation

**Answer:**

`bit_floor` is useful when you want to process data in the largest natural chunk that fits the remaining size - for example, to align SIMD processing or divide-and-conquer recursion.

```cpp
#include <bit>
#include <cstddef>
#include <iostream>
#include <cassert>

// Process data in the largest power-of-2 block that fits
// (e.g., SIMD processing where vector width must be power of 2)
void process_in_blocks(const char* data, std::size_t total) {
    std::size_t offset = 0;
    while (offset < total) {
        std::size_t remaining = total - offset;
        std::size_t block = std::bit_floor(remaining);
        if (block == 0) block = 1;  // handle remaining == 0 edge case
        std::cout << "  block at offset " << offset
                  << ", size " << block << "\n";
        // process data[offset .. offset+block)
        offset += block;
    }
}

int main() {
    std::cout << "Processing 100 bytes:\n";
    process_in_blocks(nullptr, 100);
    // Output:
    //   block at offset 0, size 64
    //   block at offset 64, size 32
    //   block at offset 96, size 4

    std::cout << "\nProcessing 17 bytes:\n";
    process_in_blocks(nullptr, 17);
    // Output:
    //   block at offset 0, size 16
    //   block at offset 16, size 1

    // bit_floor examples
    std::cout << "\nbit_floor(0)  = " << std::bit_floor(0u) << "\n";   // 0
    std::cout << "bit_floor(1)  = " << std::bit_floor(1u) << "\n";   // 1
    std::cout << "bit_floor(6)  = " << std::bit_floor(6u) << "\n";   // 4
    std::cout << "bit_floor(10) = " << std::bit_floor(10u) << "\n";  // 8
    std::cout << "bit_floor(16) = " << std::bit_floor(16u) << "\n";  // 16
}
```

---

### Q3: Show that these are constexpr and can replace manual (n & (n-1)) == 0 tricks at compile time

**Answer:**

Because all `<bit>` functions are `constexpr`, you can use them in `static_assert`, `requires` clauses, template non-type parameters, and anywhere else the compiler needs to evaluate something at compile time. The old bit-twiddling tricks were also `constexpr`-able, but the new versions are far more readable and less error-prone.

```cpp
#include <bit>
#include <cstdint>
#include <iostream>

// Old trick: check if n is a power of 2
constexpr bool is_power_of_2_old(unsigned n) {
    return n > 0 && (n & (n - 1)) == 0;
}

// New: readable, self-documenting
constexpr bool is_power_of_2_new(unsigned n) {
    return std::has_single_bit(n);
}

// Old trick: round up to next power of 2
constexpr unsigned round_up_old(unsigned n) {
    if (n <= 1) return 1;
    --n;
    n |= n >> 1; n |= n >> 2; n |= n >> 4;
    n |= n >> 8; n |= n >> 16;
    return n + 1;
}

// New: one call
constexpr unsigned round_up_new(unsigned n) {
    return n <= 1 ? 1 : std::bit_ceil(n);
}

// Compile-time proof
static_assert(is_power_of_2_old(8) == is_power_of_2_new(8));
static_assert(is_power_of_2_old(7) == is_power_of_2_new(7));
static_assert(round_up_old(100)    == round_up_new(100));
static_assert(round_up_old(64)     == round_up_new(64));

// All bit functions are constexpr
static_assert(std::bit_ceil(5u)  == 8);
static_assert(std::bit_floor(5u) == 4);
static_assert(std::has_single_bit(16u) == true);
static_assert(std::bit_width(15u) == 4);
static_assert(std::countl_zero(uint8_t{0b00110000}) == 2);
static_assert(std::countr_zero(uint8_t{0b00110000}) == 4);
static_assert(std::popcount(0b10110101u) == 5);

// Template parameter using constexpr bit functions
template<unsigned N>
    requires (std::has_single_bit(N))  // C++20 concept constraint
struct PowerOfTwoBuffer {
    char data[N];
    static constexpr unsigned mask = N - 1;  // fast modulo

    char& operator[](unsigned i) { return data[i & mask]; }
};

int main() {
    PowerOfTwoBuffer<64> buf;
    buf[0] = 'A';
    buf[64] = 'B';   // wraps to index 0
    std::cout << buf[0] << "\n";  // Output: B

    // PowerOfTwoBuffer<100> bad;  // COMPILE ERROR: 100 is not power of 2

    std::cout << "All static_asserts passed!\n";
    std::cout << "bit_ceil(100)  = " << std::bit_ceil(100u) << "\n";  // 128
    std::cout << "bit_floor(100) = " << std::bit_floor(100u) << "\n"; // 64
    std::cout << "popcount(0xFF) = " << std::popcount(0xFFu) << "\n"; // 8
}
```

**Advantages of the standard functions:**

- **Readable:** `std::has_single_bit(n)` is clearer than `(n & (n-1)) == 0`.
- **Correct:** The bit-twiddling tricks are easy to get wrong for edge cases (0, 1, max values).
- **Portable:** Compilers map these to hardware instructions (`BSR`, `LZCNT`, `POPCNT`, `CLZ`).
- **`constexpr`:** Usable in template arguments, `static_assert`, `requires` clauses.

---

## Notes

- All `<bit>` functions work only with **unsigned** integer types - passing a signed type is a compile error.
- `std::bit_ceil` has undefined behavior if the result would overflow the type. For `uint32_t`, `bit_ceil(x)` is UB if `x > 2^31`.
- `std::bit_floor(0)` returns 0 (not 1).
- On x86, `std::popcount` compiles to the `POPCNT` instruction, `countl_zero` to `LZCNT`/`BSR`, etc.
- These functions are in `<bit>`, not `<cmath>` or `<numeric>`.
