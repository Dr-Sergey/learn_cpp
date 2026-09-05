# Apply struct packing and padding to minimize memory footprint

**Category:** Performance and CPU Architecture  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/object#Alignment>  

---

## Topic Overview

Compilers insert silent padding bytes between struct members to keep each member aligned to its natural boundary. The tricky part is that this happens automatically and invisibly, so a struct can end up much larger than the sum of its fields.

The rule of thumb is that every member must start at an address that is a multiple of its own `alignof`. So a `double` (8-byte aligned) that follows a single `char` forces the compiler to insert 7 bytes of dead space before it. The struct's total size is then rounded up to a multiple of the largest member's alignment.

Here is a side-by-side picture of a badly ordered and a well ordered struct:

```cpp
Poorly ordered (24 bytes):       Well ordered (16 bytes):

struct Bad {                     struct Good {
  char  a;   // offset 0          double d;  // offset 0  (8)
  // 7 bytes padding               int    i;  // offset 8  (4)
  double d;  // offset 8          short  s;  // offset 12 (2)
  int   i;   // offset 16         char   a;  // offset 14 (1)
  // 4 bytes padding               char   b;  // offset 15 (1)
};           // size = 24         };           // size = 16
```

Simply reordering the fields from largest to smallest cuts 8 bytes - a 33% reduction - without changing anything else about the program. Multiply that by a million elements in an array, and you have saved 8 MB and noticeably improved cache utilization.

| Rule | Description |
| --- | --- |
| Alignment | Each member aligned to its `alignof` value |
| Padding | Compiler inserts bytes to satisfy alignment |
| Struct size | Multiple of largest member's alignment |
| Reorder trick | Sort fields largest-to-smallest |
| `#pragma pack(1)` | Remove padding (may cause misaligned access) |

---

## Self-Assessment

### Q1: Reveal padding with `sizeof` and `offsetof`, then reorder

The standard way to investigate a struct's layout is with `sizeof` and `offsetof`. `sizeof` tells you the total size including padding, and `offsetof` shows you exactly where each field lands - so you can spot the gaps.

```cpp
#include <iostream>
#include <cstddef>

// BAD layout: lots of padding
struct Padded {
    char  a;     // 1 byte + 7 padding
    double d;    // 8 bytes
    char  b;     // 1 byte + 3 padding
    int   i;     // 4 bytes
};               // Total: 24 bytes (8 bytes wasted!)

// GOOD layout: minimal padding
struct Compact {
    double d;    // 8 bytes (offset 0)
    int    i;    // 4 bytes (offset 8)
    char   a;    // 1 byte  (offset 12)
    char   b;    // 1 byte  (offset 13)
};               // Total: 16 bytes (0 bytes wasted)

int main() {
    // Reveal padding in Padded:
    std::cout << "=== Padded (bad order) ===\n";
    std::cout << "sizeof:    " << sizeof(Padded) << '\n';           // 24
    std::cout << "a offset:  " << offsetof(Padded, a) << '\n';     // 0
    std::cout << "d offset:  " << offsetof(Padded, d) << '\n';     // 8  (7 padding!)
    std::cout << "b offset:  " << offsetof(Padded, b) << '\n';     // 16
    std::cout << "i offset:  " << offsetof(Padded, i) << '\n';     // 20 (3 padding!)

    std::cout << "\n=== Compact (good order) ===\n";
    std::cout << "sizeof:    " << sizeof(Compact) << '\n';          // 16
    std::cout << "d offset:  " << offsetof(Compact, d) << '\n';    // 0
    std::cout << "i offset:  " << offsetof(Compact, i) << '\n';    // 8
    std::cout << "a offset:  " << offsetof(Compact, a) << '\n';    // 12
    std::cout << "b offset:  " << offsetof(Compact, b) << '\n';    // 13

    // For an array of 1M elements:
    // Padded:  24 MB
    // Compact: 16 MB  (33% less memory, better cache utilization)
}
```

Notice that in `Padded` there is a 7-byte gap between `a` and `d`, and a 3-byte gap between `b` and `i`. Those bytes do nothing useful. In `Compact`, all four fields fit end-to-end because the largest type comes first and the small types pack tightly at the end.

### Q2: `__attribute__((packed))` and alignment fault risk

You can also force a struct to have zero padding with `__attribute__((packed))` or `#pragma pack`. This is tempting because it minimizes size, but it comes with a real cost: members may end up at misaligned addresses, which causes hardware penalties or outright faults on strict-alignment platforms.

```cpp
#include <iostream>
#include <cstddef>
#include <cstdint>

// Force no padding:
struct __attribute__((packed)) PackedStruct {
    char  a;     // offset 0
    double d;    // offset 1  (MISALIGNED!)
    int   i;     // offset 9  (MISALIGNED!)
    char  b;     // offset 13
};               // Total: 14 bytes (vs 24 unpacked)

struct NormalStruct {
    char  a;
    double d;
    int   i;
    char  b;
};

int main() {
    std::cout << "Packed:  " << sizeof(PackedStruct) << '\n';   // 14
    std::cout << "Normal:  " << sizeof(NormalStruct) << '\n';   // 24
    std::cout << "d offset (packed): " << offsetof(PackedStruct, d) << '\n'; // 1

    // RISKS of packing:
    // 1. x86/x64: misaligned access works but is SLOWER
    //    (hardware handles it, ~2x penalty for crossing cache line)
    //
    // 2. ARM/MIPS/RISC-V (strict alignment):
    //    SIGBUS or hardware trap on misaligned access!
    //
    // 3. Cannot take address of packed member safely:
    PackedStruct ps{.a='x', .d=3.14, .i=42, .b='y'};
    // double* ptr = &ps.d;  // WARNING: taking address of packed member
    //                       // ptr may be misaligned!

    // BETTER approach: reorder fields instead of packing
    struct Reordered {
        double d;  // 8
        int    i;  // 4
        char   a;  // 1
        char   b;  // 1 + 2 padding
    };              // 16 bytes, properly aligned

    std::cout << "Reordered: " << sizeof(Reordered) << '\n';  // 16
}
```

The reason this trips people up is that x86 silently handles misaligned reads - you just pay a performance tax - while ARM and RISC-V can signal a bus error. Code written on x86 that relies on packing can silently break when ported to embedded or mobile platforms. Reordering fields is almost always the right answer instead.

### Q3: Packed vs aligned struct cache efficiency in arrays

This benchmark makes the performance difference concrete. The packed struct saves memory, but every array access may straddle a cache-line boundary, costing extra cycles on every read.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <numeric>

struct Aligned {
    double value;  // 8
    int    id;     // 4
    char   flag;   // 1 + 3 padding
};                 // 16 bytes, naturally aligned

struct __attribute__((packed)) Packed {
    double value;  // 8
    int    id;     // 4
    char   flag;   // 1
};                 // 13 bytes, misaligned doubles in array!

int main() {
    constexpr int N = 1'000'000;

    // Arrays:
    std::vector<Aligned> aligned(N);
    std::vector<Packed> packed(N);

    for (int i = 0; i < N; ++i) {
        aligned[i] = {static_cast<double>(i), i, 'a'};
        packed[i]  = {static_cast<double>(i), i, 'a'};
    }

    // Benchmark: sum all values
    auto bench = [](auto& vec, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        double sum = 0;
        for (int rep = 0; rep < 100; ++rep) {
            for (auto& e : vec) sum += e.value;
        }
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0);
        std::cout << label << ": " << us.count() << "us (sum=" << sum << ")\n";
    };

    bench(aligned, "Aligned (16B)");
    bench(packed,  "Packed  (13B)");

    // Typical result on x86:
    // Aligned (16B): ~150ms  -- doubles always cache-line aligned
    // Packed  (13B): ~180ms  -- some doubles cross cache lines
    //
    // Memory: aligned=16MB, packed=13MB (19% less)
    // Speed: aligned ~20% faster (no misalignment penalty)
    //
    // Verdict: reorder fields > pack. Only pack for wire protocols.
}
```

The verdict in the comment is worth repeating: reordering fields beats packing in almost every real-world scenario. Packing belongs in network protocols and file formats where you need an exact on-the-wire layout that must match a spec - not in data structures you access in a hot loop.

---

## Notes

- **Rule of thumb:** sort fields from largest `alignof` to smallest.
- `#pragma pack(push, 1)` / `#pragma pack(pop)` for MSVC compatibility.
- Use `static_assert(sizeof(T) == expected)` to catch layout changes.
- `-Wpadded` (GCC/Clang) warns about implicit padding.
- Packing is appropriate for network protocols and file formats where layout must be exact.
- For arrays, the savings compound: 8 bytes x 1M elements = 8MB less memory.
