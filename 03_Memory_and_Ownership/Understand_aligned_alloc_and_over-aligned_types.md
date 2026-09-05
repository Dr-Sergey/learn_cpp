# Understand `aligned_alloc` and Over-Aligned Types

**Category:** Memory and Ownership  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/memory/c/aligned_alloc>  

---

## Topic Overview

### What Is Over-Alignment

Every type has a **natural alignment** (`alignof(T)`). Over-aligned types request alignment **stricter** than the default maximum alignment (`alignof(std::max_align_t)`, typically 16 bytes). You run into over-alignment as soon as you start working with SIMD registers, cache-line isolation, or hardware buffers - all of which have strict address requirements that the default allocator does not satisfy.

Here is what over-aligned types look like in code:

```cpp
struct Normal { double d; };              // alignof = 8 (natural)
struct alignas(32) Wide { float v[8]; };  // alignof = 32 (over-aligned for SIMD)
struct alignas(64) CacheLine { int data[16]; };  // alignof = 64 (one cache line)
```

The `alignas` specifier is just a promise to the compiler: "I need this type to start on a boundary that is a multiple of N."

### Why Over-Alignment Matters

Different hardware features impose different alignment requirements. If you hand a misaligned pointer to a SIMD load instruction, you get either a fault or a silent performance penalty, depending on the CPU generation. Cache-line alignment is different - it is not about correctness but about avoiding false sharing between threads.

| Use Case | Required Alignment |
| --- | --- |
| SSE (SIMD) | 16 bytes |
| AVX (SIMD) | 32 bytes |
| AVX-512 | 64 bytes |
| Cache line isolation | 64 bytes |
| GPU buffers | 256+ bytes |
| Memory-mapped I/O | Page size (4096) |

### `std::aligned_alloc` (C11/C++17)

When you need a raw byte buffer at a specific alignment, the C11/C++17 function is:

```cpp
void* aligned_alloc(size_t alignment, size_t size);
```

**Constraints:**

- `alignment` must be a power of 2
- `size` must be a **multiple of `alignment`** (unlike `posix_memalign`)
- Freed with `std::free()`

The size constraint is the one that trips people up most often. If you want 100 bytes aligned to 64, you have to round 100 up to 128 before passing it in.

### C++17 Aligned `operator new`

Before C++17, `new T` for over-aligned types was **undefined behavior** - the default `operator new` only guaranteed `alignof(std::max_align_t)`. C++17 fixed this by adding an overload that carries the alignment requirement along:

```cpp
void* operator new(std::size_t size, std::align_val_t align);
```

Now `new T` for over-aligned `T` **automatically** calls the aligned version. You do not need to do anything special; the compiler wires it up for you. This is the preferred path for heap-allocating over-aligned class types in C++17 and later.

---

## Self-Assessment

### Q1: Declare an SIMD-friendly struct with `alignas(32)` and verify alignment with `alignof`

The key thing to watch here is that both the stack-allocated instance and the heap-allocated one come back with the correct alignment - the compiler handles both cases in C++17.

```cpp
#include <iostream>
#include <cstddef>
#include <cstdint>
#include <new>

// SIMD-friendly struct for AVX operations (256-bit = 32 bytes)
struct alignas(32) AVXVector {
    float data[8];  // 8 x 4 bytes = 32 bytes, fits one AVX register
};

// Cache-line aligned struct to prevent false sharing
struct alignas(64) CacheLineData {
    int counter;
    char padding[60];  // Fill the rest of the cache line
};

// Multiple alignment levels
struct alignas(16) SSEData { float v[4]; };
struct alignas(32) AVXData { float v[8]; };
struct alignas(64) AVX512Data { float v[16]; };

int main() {
    std::cout << "=== Alignment verification ===\n";
    std::cout << "alignof(float):        " << alignof(float) << "\n";         // 4
    std::cout << "alignof(double):       " << alignof(double) << "\n";        // 8
    std::cout << "alignof(max_align_t):  " << alignof(std::max_align_t) << "\n"; // 16
    std::cout << "alignof(AVXVector):    " << alignof(AVXVector) << "\n";     // 32
    std::cout << "alignof(CacheLineData):" << alignof(CacheLineData) << "\n"; // 64

    std::cout << "\n=== Size includes padding for alignment ===\n";
    std::cout << "sizeof(AVXVector):     " << sizeof(AVXVector) << "\n";      // 32
    std::cout << "sizeof(CacheLineData): " << sizeof(CacheLineData) << "\n";  // 64

    std::cout << "\n=== Stack-allocated alignment ===\n";
    AVXVector v;
    std::cout << "AVXVector address: " << &v << "\n";
    std::cout << "Aligned to 32? " << (reinterpret_cast<uintptr_t>(&v) % 32 == 0 ? "YES" : "NO") << "\n";

    std::cout << "\n=== Heap-allocated alignment (C++17) ===\n";
    AVXVector* hv = new AVXVector;
    std::cout << "Heap AVXVector address: " << hv << "\n";
    std::cout << "Aligned to 32? " << (reinterpret_cast<uintptr_t>(hv) % 32 == 0 ? "YES" : "NO") << "\n";
    delete hv;

    std::cout << "\n=== Comparison table ===\n";
    std::cout << "Type          | alignof | sizeof\n";
    std::cout << "SSEData       |   " << alignof(SSEData) << "    |  " << sizeof(SSEData) << "\n";
    std::cout << "AVXData       |   " << alignof(AVXData) << "    |  " << sizeof(AVXData) << "\n";
    std::cout << "AVX512Data    |   " << alignof(AVX512Data) << "    |  " << sizeof(AVX512Data) << "\n";

    return 0;
}
```

**Output:**

```text
=== Alignment verification ===
alignof(float):        4
alignof(double):       8
alignof(max_align_t):  16
alignof(AVXVector):    32
alignof(CacheLineData):64

=== Size includes padding for alignment ===
sizeof(AVXVector):     32
sizeof(CacheLineData): 64

=== Stack-allocated alignment ===
AVXVector address: 0x7ffe...e0
Aligned to 32? YES

=== Heap-allocated alignment (C++17) ===
Heap AVXVector address: 0x55e...e0
Aligned to 32? YES
```

Both stack and heap allocations come out correctly aligned. In C++17, that is the expected behavior - nothing extra needed on your part.

### Q2: Show that `new T` handles over-alignment in C++17 with aligned `operator new`

The `Tracked` struct below overrides the aligned `operator new` just to make the call visible. In real code you would not do this - C++17 routes it automatically.

```cpp
#include <iostream>
#include <cstddef>
#include <cstdint>
#include <new>

struct alignas(64) CacheLine {
    int value;
};

struct Normal {
    int value;
};

// Custom aligned operator new to prove it's called
struct alignas(128) Tracked {
    int data;

    static void* operator new(std::size_t size, std::align_val_t align) {
        std::cout << "[Tracked::operator new] size=" << size
                  << ", align=" << static_cast<size_t>(align) << "\n";
        return ::operator new(size, align);
    }

    static void operator delete(void* p, std::size_t size, std::align_val_t align) {
        std::cout << "[Tracked::operator delete] size=" << size
                  << ", align=" << static_cast<size_t>(align) << "\n";
        ::operator delete(p, size, align);
    }
};

int main() {
    // Normal type: uses standard operator new (alignment <= max_align_t)
    std::cout << "=== Normal type ===\n";
    Normal* n = new Normal{42};
    std::cout << "address: " << n << "\n";
    std::cout << "alignof: " << alignof(Normal) << "\n";
    delete n;

    // Over-aligned type: C++17 automatically calls aligned operator new
    std::cout << "\n=== Over-aligned type (alignas(64)) ===\n";
    CacheLine* cl = new CacheLine{99};
    std::cout << "address: " << cl << "\n";
    std::cout << "alignof: " << alignof(CacheLine) << "\n";
    std::cout << "aligned to 64? "
              << (reinterpret_cast<uintptr_t>(cl) % 64 == 0 ? "YES" : "NO") << "\n";
    delete cl;

    // Custom type that logs aligned new
    std::cout << "\n=== Tracked (alignas(128)) ===\n";
    Tracked* t = new Tracked{7};
    std::cout << "address: " << t << "\n";
    std::cout << "aligned to 128? "
              << (reinterpret_cast<uintptr_t>(t) % 128 == 0 ? "YES" : "NO") << "\n";
    delete t;

    // Array of over-aligned objects
    std::cout << "\n=== Array of CacheLine ===\n";
    CacheLine* arr = new CacheLine[4];
    for (int i = 0; i < 4; ++i) {
        std::cout << "arr[" << i << "] at " << &arr[i]
                  << " aligned? " << (reinterpret_cast<uintptr_t>(&arr[i]) % 64 == 0 ? "YES" : "NO")
                  << "\n";
    }
    delete[] arr;

    return 0;
}
```

Notice that the `Tracked` output shows the aligned overload being called - and that `delete` also uses the aligned version. Mismatch between aligned `new` and unaligned `delete` would be UB, so the compiler pairs them correctly.

### Q3: Use `std::aligned_alloc` and explain the restriction that `size` must be a multiple of `alignment`

The size restriction is the main gotcha with `std::aligned_alloc`. The fix is simple: round `size` up to the next multiple of `alignment` before the call. The example below shows both the correct usage and a practical SIMD buffer allocation.

```cpp
#include <iostream>
#include <cstdlib>
#include <cstdint>
#include <cstring>
#include <new>

int main() {
    // std::aligned_alloc(alignment, size)
    // RULE: size MUST be a multiple of alignment!

    constexpr size_t alignment = 64;

    // Correct: size (128) is a multiple of alignment (64)
    std::cout << "=== Correct usage: size is multiple of alignment ===\n";
    {
        size_t size = 128;  // 128 / 64 = 2, OK
        void* p = std::aligned_alloc(alignment, size);
        if (p) {
            std::cout << "allocated " << size << " bytes at " << p << "\n";
            std::cout << "aligned to " << alignment << "? "
                      << (reinterpret_cast<uintptr_t>(p) % alignment == 0 ? "YES" : "NO") << "\n";
            std::free(p);
        }
    }

    // If you need size that's NOT a multiple, round up!
    std::cout << "\n=== Rounding up size to meet requirement ===\n";
    {
        size_t wanted = 100;  // Not a multiple of 64
        // Round up: ((wanted + alignment - 1) / alignment) * alignment
        size_t actual = ((wanted + alignment - 1) / alignment) * alignment;  // = 128

        std::cout << "Wanted " << wanted << " bytes, rounded to " << actual << "\n";

        void* p = std::aligned_alloc(alignment, actual);
        if (p) {
            std::cout << "allocated at " << p << "\n";
            std::cout << "aligned? "
                      << (reinterpret_cast<uintptr_t>(p) % alignment == 0 ? "YES" : "NO") << "\n";
            std::free(p);
        }
    }

    // Practical example: SIMD-friendly buffer
    std::cout << "\n=== SIMD buffer allocation ===\n";
    {
        constexpr size_t simd_align = 32;  // AVX
        constexpr size_t num_floats = 100;
        size_t raw_size = num_floats * sizeof(float);  // 400 bytes
        size_t alloc_size = ((raw_size + simd_align - 1) / simd_align) * simd_align;  // 416

        float* buffer = static_cast<float*>(std::aligned_alloc(simd_align, alloc_size));
        if (buffer) {
            std::cout << "Buffer for " << num_floats << " floats\n";
            std::cout << "Raw size: " << raw_size << ", Alloc size: " << alloc_size << "\n";
            std::cout << "Address: " << buffer << "\n";
            std::cout << "Aligned to " << simd_align << "? "
                      << (reinterpret_cast<uintptr_t>(buffer) % simd_align == 0 ? "YES" : "NO") << "\n";

            // Initialize with SIMD-friendly pattern
            for (size_t i = 0; i < num_floats; ++i) buffer[i] = static_cast<float>(i);

            std::free(buffer);
        }
    }

    // Comparison of allocation functions
    std::cout << "\n=== Alignment allocation options ===\n";
    std::cout << "Function             | Header    | Requirements\n";
    std::cout << "std::aligned_alloc() | <cstdlib> | size % alignment == 0, free() to release\n";
    std::cout << "operator new(sz,al)  | <new>     | C++17, delete with alignment\n";
    std::cout << "posix_memalign()     | POSIX     | No size restriction, free() to release\n";
    std::cout << "_aligned_malloc()    | MSVC      | Use _aligned_free() to release!\n";

    return 0;
}
```

The MSVC row in the table is worth remembering: `_aligned_malloc` requires `_aligned_free`, not plain `free`. Mixing them is UB and typically a runtime crash.

---

## Notes

- **Prefer `new T` for over-aligned types in C++17** - it handles alignment automatically.
- Use `std::aligned_alloc` when you need raw byte buffers with specific alignment (e.g., SIMD, DMA).
- On Windows/MSVC, `std::aligned_alloc` may not be available - use `_aligned_malloc`/`_aligned_free` instead.
- `alignas` applies to types and variables; `alignof` queries the alignment requirement.
- Cache line alignment (`alignas(64)`) prevents false sharing in multithreaded code.
