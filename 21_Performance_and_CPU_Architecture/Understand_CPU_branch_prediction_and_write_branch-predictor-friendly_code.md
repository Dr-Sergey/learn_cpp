# Understand CPU branch prediction and write branch-predictor-friendly code

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/likely>  

---

## Topic Overview

Writing branch-predictor-friendly code means structuring your branches so the common path is predictable and falls through naturally, minimizing the costly pipeline flushes that mispredictions cause. There are a few concrete techniques you can reach for, ranging from simple ordering choices to eliminating branches entirely.

If you remember only one thing from this topic: a branch the CPU cannot predict costs you 15-20 cycles every time it mispredicts. For a tight loop, that can easily 3-4x your runtime. The techniques below address this at different levels.

| Technique | Description | Benefit |
| --- | --- | --- |
| Common case fall-through | Put likely path as the "not-taken" branch | Predict-not-taken is default for forward branches |
| `[[likely]]` / `[[unlikely]]` | C++20 branch hints | Compiler arranges code layout |
| `__builtin_expect` | GCC/Clang branch hint | Pre-C++20 equivalent |
| Branchless arithmetic | Replace branch with `cmov` or math | Eliminates misprediction entirely |
| Loop peeling | Handle boundary cases outside loop | Removes branches from hot loop |

The most powerful option - and the cleanest fallback when you have truly unpredictable data - is to eliminate the branch entirely with arithmetic or conditional moves:

```cpp
Branchy hot loop:                   Branchless equivalent:
  for each x:                        for each x:
    if (x >= 128)   <- 50% miss        sum += (x >= 128) * x;
      sum += x;                         ^-- no branch, just multiply
```

---

## Self-Assessment

### Q1: Mispredict penalty on out-of-order CPUs

Understanding why the penalty is specifically 10-20 cycles helps you reason about when it matters and when it does not. The key is that modern out-of-order CPUs keep a large window of speculative work in flight - far more than the simple pipeline picture suggests.

```cpp
#include <iostream>

// Modern out-of-order (OoO) CPUs execute instructions speculatively:
//
// 1. FETCH STAGE: Branch predictor guesses direction before decode
// 2. SPECULATIVE EXECUTION: CPU executes ~100+ instructions past the branch
// 3. ON CORRECT: Speculative results committed (free!)
// 4. ON MISPREDICT:
//    a. All speculative work is DISCARDED
//    b. Pipeline is FLUSHED (14-20 stages emptied)
//    c. Fetch restarts from correct target
//    d. ~10-20 cycles of no useful work ("pipeline bubble")
//
// Why 10-20 cycles specifically?
//   - Pipeline depth determines the cost
//   - Skylake: 14-19 stages -> ~15 cycle penalty
//   - The OoO engine can partially hide this by doing other work,
//     but in tight loops there's nothing else to do
//
// Practical impact:
//   CPU at 4 GHz, 4-wide issue:
//   1 mispredict = 15 cycles * 4 IPC = 60 instruction slots wasted
//   If loop body is 10 instructions:
//     Perfect prediction: 10/4 = 2.5 cycles per iteration
//     50% mispredict:     2.5 + 0.5 * 15 = 10 cycles per iteration (4x slower!)

int main() {
    std::cout << "Mispredict penalty breakdown:\n";
    std::cout << "  1. Detect mispredict: ~3 cycles (at execute stage)\n";
    std::cout << "  2. Flush pipeline:    ~2 cycles\n";
    std::cout << "  3. Refetch correct:   ~5 cycles (icache hit)\n";
    std::cout << "  4. Refill pipeline:   ~5 cycles\n";
    std::cout << "  Total: ~15 cycles on Skylake\n";
}
```

The reason this trips people up is the phrase "tight loops there's nothing else to do." An out-of-order CPU can often hide a mispredict by executing independent instructions from other parts of the program while the pipeline refills. But in a hot loop the CPU is fully committed to executing loop iterations - there is no slack work to absorb the penalty.

### Q2: Fall-through path optimization

Compilers and CPUs both have a bias toward forward-not-taken branches. The compiler exploits this when it lays out code: it tries to place the common path as straight-line instructions and put rare paths (error handling, cold branches) out-of-line. You can help the compiler make the right choice by using `[[likely]]`.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// "Fall-through" = the path taken when the branch is NOT taken.
// Static prediction: forward branches are predicted NOT-taken.
// Compilers arrange code so the likely path is the fall-through.

// BAD: common case requires a branch (jump over error handling)
[[gnu::noinline]] int process_bad(int x) {
    if (x < 0) {          // rare: error case
        return -1;         // <- fall-through (but x<0 is rare!)
    }
    return x * 2;          // <- common case requires jump
}

// GOOD: common case is fall-through
[[gnu::noinline]] int process_good(int x) {
    if (x >= 0) [[likely]] {  // common: 99% of cases
        return x * 2;         // <- fall-through (common case!)
    }
    return -1;                 // <- rare error path
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = (rng() % 100 == 0) ? -1 : rng() % 1000;
    // 1% negative, 99% positive

    auto bench = [&](auto fn, const char* label) {
        volatile long long sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int x : data) sum += fn(x);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(process_bad,  "Error-first  ");
    bench(process_good, "Common-first ");
    // Difference is small here (predictor learns), but code layout
    // affects icache: common-first keeps hot code contiguous.
}
```

The hardware predictor will learn the 99% pattern in both versions, so the runtime difference is modest. But the code layout effect is real: keeping the common path as straight-line code means it occupies fewer instruction cache lines and is less likely to compete with other hot code for cache space.

### Q3: Branchless arithmetic eliminates misprediction

When the data is genuinely random and the predctor has no chance to learn, the only reliable fix is to eliminate the branch entirely. Branchless arithmetic replaces the conditional jump with a multiplication or bitmask that the CPU executes without any prediction at all.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// The ultimate branch-predictor-friendly code: NO BRANCHES.

// Branchy version:
[[gnu::noinline]] long long sum_branchy(const int* data, int n) {
    long long sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] >= 128)     // branch: ~50% mispredict for random data
            sum += data[i];
    }
    return sum;
}

// Branchless version 1: multiply by boolean
[[gnu::noinline]] long long sum_branchless_mul(const int* data, int n) {
    long long sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += static_cast<long long>(data[i] >= 128) * data[i];
        // (data[i] >= 128) is 0 or 1
        // Multiply: either 0 * data[i] = 0, or 1 * data[i] = data[i]
        // No branch instruction generated!
    }
    return sum;
}

// Branchless version 2: bitwise mask
[[gnu::noinline]] long long sum_branchless_mask(const int* data, int n) {
    long long sum = 0;
    for (int i = 0; i < n; ++i) {
        int mask = -(data[i] >= 128);  // 0x00000000 or 0xFFFFFFFF
        sum += data[i] & mask;
    }
    return sum;
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 256;  // random: worst case for branch predictor

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile auto s = fn(data.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms (sum=" << s << ")\n";
    };

    bench(sum_branchy,         "Branchy        ");
    bench(sum_branchless_mul,  "Branchless *   ");
    bench(sum_branchless_mask, "Branchless mask");
    // Typical (random data, N=10M):
    // Branchy:         ~25 ms  (50% branch mispredict)
    // Branchless *:    ~8 ms   (no branches, auto-vectorized)
    // Branchless mask: ~8 ms   (same, different approach)
    // 3x speedup from eliminating branches!
}
```

Notice that the branchless versions also tend to auto-vectorize better. Without a branch breaking the data dependency chain, the compiler can transform the inner loop into SIMD instructions that process multiple elements simultaneously, stacking a vectorization speedup on top of the branch-elimination speedup.

---

## Notes

- `[[likely]]` / `[[unlikely]]` (C++20) affect compiler code layout, not hardware prediction.
- `__builtin_expect(expr, val)` is the pre-C++20 GCC/Clang equivalent.
- The compiler at `-O2`+ often converts simple branches to `cmov` automatically.
- For complex conditions, manually writing branchless code can beat the compiler.
- Profile first with `perf stat -e branch-misses` to confirm misprediction is the bottleneck.
