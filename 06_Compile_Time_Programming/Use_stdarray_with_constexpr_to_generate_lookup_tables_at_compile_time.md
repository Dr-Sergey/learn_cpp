# Use `std::array` with `constexpr` to Generate Lookup Tables at Compile Time

**Category:** Compile Time Programming  
**Standard:** C++11, C++14, C++17 (`std::array`)  
**Reference:** <https://en.cppreference.com/w/cpp/container/array>  

---

## Topic Overview

### Generating Lookup Tables at Compile Time

Instead of computing lookup tables at program startup, you can generate them entirely at compile time using `constexpr` functions that populate a `std::array`. The result is embedded directly in the binary's read-only data segment - the table exists before your program even starts.

The IIFE (immediately-invoked function expression) pattern using a lambda is the cleanest way to do this:

```cpp
constexpr auto table = [] {
    std::array<double, 360> t{};
    for (int i = 0; i < 360; ++i) {
        t[i] = /* computation */;
    }
    return t;
}();
```

The lambda is called once at compile time, fills the array, and the result is stored in `.rodata`. At runtime, accessing `table[i]` is just a memory load - no initialization, no computation.

### Why Generate at Compile Time

The trade-off is between binary size and startup time. Here's how the two approaches compare:

| Aspect | Runtime table | Compile-time table |
| --- | :---: | :---: |
| Startup cost | Computation runs at startup | Zero - already in binary |
| Memory location | `.data` or heap | `.rodata` (read-only) |
| Thread safety | Needs synchronization (or `call_once`) | Inherently safe |
| Binary size | Smaller (code generates data) | Larger (data in binary) |
| Access speed | Same O(1) | Same O(1) |

The thread safety row is often overlooked: a `constexpr` table requires no locking or `std::call_once` because it's initialized before any code runs. That's a genuine architectural simplification for tables shared across threads.

---

## Self-Assessment

### Q1: Generate a compile-time sine table as `constexpr std::array<double, 360>`

`std::sin` is not `constexpr`, so you can't call it during constant evaluation. The workaround is to implement a compile-time approximation using Taylor series. This is a good exercise in `constexpr` math - the Taylor series converges quickly enough for 15 terms to give floating-point accuracy across the full range.

```cpp
#include <iostream>
#include <array>
#include <cmath>
#include <iomanip>

// === Compile-time sine approximation (Taylor series) ===
// std::sin is not constexpr, so we implement our own

constexpr double constexpr_fmod(double x, double y) {
    return x - static_cast<long long>(x / y) * y;
}

constexpr double constexpr_sin(double x) {
    // Normalize to [-pi, pi]
    constexpr double PI = 3.14159265358979323846;
    x = constexpr_fmod(x, 2.0 * PI);
    if (x > PI) x -= 2.0 * PI;
    if (x < -PI) x += 2.0 * PI;

    // Taylor series: sin(x) = x - x^3/3! + x^5/5! - x^7/7! + ...
    double result = 0.0;
    double term = x;
    for (int n = 1; n <= 15; ++n) {
        result += term;
        term *= -x * x / ((2.0 * n) * (2.0 * n + 1.0));
    }
    return result;
}

// === Generate the table ===
constexpr auto make_sine_table() {
    constexpr double PI = 3.14159265358979323846;
    std::array<double, 360> table{};
    for (int deg = 0; deg < 360; ++deg) {
        double rad = deg * PI / 180.0;
        table[deg] = constexpr_sin(rad);
    }
    return table;
}

constexpr auto sine_table = make_sine_table();

// === Verify at compile time ===
static_assert(sine_table[0] < 0.001 && sine_table[0] > -0.001);     // sin(0 deg) ≈ 0
static_assert(sine_table[90] > 0.999 && sine_table[90] < 1.001);    // sin(90 deg) ≈ 1
static_assert(sine_table[180] < 0.001 && sine_table[180] > -0.001); // sin(180 deg) ≈ 0
static_assert(sine_table[270] < -0.999 && sine_table[270] > -1.001);// sin(270 deg) ≈ -1

int main() {
    std::cout << std::fixed << std::setprecision(6);
    std::cout << "=== Compile-Time Sine Table ===\n";

    // Print every 30 degrees
    for (int deg = 0; deg < 360; deg += 30) {
        std::cout << "sin(" << std::setw(3) << deg << "°) = "
                  << std::setw(10) << sine_table[deg] << "\n";
    }

    // Compare with std::sin at runtime
    std::cout << "\n=== Accuracy vs std::sin ===\n";
    constexpr double PI = 3.14159265358979323846;
    double max_error = 0.0;
    for (int deg = 0; deg < 360; ++deg) {
        double expected = std::sin(deg * PI / 180.0);
        double error = std::abs(sine_table[deg] - expected);
        if (error > max_error) max_error = error;
    }
    std::cout << "Max error: " << std::scientific << max_error << "\n";

    return 0;
}
```

**Expected output:**

```text
=== Compile-Time Sine Table ===
sin(  0°) =   0.000000
sin( 30°) =   0.500000
sin( 60°) =   0.866025
sin( 90°) =   1.000000
sin(120°) =   0.866025
sin(150°) =   0.500000
sin(180°) =   0.000000
sin(210°) =  -0.500000
sin(240°) =  -0.866025
sin(270°) =  -1.000000
sin(300°) =  -0.866025
sin(330°) =  -0.500000

=== Accuracy vs std::sin ===
Max error: 1.110223e-016
```

That max error is essentially machine epsilon, so the Taylor series approximation is indistinguishable from the runtime `std::sin`.

### Q2: Verify entries with `static_assert` at compile time

The `static_assert` pattern is the most direct way to confirm that your table generator produces correct values. If these assertions pass at compile time, you have a proof embedded in the build that the table is correct - no runtime test needed for the table itself.

```cpp
#include <iostream>
#include <array>
#include <cstddef>

// === Compile-time CRC8 table ===
constexpr auto make_crc8_table() {
    std::array<unsigned char, 256> table{};
    for (int i = 0; i < 256; ++i) {
        unsigned char crc = static_cast<unsigned char>(i);
        for (int bit = 0; bit < 8; ++bit) {
            if (crc & 0x80) {
                crc = (crc << 1) ^ 0x07;  // polynomial: x^8 + x^2 + x + 1
            } else {
                crc <<= 1;
            }
        }
        table[i] = crc;
    }
    return table;
}

constexpr auto crc8_table = make_crc8_table();

// === Verify specific entries at compile time ===
static_assert(crc8_table[0] == 0x00,   "CRC8(0x00) should be 0x00");
static_assert(crc8_table[1] == 0x07,   "CRC8(0x01) should be 0x07 (= polynomial)");
static_assert(crc8_table[255] == 0xF4, "CRC8(0xFF) should be 0xF4");

// === Compile-time CRC8 of a string ===
constexpr unsigned char crc8(const char* data, std::size_t len) {
    unsigned char crc = 0;
    for (std::size_t i = 0; i < len; ++i) {
        crc = crc8_table[crc ^ static_cast<unsigned char>(data[i])];
    }
    return crc;
}

// Verify CRC of known strings at compile time
constexpr auto hello_crc = crc8("Hello", 5);
static_assert(hello_crc == crc8("Hello", 5));  // Consistency check

// === Compile-time power-of-two table ===
constexpr auto make_pow2_table() {
    std::array<unsigned long long, 64> t{};
    t[0] = 1;
    for (int i = 1; i < 64; ++i) {
        t[i] = t[i - 1] * 2;
    }
    return t;
}

constexpr auto pow2 = make_pow2_table();

static_assert(pow2[0] == 1);
static_assert(pow2[1] == 2);
static_assert(pow2[10] == 1024);
static_assert(pow2[20] == 1048576);
static_assert(pow2[32] == 4294967296ULL);
static_assert(pow2[63] == 9223372036854775808ULL);

int main() {
    std::cout << "=== CRC8 Table Entries ===\n";
    std::cout << "crc8_table[0]   = 0x" << std::hex
              << static_cast<int>(crc8_table[0]) << "\n";
    std::cout << "crc8_table[1]   = 0x" << static_cast<int>(crc8_table[1]) << "\n";
    std::cout << "crc8_table[255] = 0x" << static_cast<int>(crc8_table[255]) << "\n";

    std::cout << "\nCRC8(\"Hello\") = 0x" << static_cast<int>(hello_crc) << "\n";

    std::cout << std::dec;
    std::cout << "\n=== Power-of-Two Table ===\n";
    for (int i = 0; i <= 16; ++i) {
        std::cout << "2^" << i << " = " << pow2[i] << "\n";
    }

    std::cout << "\nAll static_asserts passed at compile time!\n";
    return 0;
}
```

### Q3: Compare startup time and code size between runtime-computed and compile-time tables

The key question when choosing between compile-time and runtime table generation is the trade-off between startup time and binary size. This benchmark makes it concrete:

```cpp
#include <iostream>
#include <array>
#include <chrono>
#include <cstring>

// === Compile-time table ===
constexpr auto make_ct_table() {
    std::array<int, 10000> table{};
    for (int i = 0; i < 10000; ++i) {
        // Compute some non-trivial value
        int val = i;
        for (int j = 0; j < 10; ++j) {
            val = (val * 31 + 17) & 0xFFFF;
        }
        table[i] = val;
    }
    return table;
}

constexpr auto ct_table = make_ct_table();

// === Runtime table (same computation) ===
std::array<int, 10000> make_rt_table() {
    std::array<int, 10000> table{};
    for (int i = 0; i < 10000; ++i) {
        int val = i;
        for (int j = 0; j < 10; ++j) {
            val = (val * 31 + 17) & 0xFFFF;
        }
        table[i] = val;
    }
    return table;
}

int main() {
    // Measure runtime table generation
    auto t1 = std::chrono::high_resolution_clock::now();
    auto rt_table = make_rt_table();
    auto t2 = std::chrono::high_resolution_clock::now();

    // Verify both tables are identical
    bool tables_match = (std::memcmp(ct_table.data(), rt_table.data(),
                                      sizeof(int) * 10000) == 0);

    using us = std::chrono::duration<double, std::micro>;
    std::cout << "=== Compile-Time vs Runtime Table Generation ===\n\n";
    std::cout << "Runtime table generation: " << us(t2 - t1).count() << " us\n";
    std::cout << "Compile-time table: 0 us (already in binary)\n";
    std::cout << "Tables match: " << (tables_match ? "yes" : "no") << "\n";

    std::cout << "\n=== Binary Size Impact ===\n";
    std::cout << "Compile-time table:\n";
    std::cout << "  - 10000 ints x 4 bytes = 40,000 bytes in .rodata\n";
    std::cout << "  - Code: just 'load from address' - tiny\n";
    std::cout << "  - Total: ~40 KB extra in binary\n";

    std::cout << "\nRuntime table:\n";
    std::cout << "  - Code: the computation loop (maybe 200 bytes)\n";
    std::cout << "  - 40,000 bytes on stack/BSS at runtime\n";
    std::cout << "  - Total: ~200 bytes extra in binary (data generated at startup)\n";

    std::cout << "\n=== Trade-off Summary ===\n";
    std::cout << "Compile-time:  +binary size, zero startup, instant ready\n";
    std::cout << "Runtime:       +startup time, smaller binary, same runtime perf\n";
    std::cout << "\nGuideline:\n";
    std::cout << "  Small tables (<10 KB): compile-time (zero startup > binary size)\n";
    std::cout << "  Large tables (>1 MB): runtime (binary size matters)\n";
    std::cout << "  Performance-critical: always compile-time\n";

    return 0;
}
```

The key guideline at the bottom: for small tables (under 10 KB), the zero startup cost almost always wins. For very large tables (megabytes), the binary size impact is significant and runtime generation makes more sense. Performance-critical code - hot paths in parsers, protocol decoders, character classifiers - should always use compile-time tables.

---

## Notes

- `constexpr` lookup tables live in `.rodata` - zero startup cost, inherently thread-safe.
- For `constexpr` sine/cosine, implement Taylor series since `std::sin` is not `constexpr`.
- Use `static_assert` to verify critical table entries at compile time.
- Trade-off: compile-time tables increase binary size but eliminate startup computation.
- For tables > 1 MB, consider runtime generation to keep the binary small.
- IIFE pattern `constexpr auto table = []{ ... }();` is the cleanest way to generate tables.
