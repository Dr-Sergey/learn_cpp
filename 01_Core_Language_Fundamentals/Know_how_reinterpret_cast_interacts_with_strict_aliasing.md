# Know how reinterpret_cast interacts with strict aliasing

**Category:** Core Language Fundamentals  
**Standard:** C++11, C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/reinterpret_cast>  

---

## Topic Overview

### What Is Strict Aliasing

The **strict aliasing rule** says: if you access an object through a pointer or reference of a *different* type than the object actually is, that's undefined behavior - with a short list of exceptions. It sounds harsh, but there's a payoff. By promising the compiler that pointers of unrelated types never touch the same memory, you let it generate much faster code (it can keep values in registers instead of reloading them).

### The Rule (Simplified)

An object of type `T` is only legal to access through one of these:

1. A pointer/reference of type `T` (cv-qualifiers don't matter).
2. A pointer/reference to a **signed/unsigned variant** of `T`.
3. A pointer/reference to `char`, `unsigned char`, or `std::byte` (C++17).
4. A pointer/reference to a **base class** of `T` (for class types).
5. An aggregate or union containing one of the above.

Anything outside this list is **UB**. Keep item 3 in mind - the "byte" types are your escape hatch, and we'll come back to them.

### Why Does reinterpret_cast Matter

`reinterpret_cast` is the tool that converts between unrelated pointer types. The catch is that the *cast itself* is perfectly legal - the trouble only starts when you **dereference** the result:

```cpp
float f = 3.14f;
int* ip = reinterpret_cast<int*>(&f);
int bits = *ip;  // UB! Accessing float through int* violates strict aliasing
```

So the cast compiles and even looks reasonable. The UB is hiding on the `*ip` line, which is exactly what makes this bug so sneaky.

### How the Compiler Exploits Strict Aliasing

To really understand *why* it's UB, look at what the optimizer is allowed to assume:

```cpp
void optimize_example(int* ip, float* fp) {
    *ip = 42;
    *fp = 3.14f;
    // The compiler ASSUMES ip and fp don't alias (different types)
    // So it can reorder the stores or cache *ip in a register
    std::cout << *ip;  // Compiler may output 42 even if ip == (int*)fp
}
```

Because `int*` and `float*` are unrelated, the compiler concludes the second store can't possibly affect `*ip`, and emits something like:

```asm
mov [rdi], 42        ; store to ip
movss [rsi], 3.14    ; store to fp
mov eax, 42          ; loads 42 directly (cached, skips memory read!)
```

Without strict aliasing, it would have to pessimistically reload from memory every time, just in case those pointers overlapped. The rule is what *buys* the speed - and your type-punning is the casualty.

### The Correct Way: `std::memcpy` or `std::bit_cast`

If you genuinely need to reinterpret bytes, do it through a route the standard blesses. `std::memcpy` copies bytes without any aliasing claim; `std::bit_cast` (C++20) is the same idea with a nicer face:

```cpp
#include <cstring>
#include <bit>       // C++20
#include <iostream>

float int_bits_to_float_safe(int i) {
    float f;
    std::memcpy(&f, &i, sizeof(f));  // SAFE: no aliasing violation
    return f;
}

// Even better in C++20:
float int_bits_to_float_modern(int i) {
    return std::bit_cast<float>(i);  // SAFE, constexpr, zero-overhead
}
```

Don't let the `memcpy` scare you on performance grounds: every major compiler recognizes this idiom and compiles it down to exactly the same instructions as the (illegal) `reinterpret_cast` dereference - just without the UB.

### The `char*` / `unsigned char*` / `std::byte*` Exception

Here's the escape hatch from item 3. These byte-pointer types are allowed to alias **any** object. That's the legal foundation under serialization, hashing, and network code that pokes at raw bytes:

```cpp
#include <cstddef>
#include <iostream>

void print_bytes(const void* ptr, std::size_t size) {
    // Legal: unsigned char* can alias anything
    const unsigned char* bytes = static_cast<const unsigned char*>(ptr);
    for (std::size_t i = 0; i < size; ++i) {
        std::printf("%02x ", bytes[i]);
    }
    std::puts("");
}

int main() {
    float f = 3.14f;
    print_bytes(&f, sizeof(f));  // Legal! Prints raw bytes of float

    int arr[] = {1, 2, 3};
    print_bytes(arr, sizeof(arr));  // Legal!

    // C++17 std::byte version:
    const std::byte* bp = reinterpret_cast<const std::byte*>(&f);
    // Legal to read through std::byte*
}
```

One subtlety that trips people up: the exception only runs **one way**. A `char*` may alias a `float`, but a `float*` may *not* alias a `char[]`. The byte types are the universal observers, not a free pass in both directions.

### `-fno-strict-aliasing`

GCC and Clang let you switch the whole optimization off with a flag:

```bash
g++ -O2 -fno-strict-aliasing code.cpp
```

This is a pragmatic tool, not a blessing. It's reasonable when:

- You're porting legacy C code that type-puns through unions.
- You're talking to C libraries that cast through `void*` everywhere.
- The performance you'd lose genuinely doesn't matter for your project (some game engines, the Linux kernel).

But you should *not* lean on it when:

- You're writing new modern C++ - reach for `std::bit_cast` or `std::memcpy` instead.
- You're writing library code that has to be portable across compilers.

And note: MSVC doesn't exploit strict aliasing by default, so it behaves as though `-fno-strict-aliasing` is always on. That's exactly why aliasing bugs can hide for months and then surface only on GCC/Clang at `-O2`.

---

## Self-Assessment

### Q1: Show that accessing a float via an `int*` obtained by `reinterpret_cast` is UB under strict aliasing

This example puts the broken version next to the two correct versions, so you can see they agree *when the UB happens to behave* - which is precisely what makes it dangerous to rely on:

```cpp
#include <iostream>
#include <cstring>
#include <bit>

int main() {
    float f = 3.14f;

    // ==== UB: strict aliasing violation ====
    int* bad_ptr = reinterpret_cast<int*>(&f);
    int bad_bits = *bad_ptr;  // UB! Reading float object through int*
    // The compiler may optimize away this read or return stale data

    std::cout << "UB result: " << bad_bits << "\n";  // May print anything

    // ==== CORRECT: use std::memcpy ====
    int good_bits;
    std::memcpy(&good_bits, &f, sizeof(good_bits));
    std::cout << "memcpy result: " << good_bits << "\n";  // 1078523331 (IEEE 754 bits of 3.14f)

    // ==== BEST (C++20): use std::bit_cast ====
    int best_bits = std::bit_cast<int>(f);
    std::cout << "bit_cast result: " << best_bits << "\n"; // 1078523331

    // Verify they agree (when UB happens to "work"):
    std::cout << "Match: " << (good_bits == best_bits) << "\n"; // 1
}
```

**How this works:**

- `reinterpret_cast<int*>(&f)` converts a `float*` to `int*` - the cast is fine, the dereference is UB.
- Since the compiler is entitled to assume `int*` and `float*` never overlap, it may:
  - hand you a stale cached value,
  - reorder reads and writes around the aliasing access, or
  - delete the read entirely.
- `std::memcpy` copies bytes with no type claim attached - always safe.
- `std::bit_cast<int>(f)` (C++20) is the cleanest answer: safe, `constexpr`, and zero overhead.

### Q2: List the exceptions - `char*`, `unsigned char*`, and `std::byte*` can alias anything

These pointer types are the sanctioned way to read raw memory - accessing any object through them is always legal:

| Pointer type | Available since | Notes |
| --- | --- | --- |
| `char*` | C++98 | Can alias any type; used in legacy code |
| `unsigned char*` | C++98 | Preferred for raw byte access (no signedness issues) |
| `std::byte*` | C++17 | Strongest type - can only be manipulated via bitwise ops |
| `signed char*` | C++98 | Also aliases anything (signed/unsigned variant rule) |

```cpp
#include <cstddef>
#include <cstdint>
#include <iostream>

struct Packet {
    uint32_t header;
    uint16_t length;
    uint8_t  payload[64];
};

void inspect_memory(const Packet& pkt) {
    // ALL THREE of these are legal:

    // 1. char*
    const char* cp = reinterpret_cast<const char*>(&pkt);
    std::cout << "First byte (char): " << static_cast<int>(cp[0]) << "\n";

    // 2. unsigned char*
    const unsigned char* ucp = reinterpret_cast<const unsigned char*>(&pkt);
    std::cout << "First byte (uchar): " << static_cast<int>(ucp[0]) << "\n";

    // 3. std::byte* (C++17)
    const std::byte* bp = reinterpret_cast<const std::byte*>(&pkt);
    std::cout << "First byte (byte): " << std::to_integer<int>(bp[0]) << "\n";
}

int main() {
    Packet pkt{0x12345678, 10, {}};
    inspect_memory(pkt);  // Legal - no UB
    return 0;
}
```

**Why these exceptions exist:**

- Low-level work - serialization, hashing, networking - fundamentally needs to read raw memory.
- `memcpy`, `memset`, and friends operate through `void*`/`char*` internally, so the language has to permit it.
- `std::byte` (C++17) adds a layer of safety: it only supports bitwise operations, so you can't accidentally do arithmetic on a byte pointer the way you can with `char`.

### Q3: Demonstrate that `-fno-strict-aliasing` disables this optimization and when that is acceptable

The classic case is union type-punning - legal in C99, but UB in C++ under strict aliasing. With the flag off, compilers treat it as defined:

```cpp
#include <iostream>

// This function demonstrates the optimization opportunity
int check_aliasing(int* ip, float* fp) {
    *ip = 42;
    *fp = 3.14f;
    return *ip;  // With strict aliasing: compiler returns 42 (cached)
                  // Without: compiler reloads from memory
}

int main() {
    // Type-punning through union (common in C, UB in C++ strict aliasing)
    union Pun {
        float f;
        int i;
    };

    Pun p;
    p.f = 3.14f;

    // Reading p.i after writing p.f is UB in C++ (legal in C99)
    // With -fno-strict-aliasing, compilers treat this as defined behavior
    std::cout << "Union type pun: 0x" << std::hex << p.i << std::dec << "\n";

    // Compile and test:
    // g++ -O2 -fstrict-aliasing code.cpp -o strict     # May miscompile
    // g++ -O2 -fno-strict-aliasing code.cpp -o nostrict # Always "works"

    // Check what check_aliasing returns when pointers alias:
    Pun pun;
    int result = check_aliasing(&pun.i, &pun.f);
    std::cout << "check_aliasing result: " << result << "\n";
    // With strict aliasing: likely prints 42 (compiler caches)
    // Without: prints the int representation of 3.14f

    return 0;
}
```

When the flag is a **reasonable** call:

- Porting legacy C that relies on union type-punning.
- Interfacing with hardware/OS APIs that cast through `void*` constantly.
- Game engines or multimedia code where the lost optimization is negligible.
- Projects already committed to it (the Linux kernel builds with `-fno-strict-aliasing`).

When it's the **wrong** call:

- New C++ library code - use `std::bit_cast` or `std::memcpy`.
- Performance-critical numeric code, where you may be handing back real speed.
- Anything that must also build on MSVC, which has no equivalent flag.

---

## Additional Examples

### Safe Type Punning with `std::bit_cast` (C++20)

The big win of `std::bit_cast` over `memcpy` is that it's `constexpr` - you can inspect IEEE 754 bit patterns at compile time:

```cpp
#include <bit>
#include <cstdint>
#include <iostream>
#include <iomanip>

int main() {
    // Float to int bits (IEEE 754 inspection)
    float f = -0.0f;
    uint32_t bits = std::bit_cast<uint32_t>(f);
    std::cout << "Bits of -0.0f: 0x" << std::hex << bits << "\n";  // 0x80000000

    // Double to uint64_t
    double d = 1.0;
    uint64_t dbits = std::bit_cast<uint64_t>(d);
    std::cout << "Bits of 1.0: 0x" << dbits << std::dec << "\n";   // 0x3ff0000000000000

    // Constexpr! (unlike memcpy)
    constexpr uint32_t pi_bits = std::bit_cast<uint32_t>(3.14159f);
    static_assert(pi_bits == 0x40490fd0 || pi_bits == 0x40490fdb);  // IEEE 754
}
```

### Network Byte Order Conversion (Legal Aliasing)

Reading a multi-byte field out of a buffer is the everyday version of all this - and the safe form is just a `memcpy` away:

```cpp
#include <cstring>
#include <cstdint>
#include <iostream>

// Safe: uses memcpy, not reinterpret_cast dereference
uint32_t read_uint32_be(const unsigned char* buf) {
    uint32_t result;
    std::memcpy(&result, buf, sizeof(result));
    // Then convert from big-endian if needed
    return result;
}

int main() {
    unsigned char packet[] = {0x00, 0x00, 0x00, 0x42};
    uint32_t val = read_uint32_be(packet);
    std::cout << "Value: " << val << "\n";
}
```

---

## Notes

- **Never** dereference a `reinterpret_cast`-ed pointer of an unrelated type - use `std::bit_cast` or `std::memcpy`.
- `char*`, `unsigned char*`, and `std::byte*` may legally alias any type - but not the reverse.
- `-fno-strict-aliasing` is a compiler flag, not a language guarantee - code that depends on it isn't portable.
- `std::bit_cast` (C++20) is the modern, `constexpr`, zero-overhead way to type-pun.
- MSVC doesn't optimize on strict aliasing, so these bugs can stay hidden until you build with GCC/Clang at `-O2`.
