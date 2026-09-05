# Understand branch-free programming techniques for hot paths

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/operator_arithmetic>  

---

## Topic Overview

Every time the CPU encounters a conditional branch, it tries to predict which way it will go. When that prediction is wrong, the pipeline has to flush and restart - costing around 15-20 cycles on modern x86. In a hot loop that runs millions of times, even a 5% misprediction rate adds up fast. Branchless code sidesteps this entirely by using conditional moves (`cmov`), arithmetic masks, and bitwise tricks to compute both outcomes and select the right one - no prediction needed.

Here is a quick overview of the main techniques and when to reach for each one:

| Technique | Mechanism | When to use |
| --- | --- | --- |
| `cmov` | CPU selects result without branching | min/max, abs, clamp |
| Arithmetic mask | `(x >> 31)` produces 0 or -1 | sign-dependent selection |
| Bitwise select | `(mask & a) \| (~mask & b)` | branchless mux |
| Lookup table | Precomputed results | small domain functions |
| `? :` with `-O2` | Compiler may emit cmov | simple ternaries |

To make the difference concrete, here is what the assembly looks like for a simple min - with and without a branch:

```asm
Branchy (misprediction penalty):     Branchless (no penalty):
  cmp eax, ebx                        cmp eax, ebx
  jle .L1        <- branch!           cmovg eax, ebx   <- no branch
  mov eax, ebx
.L1:
```

The `cmov` version does the same comparison but never redirects the instruction stream. The pipeline keeps moving regardless of the outcome.

---

## Self-Assessment

### Q1: Branchless min using conditional move

There are three ways to write a branchless min. The simplest is to let the compiler handle it - a plain ternary at `-O2` almost always produces `cmov`. The arithmetic version shows you the underlying mechanism, which is useful to understand even if you rarely write it by hand.

```cpp
#include <iostream>
#include <algorithm>

// Branchy min (compiler may or may not use cmov):
int branchy_min(int a, int b) {
    if (a < b) return a;  // branch instruction
    return b;
}

// Guaranteed branchless min using ternary (compiler emits cmov at -O2):
int branchless_min(int a, int b) {
    return a < b ? a : b;
    // Assembly (g++ -O2):
    //   cmp  edi, esi
    //   cmovg edi, esi   <- conditional move, NO branch
    //   mov  eax, edi
}

// Branchless min using arithmetic (manual approach):
int arithmetic_min(int a, int b) {
    // (a - b) >> 31 is 0 if a >= b, or -1 (all 1s) if a < b
    int diff = a - b;  // Note: undefined for overflow; safe for small values
    int mask = diff >> 31;  // arithmetic shift: 0 or 0xFFFFFFFF
    return b + (diff & mask); // if a < b: b + (a-b) = a. if a >= b: b + 0 = b
}

int main() {
    std::cout << branchless_min(5, 3) << '\n';   // 3
    std::cout << branchless_min(2, 7) << '\n';   // 2
    std::cout << arithmetic_min(5, 3) << '\n';   // 3
    std::cout << arithmetic_min(2, 7) << '\n';   // 2

    // Verify on Compiler Explorer:
    //   g++ -O2: both branchless_min and std::min emit cmov
    //   The ternary operator is the simplest way to get cmov
}
```

The key takeaway is that you rarely need to write the arithmetic version yourself. Use the ternary form and check the assembly on Compiler Explorer if you want to confirm `cmov` was generated.

### Q2: Branchless clamp using arithmetic masking

Clamping is just two back-to-back min/max operations, and the same pattern applies - two `cmov` instructions instead of two branches. The arithmetic mask version spells out exactly what the hardware is doing under the hood.

```cpp
#include <iostream>
#include <algorithm>

// Branchy clamp:
int branchy_clamp(int x, int lo, int hi) {
    if (x < lo) return lo;     // branch 1
    if (x > hi) return hi;     // branch 2
    return x;
}

// Branchless clamp using ternary (compiler emits cmov):
int branchless_clamp(int x, int lo, int hi) {
    // Two cmov instructions, no branches:
    int t = x < lo ? lo : x;   // cmov
    return t > hi ? hi : t;    // cmov
    // Assembly:
    //   cmp edi, esi
    //   cmovl edi, esi    ; clamp to lo
    //   cmp edi, edx
    //   cmovg edi, edx    ; clamp to hi
}

// Branchless clamp using sign-extension mask:
int mask_clamp(int x, int lo, int hi) {
    // Step 1: clamp to lower bound
    int underflow = lo - x;            // positive if x < lo
    int mask_lo = ~(underflow >> 31);  // 0xFFFFFFFF if x < lo, else 0
    x = (lo & mask_lo) | (x & ~mask_lo);

    // Step 2: clamp to upper bound
    int overflow = x - hi;             // positive if x > hi
    int mask_hi = ~(overflow >> 31);   // 0xFFFFFFFF if x > hi, else 0
    x = (hi & mask_hi) | (x & ~mask_hi);
    return x;
}

int main() {
    // Test all three:
    for (int x : {-5, 0, 3, 7, 15}) {
        std::cout << x << " -> "
                  << branchless_clamp(x, 0, 10) << " "
                  << mask_clamp(x, 0, 10) << '\n';
    }
    // Output:
    // -5 -> 0 0
    // 0  -> 0 0
    // 3  -> 3 3
    // 7  -> 7 7
    // 15 -> 10 10
}
```

Both approaches give the same results. In practice, prefer the ternary form - it is cleaner and the compiler will produce equally good code.

### Q3: Branchless binary search vs branchy

This is where branchless coding really shines. A classic binary search has a 50% misprediction rate on random queries because each comparison is essentially a coin flip. The branchless version replaces the conditional branch with arithmetic, trading a 15-20 cycle mispredict penalty for a predictable 1-cycle `cmov`.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>
#include <random>
#include <numeric>

// Standard (branchy) binary search:
int branchy_lower_bound(const int* arr, int n, int target) {
    int lo = 0, hi = n;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] < target)
            lo = mid + 1;   // branch (50% mispredict for random queries)
        else
            hi = mid;
    }
    return lo;
}

// Branchless binary search (Algorithmica-style):
int branchless_lower_bound(const int* arr, int n, int target) {
    const int* base = arr;
    int len = n;
    while (len > 1) {
        int half = len / 2;
        // cmov: no branch, always executes both paths:
        base += (base[half - 1] < target) * half;  // arithmetic, no branch
        len -= half;
    }
    return static_cast<int>(base - arr) + (*base < target);
}

// Why branchless wins for random lookups:
// - Branchy: each comparison is 50/50 -> ~50% mispredict rate
//   -> log2(N) * 15 cycles penalty
// - Branchless: cmov has ~1 cycle latency, no mispredict
//   -> log2(N) * ~5 cycles total
//
// When branchy wins:
// - Sorted/sequential queries (predictor learns the pattern)
// - Very small arrays (fits in L1, branch cost hidden)

int main() {
    constexpr int N = 1'000'000;
    std::vector<int> arr(N);
    std::iota(arr.begin(), arr.end(), 0);

    std::mt19937 rng(42);
    std::vector<int> queries(N);
    for (auto& q : queries) q = rng() % N;

    auto bench = [&](auto fn, const char* label) {
        volatile int sink = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int q : queries) sink = fn(arr.data(), N, q);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0);
        std::cout << label << ": " << ns.count() / N << " ns/query\n";
    };

    bench(branchy_lower_bound,    "Branchy   ");
    bench(branchless_lower_bound, "Branchless");
    // Typical results (N=1M, random queries):
    // Branchy:    ~200 ns/query
    // Branchless: ~120 ns/query  (40% faster)
}
```

The ~40% win comes entirely from eliminating mispredictions. Notice the `(base[half - 1] < target) * half` trick - that boolean-to-integer multiply is the key. The CPU evaluates the comparison, converts it to 0 or 1, and multiplies. No branch, no mispredict.

---

## Notes

- At `-O2`/`-O3`, compilers often convert ternary `? :` to `cmov` automatically.
- Use `__builtin_expect(expr, val)` or `[[likely]]`/`[[unlikely]]` for branch hints when branchless is not an option.
- Branchless code is NOT always faster - measure! Predictable branches can be faster because the predictor eliminates the penalty entirely.
- `perf stat -e branch-misses` reveals whether a branchless rewrite is actually worthwhile for your workload.
- SIMD branchless: use `_mm_min_epi32` / `_mm_max_epi32` for vectorized clamp.
