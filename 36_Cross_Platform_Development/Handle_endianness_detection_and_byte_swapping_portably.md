# Handle Endianness Detection and Byte Swapping Portably

**Category:** Cross Platform Development  
**Standard:** C++20, C++23  
**Reference:** <https://en.cppreference.com/w/cpp/types/endian>  

---

## Topic Overview

Endianness determines how multi-byte scalar values are stored in memory. x86/x64 and ARM (in its default mode) are little-endian; many network protocols and file formats (TIFF, Java class files, big-endian ELF) use big-endian (network byte order). Portable serialization, binary protocols, and memory-mapped I/O all require explicit handling.

C++20 introduced `std::endian` in `<bit>`, giving a compile-time constant for native byte order. C++23 added `std::byteswap` for unconditional reversal of bytes in any integral type. Before these, developers relied on compiler intrinsics (`__builtin_bswap*`, `_byteswap_*`) or manual bit manipulation.

| Facility | Standard | Header | Notes |
| --- | --- | --- | --- |
| `std::endian::native` | C++20 | `<bit>` | Compile-time detection |
| `std::byteswap(x)` | C++23 | `<bit>` | Reverses bytes of integral `x` |
| `__builtin_bswap16/32/64` | GCC/Clang ext. | - | Intrinsic, compiles to single instr. |
| `_byteswap_ushort/ulong/uint64` | MSVC ext. | `<stdlib.h>` | Intrinsic |
| `htonl / ntohl` | POSIX / Winsock | `<arpa/inet.h>` / `<winsock2.h>` | 32-bit network order only |

The memory layout diagram makes the concept concrete. For the value `0x01020304`:

```cpp
Memory layout of 0x01020304:

Big-endian:     [01] [02] [03] [04]   (MSB first - "network byte order")
Little-endian:  [04] [03] [02] [01]   (LSB first - x86 native)
Mixed-endian:   [02] [01] [04] [03]   (rare, some legacy ARM)
```

The key design principle: **convert to a canonical byte order at serialization boundaries, use native order internally**. This avoids scattered byte-swap calls and keeps hot loops fast. The conversion only happens when data crosses a process or network boundary.

---

## Self-Assessment

### Q1: Write a portable byte_swap function that uses `std::byteswap` on C++23, compiler intrinsics on older compilers, and a manual fallback

The reason this needs layers is that `std::byteswap` only arrived in C++23, GCC/Clang have intrinsics that compile to a single `bswap` instruction, and MSVC has its own set of intrinsics. The manual fallback is there for truly unusual toolchains. Thanks to `if constexpr`, the compiler picks exactly one path and the others are discarded entirely:

```cpp
#include <cstdint>
#include <cstring>
#include <iostream>
#include <type_traits>
#include <array>

#if __cplusplus >= 202302L || (defined(__cpp_lib_byteswap) && __cpp_lib_byteswap >= 202110L)
    #include <bit>
    #define HAS_STD_BYTESWAP 1
#else
    #define HAS_STD_BYTESWAP 0
#endif

template <typename T>
    requires std::is_integral_v<T>
constexpr T portable_byteswap(T value) noexcept {
#if HAS_STD_BYTESWAP
    return std::byteswap(value);
#elif defined(__GNUC__) || defined(__clang__)
    if constexpr (sizeof(T) == 1) return value;
    else if constexpr (sizeof(T) == 2)
        return static_cast<T>(__builtin_bswap16(static_cast<std::uint16_t>(value)));
    else if constexpr (sizeof(T) == 4)
        return static_cast<T>(__builtin_bswap32(static_cast<std::uint32_t>(value)));
    else if constexpr (sizeof(T) == 8)
        return static_cast<T>(__builtin_bswap64(static_cast<std::uint64_t>(value)));
#elif defined(_MSC_VER)
    if constexpr (sizeof(T) == 1) return value;
    else if constexpr (sizeof(T) == 2)
        return static_cast<T>(_byteswap_ushort(static_cast<unsigned short>(value)));
    else if constexpr (sizeof(T) == 4)
        return static_cast<T>(_byteswap_ulong(static_cast<unsigned long>(value)));
    else if constexpr (sizeof(T) == 8)
        return static_cast<T>(_byteswap_uint64(static_cast<unsigned __int64>(value)));
#else
    // Manual fallback - works at compile time
    if constexpr (sizeof(T) == 1) return value;
    else {
        auto u = static_cast<std::make_unsigned_t<T>>(value);
        std::make_unsigned_t<T> result = 0;
        for (std::size_t i = 0; i < sizeof(T); ++i) {
            result = (result << 8) | (u & 0xFF);
            u >>= 8;
        }
        return static_cast<T>(result);
    }
#endif
}

int main() {
    constexpr std::uint32_t val = 0x01020304;
    constexpr auto swapped = portable_byteswap(val);
    static_assert(swapped == 0x04030201);

    std::cout << std::hex
              << "Original: 0x" << val << "\n"
              << "Swapped:  0x" << swapped << "\n";
    return 0;
}
```

The `static_assert` is important - it verifies the swap at compile time, so if your manual fallback path has a bug, you find out immediately rather than at runtime on the target architecture.

### Q2: Implement a `NetworkSerializer` that writes integers in big-endian (network byte order) regardless of host endianness, using `std::endian` for detection

The design principle here is elegant: `to_big_endian` uses `std::endian::native` as a compile-time constant in `if constexpr`. On a big-endian host, the value passes through unchanged. On a little-endian host (x86, most ARM), the bytes are swapped. The serializer then writes the corrected bytes to its buffer, and the same `from_big_endian` call inverts the swap on read. Because byte-swap is its own inverse, the same function works in both directions:

```cpp
#include <bit>
#include <cstdint>
#include <cstring>
#include <vector>
#include <iostream>
#include <iomanip>
#include <concepts>
#include <span>

template <typename T>
    requires std::is_integral_v<T>
constexpr T to_big_endian(T value) noexcept {
    if constexpr (std::endian::native == std::endian::big) {
        return value;
    } else {
        // Use std::byteswap if available, else manual swap
        #if __cpp_lib_byteswap >= 202110L
        return std::byteswap(value);
        #else
        auto u = static_cast<std::make_unsigned_t<T>>(value);
        std::make_unsigned_t<T> result = 0;
        for (std::size_t i = 0; i < sizeof(T); ++i) {
            result = (result << 8) | (u & 0xFF);
            u >>= 8;
        }
        return static_cast<T>(result);
        #endif
    }
}

template <typename T>
    requires std::is_integral_v<T>
constexpr T from_big_endian(T value) noexcept {
    return to_big_endian(value); // Byte-swap is its own inverse
}

class NetworkSerializer {
    std::vector<std::uint8_t> buffer_;
public:
    template <std::integral T>
    void write(T value) {
        T be = to_big_endian(value);
        const auto* bytes = reinterpret_cast<const std::uint8_t*>(&be);
        buffer_.insert(buffer_.end(), bytes, bytes + sizeof(T));
    }

    template <std::integral T>
    T read(std::size_t offset) const {
        T be{};
        std::memcpy(&be, buffer_.data() + offset, sizeof(T));
        return from_big_endian(be);
    }

    std::span<const std::uint8_t> data() const { return buffer_; }
    std::size_t size() const { return buffer_.size(); }

    void dump() const {
        for (auto b : buffer_)
            std::cout << std::hex << std::setfill('0') << std::setw(2)
                      << static_cast<int>(b) << ' ';
        std::cout << "\n";
    }
};

int main() {
    NetworkSerializer ser;
    ser.write<std::uint32_t>(0x01020304);
    ser.write<std::uint16_t>(0xCAFE);

    std::cout << "Wire bytes: ";
    ser.dump();  // Always prints: 01 02 03 04 ca fe

    auto val32 = ser.read<std::uint32_t>(0);
    auto val16 = ser.read<std::uint16_t>(4);
    std::cout << std::hex
              << "Read back: 0x" << val32 << ", 0x" << val16 << "\n";
    return 0;
}
```

The wire bytes are always `01 02 03 04 ca fe` regardless of the host architecture. That is the guarantee you want from a network serializer.

### Q3: Demonstrate compile-time endianness detection to select struct packing strategy and validate with `static_assert`

This example goes a step further: the `HeaderCodec` template encodes and decodes a binary file header in a specified target byte order. The `needs_swap` constant is computed at compile time from `std::endian::native`, so the `maybe_swap` helper either passes the value through or reverses it, with zero runtime cost. The `static_assert` at the end catches mixed-endian platforms early rather than producing silently corrupt file headers:

```cpp
#include <bit>
#include <cstdint>
#include <cstring>
#include <iostream>
#include <array>

// Canonical on-disk format: little-endian (common for x86-centric formats)
struct FileHeader {
    std::uint32_t magic;
    std::uint32_t version;
    std::uint64_t payload_size;
};

template <std::endian TargetOrder = std::endian::little>
class HeaderCodec {
    static constexpr bool needs_swap =
        (TargetOrder != std::endian::native);

    template <std::integral T>
    static constexpr T maybe_swap(T v) noexcept {
        if constexpr (!needs_swap) {
            return v;
        } else {
            auto u = static_cast<std::make_unsigned_t<T>>(v);
            std::make_unsigned_t<T> r = 0;
            for (std::size_t i = 0; i < sizeof(T); ++i) {
                r = (r << 8) | (u & 0xFF);
                u >>= 8;
            }
            return static_cast<T>(r);
        }
    }

public:
    static std::array<std::uint8_t, sizeof(FileHeader)>
    encode(const FileHeader& h) {
        FileHeader wire{
            maybe_swap(h.magic),
            maybe_swap(h.version),
            maybe_swap(h.payload_size)
        };
        std::array<std::uint8_t, sizeof(FileHeader)> buf{};
        std::memcpy(buf.data(), &wire, sizeof(wire));
        return buf;
    }

    static FileHeader decode(const std::uint8_t* data) {
        FileHeader wire;
        std::memcpy(&wire, data, sizeof(wire));
        return FileHeader{
            maybe_swap(wire.magic),
            maybe_swap(wire.version),
            maybe_swap(wire.payload_size)
        };
    }
};

// Compile-time checks
static_assert(std::endian::native == std::endian::little ||
              std::endian::native == std::endian::big,
              "Mixed-endian platforms are not supported");

static_assert(sizeof(FileHeader) == 16 || sizeof(FileHeader) == 24,
              "Unexpected padding - add #pragma pack or alignas");

int main() {
    FileHeader original{0xDEADBEEF, 2, 1024};

    auto bytes = HeaderCodec<std::endian::little>::encode(original);

    // First 4 bytes should be EF BE AD DE on any platform (LE wire format)
    std::cout << "First 4 wire bytes: ";
    for (int i = 0; i < 4; ++i)
        std::printf("%02X ", bytes[i]);
    std::cout << "\n";

    auto decoded = HeaderCodec<std::endian::little>::decode(bytes.data());
    std::cout << std::hex
              << "Magic:   0x" << decoded.magic << "\n"
              << "Version: "   << std::dec << decoded.version << "\n"
              << "Size:    "   << decoded.payload_size << "\n";
    return 0;
}
```

On a little-endian host, `needs_swap` is `false`, so the magic `0xDEADBEEF` lands in the buffer as `EF BE AD DE` without any runtime swap. On a big-endian host, the swap kicks in and produces the same result. Same wire format, both platforms.

---

## Notes

- `std::endian::native` is evaluated at compile time - use it in `if constexpr` and `static_assert` for zero-overhead dispatch.
- `std::byteswap` (C++23) handles signed types correctly by operating on the object representation; it does not promote to unsigned.
- All major compilers optimize manual byte-swap loops into single `bswap`/`rev` instructions at `-O2`; the intrinsic wrappers are mainly for clarity.
- Mixed-endian (middle-endian) platforms are effectively extinct; `std::endian::native` will equal neither `big` nor `little` on such platforms - use `static_assert` to reject them.
- Network protocols use big-endian; most modern file formats (ELF on x86, Protocol Buffers varint) use little-endian or variable-length encoding.
- Never byte-swap floating-point values directly - serialize the bit pattern via `std::bit_cast<uint32_t>(float_val)`, swap the integer, then transmit.
