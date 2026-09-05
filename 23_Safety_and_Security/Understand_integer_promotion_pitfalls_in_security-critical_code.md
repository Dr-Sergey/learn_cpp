# Understand integer promotion pitfalls in security-critical code

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/implicit_conversion>  

---

## Topic Overview

Integer promotion and implicit conversions are among the most dangerous bug classes in security-critical C++ code. The compiler silently promotes, narrows, and sign-converts integers, creating bugs that are invisible in code review but exploitable by attackers. These lead to buffer overflows, allocation size errors, and comparison logic bypasses.

The reason this trips people up constantly is that the bugs look like correct code. The math appears fine at a glance - the problem only becomes visible when you think about what types the compiler is actually using, which are often not what you'd guess.

### Integer Promotion Rules

Here's a condensed version of the rules. The critical one for security is the mixed signed/unsigned case - when you compare or do arithmetic with a signed and unsigned value, the signed value silently becomes unsigned, which can flip its meaning entirely:

```cpp
C++ integer promotion (simplified):

Types narrower than int:
  char, signed char, unsigned char, short, unsigned short, bool
  -> ALL promoted to int (or unsigned int if int can't hold all values)

Binary operations with mixed types:
  signed + unsigned -> unsigned  (sign lost!)
  smaller + larger  -> larger
  int + long        -> long
  int + unsigned    -> unsigned int

Assignment/initialization:
  Narrowing conversions are implicit (no error without -Wconversion)
  int x = 100000; char c = x;  // silently truncates!
```

That first rule in the binary operations section - `signed + unsigned -> unsigned` - is the one that enables entire categories of security bugs. A negative validation index becomes an enormous positive number when compared against an unsigned size.

### Common Vulnerability Patterns

| Pattern | Bug | Security impact |
| --- | --- | --- |
| `uint32_t len - offset` when `len < offset` | Unsigned underflow -> huge value | Buffer over-read/overflow |
| `if (signed_len < 0)` on unsigned type | Condition never true | Bypass validation |
| `malloc(count * sizeof(T))` | Integer overflow in multiplication | Heap overflow |
| `char c = 0xFF; if (c == 0xFF)` | Sign extension on comparison | Logic bypass |
| `size_t index = user_int` | Negative -> huge positive | Array out-of-bounds |

### Core Example

These two examples show the bugs as they actually happen in real code - not as abstract rules, but as concrete arithmetic that produces wildly unexpected results:

```cpp
#include <cstdint>
#include <iostream>

int main() {
    // Unsigned underflow
    uint32_t packet_len = 2;    // attacker sends tiny packet
    uint32_t header_size = 4;
    uint32_t payload_len = packet_len - header_size;
    // payload_len = 4294967294 (0xFFFFFFFE) - enormous!
    // If used as memcpy size -> catastrophic buffer overflow

    std::cout << "payload_len = " << payload_len << "\n";
    // Output: payload_len = 4294967294

    // Signed/unsigned comparison
    int user_offset = -1;      // attacker sends -1
    unsigned int buffer_size = 1024;
    if (user_offset < buffer_size) {
        // Compiler converts user_offset to unsigned: -1 -> 4294967295
        // 4294967295 < 1024 is FALSE
        // Validation BYPASSED!
        std::cout << "Should be here but isn't\n";
    } else {
        std::cout << "Validation bypassed!\n";
    }
    // Output: Validation bypassed!
}
```

That `payload_len = 4294967294` is not a typo - that's what happens when you subtract 4 from 2 using unsigned arithmetic. And that's the number that would be passed to `memcpy`.

---

## Self-Assessment

### Q1: Show the sign extension bug: uint8_t b = 0xFF; int x = b; x is 255, not -1 - but verify

**Answer:**

This one is subtler than it looks. Whether `0xFF` becomes 255 or -1 when promoted to `int` depends entirely on whether the original type is signed or unsigned. The security implication is that `char` (whose signedness is implementation-defined) can silently become a huge array index:

```cpp
#include <cstdint>
#include <iostream>
#include <bitset>

int main() {
    // uint8_t is UNSIGNED - no sign extension

    uint8_t b = 0xFF;  // 255 unsigned (bit pattern: 1111 1111)
    int x = b;          // promoted to int: 0x000000FF = 255

    std::cout << "uint8_t b = 0xFF:\n";
    std::cout << "  b as int = " << static_cast<int>(b) << "\n";      // 255
    std::cout << "  x        = " << x << "\n";                         // 255
    std::cout << "  bits     = " << std::bitset<32>(x) << "\n";
    // 00000000 00000000 00000000 11111111  (no sign extension)

    // uint8_t is unsigned char - bit 7 is NOT a sign bit.
    // Promotion to int preserves the unsigned value: 0xFF -> 255.

    // BUT: int8_t IS signed - sign extension DOES occur

    int8_t sb = static_cast<int8_t>(0xFF);  // -1 signed (two's complement)
    int sx = sb;                              // promoted to int: 0xFFFFFFFF = -1

    std::cout << "\nint8_t sb = (int8_t)0xFF:\n";
    std::cout << "  sb as int = " << static_cast<int>(sb) << "\n";    // -1
    std::cout << "  sx        = " << sx << "\n";                       // -1
    std::cout << "  bits      = " << std::bitset<32>(sx) << "\n";
    // 11111111 11111111 11111111 11111111  (sign-extended!)

    // Security implication

    // Reading a byte from network as signed char:
    int8_t  network_byte = static_cast<int8_t>(0xFF);
    size_t  index1 = static_cast<size_t>(network_byte);
    // index1 = 18446744073709551615 (on 64-bit) - sign extension then unsigned!
    // -> Array out-of-bounds!

    uint8_t safe_byte = 0xFF;
    size_t  index2 = safe_byte;
    // index2 = 255 - correct!

    std::cout << "\nSecurity impact:\n";
    std::cout << "  signed byte 0xFF as size_t:   " << index1 << "\n";
    std::cout << "  unsigned byte 0xFF as size_t: " << index2 << "\n";

    // RULE: Always use unsigned types (uint8_t) for raw byte data!
    // char may be signed on some platforms (x86), unsigned on others (ARM).

    // The char trap
    // `char` signedness is implementation-defined!
    char c = 0xFF;  // May be 255 OR -1 depending on platform
    if (c == 0xFF) {
        std::cout << "\nchar is unsigned on this platform\n";
    } else {
        std::cout << "\nchar is SIGNED: c = " << static_cast<int>(c) << "\n";
        // c = -1, and -1 != 255 (after promotion to int)
    }
}
// Output (typical x86-64):
// uint8_t b = 0xFF:
//   b as int = 255
//   x        = 255
//   bits     = 00000000000000000000000011111111
//
// int8_t sb = (int8_t)0xFF:
//   sb as int = -1
//   sx        = -1
//   bits      = 11111111111111111111111111111111
//
// Security impact:
//   signed byte 0xFF as size_t:   18446744073709551615
//   unsigned byte 0xFF as size_t: 255
//
// char is SIGNED: c = -1
```

`uint8_t` is unsigned - promoting `0xFF` to `int` gives 255, not -1. There's no sign extension because the high bit isn't a sign bit. However, `int8_t` (signed) and `char` (implementation-defined signedness) do sign-extend: `0xFF` becomes -1, which when cast to `size_t` becomes an enormous value. Always use unsigned types for raw byte data.

### Q2: Demonstrate a comparison inversion: (uint32_t)len - 4 > 0 is always true when len < 4

**Answer:**

This is a pattern that has caused dozens of real CVEs in network parsers. The check looks like it validates a minimum packet size, but unsigned arithmetic inverts the logic for small inputs:

```cpp
#include <cstdint>
#include <iostream>
#include <cstddef>

// The Bug: Unsigned underflow

bool vulnerable_check(uint32_t packet_length) {
    // Intent: reject packets with less than 4 bytes of payload
    // Bug: unsigned subtraction wraps around!
    uint32_t payload_size = packet_length - 4;

    if (payload_size > 0) {
        // When packet_length = 3:
        //   payload_size = 3 - 4 = 4294967295 (0xFFFFFFFF)
        //   4294967295 > 0 -> TRUE -> validation bypassed!
        return true;  // "valid" packet
    }
    return false;
}

// The Fix

bool safe_check(uint32_t packet_length) {
    // Fix: compare BEFORE subtracting
    if (packet_length <= 4) {
        return false;  // too short
    }
    uint32_t payload_size = packet_length - 4;  // safe: packet_length > 4
    return payload_size > 0;
}

// Real-world CVE patterns

void process_packet(const uint8_t* data, uint32_t total_len) {
    constexpr uint32_t HEADER_SIZE = 8;

    // VULNERABLE: signed/unsigned comparison
    // int remaining = total_len - HEADER_SIZE;
    // Even if remaining is signed int, the subtraction is done as unsigned
    // (total_len is uint32_t), then the result is stored in int.
    // If total_len=4: 4u - 8u = 0xFFFFFFFC -> stored in int = -4
    // Seems to work... BUT:

    // VULNERABLE: size_t comparison
    // size_t remaining = total_len - HEADER_SIZE;
    // 4u - 8u = huge unsigned value -> memcpy(dst, data, remaining) overflows!

    // SAFE: check before arithmetic
    if (total_len < HEADER_SIZE) {
        std::cerr << "Packet too short\n";
        return;
    }
    uint32_t remaining = total_len - HEADER_SIZE;  // now guaranteed safe

    std::cout << "Processing " << remaining << " bytes of payload\n";
}

int main() {
    std::cout << "=== Unsigned underflow demo ===\n\n";

    // Show the bug for several values
    for (uint32_t len : {0u, 1u, 2u, 3u, 4u, 5u, 10u}) {
        uint32_t result = len - 4;
        bool vuln = vulnerable_check(len);
        bool safe = safe_check(len);

        std::cout << "len=" << len
                  << "  len-4=" << result
                  << "  vulnerable=" << (vuln ? "PASS" : "reject")
                  << "  safe=" << (safe ? "PASS" : "reject") << "\n";
    }

    // Output:
    // len=0  len-4=4294967292  vulnerable=PASS  safe=reject
    // len=1  len-4=4294967293  vulnerable=PASS  safe=reject
    // len=2  len-4=4294967294  vulnerable=PASS  safe=reject
    // len=3  len-4=4294967295  vulnerable=PASS  safe=reject
    // len=4  len-4=0           vulnerable=reject  safe=reject
    // len=5  len-4=1           vulnerable=PASS  safe=PASS
    // len=10 len-4=6           vulnerable=PASS  safe=PASS

    std::cout << "\n=== Packet processing ===\n";
    uint8_t dummy[16]{};
    process_packet(dummy, 4);   // "Packet too short"
    process_packet(dummy, 16);  // "Processing 8 bytes of payload"

    // RULE: Always compare BEFORE subtracting unsigned values.
    // RULE: Use std::cmp_less (C++20) for signed/unsigned comparisons.
}
```

Unsigned arithmetic wraps: `3u - 4u` is not `-1` but `4294967295`. The comparison `payload_size > 0` is therefore always true for any `len < 4`, bypassing the security check entirely. The fix: always check `len >= required_minimum` before performing the subtraction. This pattern is the root cause of dozens of real-world CVEs in network packet parsers.

### Q3: Use -Wconversion and -Wsign-compare to catch implicit narrowing and sign mismatch

**Answer:**

These two compiler flags together catch most of the implicit conversion bugs that matter for security. The idea is to make the compiler tell you every time it's silently doing something that could go wrong:

```cpp
#include <cstdint>
#include <cstddef>
#include <iostream>
#include <vector>
#include <type_traits>

// Compile: g++ -std=c++20 -Wall -Wextra -Wconversion -Wsign-compare -Wpedantic warnings.cpp

// Code that triggers warnings

void warning_examples() {
    // -Wconversion: implicit narrowing

    int big = 100000;
    // short small = big;
    // Warning: conversion from 'int' to 'short' may change value [-Wconversion]

    uint32_t u32 = 300;
    // uint8_t u8 = u32;
    // Warning: conversion from 'uint32_t' to 'uint8_t' may change value

    double pi = 3.14159;
    // int truncated = pi;
    // Warning: conversion from 'double' to 'int' may change value

    size_t sz = 42;
    // int i = sz;
    // Warning: conversion from 'size_t' to 'int' may change value

    // -Wsign-compare: signed/unsigned comparison

    int index = -1;
    std::vector<int> vec{1, 2, 3};

    // if (index < vec.size()) { ... }
    // Warning: comparison of integer expressions of different signedness:
    //          'int' and 'std::vector<int>::size_type' [-Wsign-compare]
    // Bug: -1 converted to unsigned -> huge value -> comparison is false!

    // -Wsign-conversion (subset of -Wconversion)

    unsigned u = 42;
    // int s = u;
    // Warning: conversion from 'unsigned int' to 'int' may change value

    int neg = -5;
    // unsigned abs_val = neg;
    // Warning: conversion from 'int' to 'unsigned int' changes value
}

// Fixed versions

void fixed_examples() {
    // Fix 1: Use explicit casts when narrowing is intentional
    int big = 100000;
    auto small = static_cast<short>(big);  // explicit: reviewer sees the intent
    (void)small;

    // Fix 2: Use matching types
    uint32_t u32 = 300;
    uint32_t u32_copy = u32;  // no narrowing
    (void)u32_copy;

    // Fix 3: Use size_t consistently for container indices
    std::vector<int> vec{1, 2, 3};
    for (size_t i = 0; i < vec.size(); ++i) {
        // No warning: both are unsigned, same type
        std::cout << vec[i] << " ";
    }
    std::cout << "\n";

    // Fix 4: Use C++20 std::cmp_less for mixed-sign comparison
    int index = -1;
    if (std::cmp_less(index, vec.size())) {
        // std::cmp_less correctly handles: -1 IS less than 3
        std::cout << "cmp_less: -1 < 3 (correct!)\n";
    }

    // Fix 5: Use gsl::narrow for checked narrowing
    // auto byte = gsl::narrow<uint8_t>(u32);  // throws if value doesn't fit

    // Fix 6: Use std::in_range (C++20) for range checks
    int64_t big_val = 1'000'000;
    if (std::in_range<uint8_t>(big_val)) {
        auto byte = static_cast<uint8_t>(big_val);
        (void)byte;
    } else {
        std::cout << big_val << " does not fit in uint8_t\n";
    }
}

// Recommended compiler flags
//
//  Flag                  What it catches
//  -----------------     ----------------------------------------
//  -Wconversion          All implicit narrowing conversions
//  -Wsign-compare        Signed/unsigned comparison
//  -Wsign-conversion     Signed/unsigned assignment
//  -Warith-conversion    Implicit arithmetic conversions (GCC 10+)
//  -Wfloat-conversion    Float -> integer conversion
//  -Wshorten-64-to-32    64-bit to 32-bit (Clang)
//
//  Recommended baseline:
//  -Wall -Wextra -Wpedantic -Wconversion -Wsign-compare -Werror
//
//  For security-critical code, also:
//  -fsanitize=integer     (UBSan integer checks: overflow, shift, truncation)
//  -ftrapv                (trap on signed overflow - older alternative)

int main() {
    fixed_examples();

    // Output:
    // 1 2 3
    // cmp_less: -1 < 3 (correct!)
    // 1000000 does not fit in uint8_t
}
```

`-Wconversion` catches implicit narrowing (`int` -> `short`, `uint32_t` -> `uint8_t`) that silently truncates values. `-Wsign-compare` catches comparisons between signed and unsigned types where the signed value is implicitly converted to unsigned (making `-1` into a huge positive number). Enable both with `-Werror` in CI. For runtime detection, use `-fsanitize=integer` (UBSan) to trap overflows and truncations at runtime.

---

## Notes

- **CWE-190 (Integer Overflow or Wraparound)** and **CWE-681 (Incorrect Conversion between Numeric Types)** are perennially in the Top 25.
- **`std::cmp_less`** etc. (C++20) correctly compare signed and unsigned integers - use them instead of raw `<`.
- **`std::in_range<T>(value)`** (C++20) checks whether a value fits in type `T` without triggering UB.
- **`-fsanitize=integer`** is a UBSan sub-check that catches unsigned overflow (normally defined behavior but often a bug), signed overflow, shift errors, and implicit truncation at runtime.
- **`char` signedness is implementation-defined** - on x86 (GCC default) it's signed, on ARM it's unsigned. Always use `uint8_t` for raw bytes.
- Compile with `-std=c++20 -Wall -Wextra -Wconversion -Wsign-compare -Werror`.
