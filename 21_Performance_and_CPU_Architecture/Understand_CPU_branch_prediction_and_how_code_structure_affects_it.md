# Understand CPU branch prediction and how code structure affects it

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/likely>  

---

## Topic Overview

Modern CPUs do not wait at a branch to see which direction it goes. Instead they guess - speculatively executing instructions down the predicted path while the branch condition is still being evaluated. If the guess is right, the work is free; the CPU committed useful instructions. If the guess is wrong, all that speculative work is thrown away and the pipeline is flushed and restarted from the correct target. That flush costs roughly 15-20 cycles, which is a significant penalty when it happens frequently.

The hardware component that makes the guess is the **Branch Prediction Unit**, which maintains several cooperating structures:

```cpp
Branch Prediction Unit:

  Branch History Table (BHT)       Branch Target Buffer (BTB)
  +--------+--------+              +--------+-----------+
  | Branch | Pattern|              | Branch | Target    |
  |  addr  | (2-bit)|              |  addr  | address   |
  +--------+--------+              +--------+-----------+
  | 0x4010 | ST (11)|              | 0x4010 | 0x4020    |
  | 0x4030 | WN (01)|              | 0x4030 | 0x4050    |
  +--------+--------+              +--------+-----------+

  2-bit saturating counter:  ST=Strongly Taken, WT=Weakly Taken
                              WN=Weakly Not-taken, SN=Strongly Not-taken
```

The 2-bit saturating counter is the reason a simple loop (taken 99 times, then not taken once) only mispredicts at the loop exit - the counter stays in the "Strongly Taken" state for all iterations and needs two consecutive "not taken" outcomes to flip its prediction.

| Predictor type | Description |
| --- | --- |
| Static | Forward branches: not-taken; Back branches: taken (loops) |
| 1-bit | Remembers last outcome |
| 2-bit saturating | Needs 2 mispredictions to change state |
| TAGE (modern) | Multiple history lengths, 95%+ accuracy |
| Loop predictor | Detects loop iteration counts |

---

## Self-Assessment

### Q1: How the CPU branch predictor works

This code block gives you a mental model of all four major components. The comments walk through each one and explain how they interact. Read through it as documentation rather than executable logic - the `main` function at the bottom is just for illustration.

```cpp
#include <iostream>

// The CPU branch predictor has several components:
//
// 1. BRANCH HISTORY TABLE (BHT) / Pattern History Table (PHT)
//    - Maps branch PC (program counter) to a 2-bit saturating counter
//    - States: SN(00) -> WN(01) -> WT(10) -> ST(11)
//    - Taken increments, not-taken decrements
//    - Prediction: counter >= 2 -> predict taken
//
// 2. BRANCH TARGET BUFFER (BTB)
//    - Cache mapping branch address -> target address
//    - For indirect jumps (virtual calls, switch), stores predicted target
//    - Misprediction on target costs ~20 cycles (flush + refetch)
//
// 3. GLOBAL HISTORY REGISTER (GHR)
//    - Shift register of last N branch outcomes (taken/not-taken)
//    - Correlates branches: "if (a) ... if (b)" where b depends on a
//    - XORed with branch PC to index BHT (gshare predictor)
//
// 4. TAGE PREDICTOR (modern: Intel Haswell+, AMD Zen+)
//    - Multiple tables with different history lengths
//    - Short history for simple patterns, long history for complex
//    - 95-97% accuracy on typical code

int main() {
    // Example: 2-bit counter evolution for a loop branch
    // "for (int i = 0; i < 100; ++i) { body; }"
    //
    // Iteration:  1    2    3   ... 99   100 (exit)
    // Actual:     T    T    T   ... T    NT
    // Counter:    01   10   11  ... 11   10  (predict T)
    //                                     ^- only 1 mispredict at exit!
    //
    // Total mispredictions for 100-iteration loop: 1 (at exit)
    // Cost: 1 * 15 cycles = 15 cycles overhead

    std::cout << "Branch prediction overview:\n";
    std::cout << "  BHT: direction prediction (taken/not-taken)\n";
    std::cout << "  BTB: target address prediction\n";
    std::cout << "  GHR: correlating branch history\n";
    std::cout << "  TAGE: multiple history lengths (modern CPUs)\n";
    std::cout << "  Mispredict penalty: ~15-20 cycles (pipeline flush)\n";
}
```

The loop exit example is a nice sanity check. Even a completely naive predictor only mispredicts once per 100-iteration loop because the pattern is almost perfectly regular. The real damage comes from branches with genuinely unpredictable patterns, like a branch conditioned on random data.

### Q2: Common-case-first ordering improves throughput

Ordering branches so the most frequent outcome comes first is a simple code-structure technique that reduces the average number of comparisons per call. The predictor accuracy ends up the same either way, but you do less work to reach the common result.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>

enum class Type { Common, Rare1, Rare2, Rare3 };

// BAD: rare cases checked first
[[gnu::noinline]] int process_bad(Type t, int val) {
    if (t == Type::Rare3) return val * 7;    // 1% of cases
    if (t == Type::Rare2) return val * 5;    // 2% of cases
    if (t == Type::Rare1) return val * 3;    // 7% of cases
    return val * 2;                           // 90% of cases!
    // For 90% of inputs, we check 3 branches before reaching common case
    // All 3 branches are "not taken" -> predictor learns this, BUT
    // we waste 3 comparisons and add instruction cache pressure
}

// GOOD: common case first
[[gnu::noinline]] int process_good(Type t, int val) {
    if (t == Type::Common) [[likely]] return val * 2;  // 90%: first check!
    if (t == Type::Rare1) return val * 3;               // 7%
    if (t == Type::Rare2) return val * 5;               // 2%
    return val * 7;                                      // 1%
    // 90% of inputs take the FIRST branch -> 1 comparison
    // Predictor accuracy is same, but fewer instructions executed
}

int main() {
    constexpr int N = 10'000'000;
    std::mt19937 rng(42);
    std::vector<Type> types(N);
    for (auto& t : types) {
        int r = rng() % 100;
        if (r < 90) t = Type::Common;
        else if (r < 97) t = Type::Rare1;
        else if (r < 99) t = Type::Rare2;
        else t = Type::Rare3;
    }

    auto bench = [&](auto fn, const char* label) {
        volatile int sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) sum += fn(types[i], i);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(process_bad,  "Rare-first ");
    bench(process_good, "Common-first");
    // Typical: Common-first is 10-20% faster
}
```

The `[[likely]]` attribute on the first branch is the modern C++20 way to tell the compiler that this path should be the fall-through. The compiler uses that hint to arrange the generated code so the common case is straight-line execution without any jump.

### Q3: Use `perf stat` to measure branch misses

This is the classic benchmark that demonstrates why unsorted data is slower than sorted data. The branch `if (x >= 128)` is the same instruction in both cases - the only difference is whether the CPU can predict it.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <random>

// Program to demonstrate branch misprediction with sorted vs unsorted data.
// The classic "branch prediction" benchmark.

int main() {
    constexpr int N = 100'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 256;

    // Unsorted: values are random -> 50% branch mispredict rate
    long long sum1 = 0;
    for (int rep = 0; rep < 1000; ++rep) {
        for (int x : data) {
            if (x >= 128)  // random: ~50% taken -> unpredictable!
                sum1 += x;
        }
    }

    // Sort the data: now values are ordered
    // First half: all < 128 (always not-taken)
    // Second half: all >= 128 (always taken)
    // Branch predictor accuracy: ~99.9%
    std::sort(data.begin(), data.end());

    long long sum2 = 0;
    for (int rep = 0; rep < 1000; ++rep) {
        for (int x : data) {
            if (x >= 128)  // sorted: predictor learns pattern -> ~0% mispredict
                sum2 += x;
        }
    }

    std::cout << "Sum unsorted: " << sum1 << '\n';
    std::cout << "Sum sorted:   " << sum2 << '\n';

    // Compile and measure:
    //   g++ -O2 -o branch branch.cpp
    //   perf stat -e branches,branch-misses ./branch
    //
    // UNSORTED:
    //   branches:         1,500,000,000
    //   branch-misses:      500,000,000  (33%!)
    //   Time: ~8 seconds
    //
    // SORTED:
    //   branches:         1,500,000,000
    //   branch-misses:          100,000  (<0.01%)
    //   Time: ~2 seconds  (4x faster!)
    //
    // Alternative: branchless version eliminates the issue entirely:
    //   sum += (x >= 128) * x;  // no branch, works on unsorted data
}
```

With sorted data there is exactly one transition point - when the values cross 128 - and the predictor mispredicts only there. With random data every iteration is a coin flip, and the 4x slowdown you see is almost entirely due to pipeline flushes, not anything else. The `perf stat` command in the comments confirms this: you will see hundreds of millions of branch misses in the unsorted case and almost none in the sorted case.

---

## Notes

- Modern CPUs (TAGE predictor) achieve 95%+ accuracy on real code.
- `[[likely]]` / `[[unlikely]]` (C++20) hint the compiler for code layout, not the hardware predictor.
- `perf stat -e branch-misses` is the key tool for diagnosing branch issues.
- Sorting input data can eliminate branch mispredictions (but adds sort cost).
- Virtual function calls use the BTB; polymorphic call sites with many types cause BTB misses.
- Switch statements: compiler may use a jump table (indirect branch) or an if-else chain depending on density.
