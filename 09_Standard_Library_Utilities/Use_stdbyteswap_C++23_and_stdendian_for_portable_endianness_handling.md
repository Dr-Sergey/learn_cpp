# Use std::byteswap (C++23) and std::endian for portable endianness handling

**Category:** Standard Library Utilities  
**Standard:** C++20, C++23 (endian)  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/byteswap>  

---

## Topic Overview

Network protocols (TCP/IP) use **big-endian** byte order, while most modern CPUs (x86, ARM in default mode) are **little-endian**. C++20 added `std::endian` to detect the native byte order at compile time, and C++23 added `std::byteswap` to reverse byte order portably.

Before these additions, every codebase that dealt with network data had to use platform-specific functions (`htonl`, `ntohl` on POSIX; `_byteswap_ulong` on MSVC) or roll their own byte-reversal. Neither approach was `constexpr`, and neither worked transparently across platforms. These two C++ additions give you a clean, portable, compiler-optimized solution.

### std::endian (C++20, `<bit>`)

```cpp
enum class endian {
    little = /* implementation-defined */,
    big    = /* implementation-defined */,
    native = /* little or big, depending on platform */
};
```

| Platform | `std::endian::native` equals |
| --- | --- |
| x86, x86_64, ARM (default) | `std::endian::little` |
| Network byte order, PowerPC (default) | `std::endian::big` |
| Mixed-endian (PDP-11) | Neither little nor big |

### std::byteswap (C++23, `<bit>`)

```cpp
template<class T>
constexpr T byteswap(T value) noexcept;
```

- Reverses the bytes of `value`.
- `T` must be an integral type with no padding bits.
- `constexpr` - works at compile time.
- Compilers emit a single `BSWAP` instruction on x86.

### Core Syntax

Here is the basic effect: `0x01020304` becomes `0x04030201`, each byte moving to the mirror position.

```cpp
#include <bit>
#include <cstdint>
#include <iostream>
#include <iomanip>

int main() {
    uint32_t x = 0x01020304;

    std::cout << std::hex << std::setfill('0');
    std::cout << "original:  0x" << std::setw(8) << x << "\n";
    std::cout << "byteswap:  0x" << std::setw(8) << std::byteswap(x) << "\n";
    // Output:
    // original:  0x01020304
    // byteswap:  0x04030201

    // Detect endianness at compile time
    if constexpr (std::endian::native == std::endian::little) {
        std::cout << "This platform is little-endian\n";
    } else if constexpr (std::endian::native == std::endian::big) {
        std::cout << "This platform is big-endian\n";
    } else {
        std::cout << "This platform is mixed-endian\n";
    }
}
```

---

## Self-Assessment

### Q1: Use std::byteswap<uint32_t>(x) to convert a little-endian integer to big-endian

**Answer:**

`byteswap` works on all integral sizes - `uint16_t`, `uint32_t`, `uint64_t` all just work. The `static_assert` at the end confirms you can rely on this at compile time too.

```cpp
#include <bit>
#include <cstdint>
#include <iostream>
#include <iomanip>

int main() {
    // On a little-endian machine, memory stores LSB first:
    // value 0x01020304 is stored as bytes: 04 03 02 01
    // Big-endian representation is: 01 02 03 04

    uint32_t le_value = 0xAABBCCDD;

    // byteswap reverses the byte order
    uint32_t be_value = std::byteswap(le_value);

    std::cout << std::hex << std::setfill('0');
    std::cout << "little-endian: 0x" << std::setw(8) << le_value << "\n";
    std::cout << "big-endian:    0x" << std::setw(8) << be_value << "\n";
    // Output:
    // little-endian: 0xaabbccdd
    // big-endian:    0xddccbbaa

    // Works with all integral sizes
    uint16_t s = 0x1234;
    uint64_t l = 0x0102030405060708ULL;
    std::cout << "swap16: 0x" << std::setw(4) << std::byteswap(s) << "\n";
    std::cout << "swap64: 0x" << std::setw(16) << std::byteswap(l) << "\n";
    // Output:
    // swap16: 0x3412
    // swap64: 0x0807060504030201

    // constexpr usage
    static_assert(std::byteswap(uint32_t{0x01020304}) == 0x04030201);
}
```

---

### Q2: Use std::endian::native == std::endian::little to conditionally compile byte-swap code

**Answer:**

The key insight here is that `if constexpr` lets the compiler eliminate the dead branch entirely - on a big-endian platform, `host_to_big` compiles to a no-op with zero instructions. On little-endian it compiles to a single `BSWAP`. That is the ideal outcome.

```cpp
#include <bit>
#include <cstdint>
#include <iostream>

// Convert host byte order to big-endian (network order)
template<typename T>
constexpr T host_to_big(T value) {
    if constexpr (std::endian::native == std::endian::big) {
        return value;  // already big-endian, no swap needed
    } else if constexpr (std::endian::native == std::endian::little) {
        return std::byteswap(value);  // swap for little-endian
    } else {
        static_assert(false, "Mixed-endian not supported");
    }
}

// Convert big-endian to host byte order
template<typename T>
constexpr T big_to_host(T value) {
    return host_to_big(value);  // swap is its own inverse
}

int main() {
    uint32_t host_val = 0x01020304;
    uint32_t net_val  = host_to_big(host_val);
    uint32_t back     = big_to_host(net_val);

    std::cout << std::hex;
    std::cout << "host:    0x" << host_val << "\n";
    std::cout << "network: 0x" << net_val  << "\n";
    std::cout << "back:    0x" << back     << "\n";
    // On little-endian:
    // host:    0x1020304
    // network: 0x4030201
    // back:    0x1020304

    // Compile-time: the if constexpr eliminates the dead branch entirely
    // On a big-endian platform, host_to_big compiles to a no-op
    // On a little-endian platform, it compiles to a single BSWAP instruction

    // static_assert works because host_to_big is constexpr
    static_assert(big_to_host(host_to_big(uint16_t{0x1234})) == 0x1234);
}
```

**Why `if constexpr` instead of runtime `if`:**

- The compiler eliminates the unused branch entirely - zero runtime cost.
- On big-endian platforms, `host_to_big` compiles to nothing.
- On little-endian platforms, it compiles to a single `BSWAP` instruction.

---

### Q3: Write a portable ntohl() equivalent using std::endian and std::byteswap

**Answer:**

The POSIX functions `ntohl`/`htonl` are not `constexpr`, are not templated, and require platform-specific headers. Here is a portable, `constexpr`, type-safe replacement. The single-byte specialization in the generic version is a nice touch - swapping a `uint8_t` is always a no-op.

```cpp
#include <bit>
#include <cstdint>
#include <iostream>
#include <array>
#include <cstring>

// Network byte order is always big-endian

// ntohl: network-to-host (32-bit)
constexpr uint32_t my_ntohl(uint32_t net) {
    if constexpr (std::endian::native == std::endian::big)
        return net;
    else
        return std::byteswap(net);
}

// htonl: host-to-network (32-bit)
constexpr uint32_t my_htonl(uint32_t host) {
    return my_ntohl(host);  // symmetric
}

// ntohs: network-to-host (16-bit)
constexpr uint16_t my_ntohs(uint16_t net) {
    if constexpr (std::endian::native == std::endian::big)
        return net;
    else
        return std::byteswap(net);
}

// htons: host-to-network (16-bit)
constexpr uint16_t my_htons(uint16_t host) {
    return my_ntohs(host);
}

// Generic version for any integral type
template<std::integral T>
constexpr T ntoh(T net) {
    if constexpr (std::endian::native == std::endian::big)
        return net;
    else if constexpr (sizeof(T) == 1)
        return net;  // single byte, no swap needed
    else
        return std::byteswap(net);
}

int main() {
    // Simulate reading a network packet header
    // IP header contains a 16-bit total length and 32-bit source IP
    // These are stored in big-endian (network byte order)

    // Simulated raw packet bytes (big-endian)
    uint16_t raw_length = 0x0050;       // 80 bytes in big-endian
    uint32_t raw_ip = 0xC0A80001;       // 192.168.0.1 in big-endian

    uint16_t length = my_ntohs(raw_length);
    uint32_t ip = my_ntohl(raw_ip);

    // Correct simulation: if the value IS 80, the big-endian bytes are 0x0050
    // On a little-endian host, reading these bytes directly as uint16_t gives 0x5000
    // ntohs(0x5000) = byteswap(0x5000) = 0x0050 = 80

    uint16_t as_read_on_le = 0x5000;  // how LE host reads big-endian 0x0050
    std::cout << "ntohs(0x5000) = " << my_ntohs(as_read_on_le) << "\n";
    // Output: ntohs(0x5000) = 80

    // constexpr proof
    static_assert(my_ntohl(my_htonl(0x12345678)) == 0x12345678);
    static_assert(my_ntohs(my_htons(uint16_t{1234})) == 1234);

    // Generic version
    uint64_t big = ntoh(uint64_t{0x0102030405060708});
    std::cout << std::hex << "ntoh64: 0x" << big << "\n";
}
```

**Advantages over POSIX ntohl():**

- **`constexpr`** - usable at compile time.
- **Type-safe** - templates prevent mixing 16-bit and 32-bit accidentally.
- **No platform headers** - no need for `<arpa/inet.h>` (POSIX) or `<winsock2.h>` (Windows).
- **Portable** - works correctly on both little-endian and big-endian platforms.

---

## Notes

- `std::endian` is in `<bit>` (C++20). `std::byteswap` is in `<bit>` (C++23).
- On older compilers, use `__builtin_bswap32` (GCC/Clang) or `_byteswap_ulong` (MSVC) as fallbacks.
- `std::byteswap` on a single byte (`uint8_t`) returns the value unchanged.
- For floating-point byteswap, use `bit_cast` to integer, swap, then `bit_cast` back.
- All functions are `constexpr` and `noexcept`.
