# Use profile-guided optimization (PGO) for data-driven compiler optimization

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization>  

---

## Topic Overview

Compilers are clever, but they're working with incomplete information at compile time. They don't know whether a particular branch is taken 1% of the time or 99% of the time. They don't know which functions are called billions of times and which are called once during startup. This limits how well static optimization can work.

Profile-Guided Optimization (PGO) fixes this by running your actual program on representative input, collecting data about what really happens at runtime, and then feeding that data back to the compiler. With PGO, the compiler can lay out likely branches first so the CPU branch predictor wins more often, inline only the hot functions (not every function it could inline), allocate registers more carefully in the loops that matter, and group hot code together so the instruction cache stays warm. The result is typically 10-30% faster code for programs with complex branching.

The workflow is always three steps. First, build an instrumented binary. Second, run it on real or representative data. Third, compile again using the profile to guide optimization.

```cpp
PGO workflow (3 steps):

1. INSTRUMENT           2. PROFILE              3. OPTIMIZE

   g++ -fprofile-generate   ./app < test_data      g++ -fprofile-use
   -> app (instrumented)    -> *.gcda files         -> app (optimized)
   (slower, collects data)  (run representative     (uses profile to
                             workload)               optimize hot paths)
```

Here's a quick summary of what the compiler does with that data:

| PGO Benefit | How |
| --- | --- |
| Better branch prediction | Compiler lays out likely branches first |
| Better inlining | Inline hot functions, don't inline cold ones |
| Better register allocation | Prioritize hot loops |
| Code layout | Group hot code together (fewer icache misses) |

---

## Self-Assessment

### Q1: Run the three PGO steps

This program has a `classify()` function whose branches have very different likelihoods. Without PGO, the compiler has no idea which case dominates; with PGO, it knows case 3 covers 50% of inputs and can order the checks accordingly.

```cpp
// workload.cpp - a CPU-bound program to optimize with PGO
#include <vector>
#include <algorithm>
#include <numeric>
#include <iostream>
#include <random>

// This function has unpredictable branches without profile data:
int classify(int x) {
    if (x < 10)    return 0;   // 5% of inputs (rare)
    if (x < 100)   return 1;   // 15% of inputs
    if (x < 1000)  return 2;   // 30% of inputs
    return 3;                   // 50% of inputs (most common)
}

long long process(const std::vector<int>& data) {
    long long result = 0;
    for (int x : data) {
        int category = classify(x);
        switch (category) {
            case 0: result += x;     break;
            case 1: result += x * 2; break;
            case 2: result += x * 3; break;
            case 3: result += x * 4; break;  // most common path
        }
    }
    return result;
}

int main() {
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(1, 5000);
    std::vector<int> data(10'000'000);
    std::generate(data.begin(), data.end(), [&]{ return dist(rng); });

    auto result = process(data);
    std::cout << "Result: " << result << '\n';
}
```

The commands below show both GCC and Clang workflows. The key difference is that Clang's IR-based PGO (the `-fprofile-instr-*` flags) operates at the LLVM intermediate representation level, which gives the optimizer more precise information than GCC's frontend-level instrumentation.

```bash
# === GCC PGO ===
# Step 1: Instrument build
g++ -O2 -std=c++20 -fprofile-generate -o app_instr workload.cpp

# Step 2: Run with representative data (generates .gcda files)
./app_instr
# Creates: workload.gcda (profile data)

# Step 3: Optimized build using profile
g++ -O2 -std=c++20 -fprofile-use -o app_pgo workload.cpp

# === Clang PGO (IR-based, more accurate) ===
# Step 1: Instrument
clang++ -O2 -std=c++20 -fprofile-instr-generate -o app_instr workload.cpp

# Step 2: Profile
./app_instr
# Creates: default.profraw
llvm-profdata merge default.profraw -o workload.profdata

# Step 3: Optimize
clang++ -O2 -std=c++20 -fprofile-instr-use=workload.profdata -o app_pgo workload.cpp

# === CMake integration ===
# cmake -DCMAKE_CXX_FLAGS="-fprofile-generate" -B build-instr
# cmake --build build-instr && ./build-instr/app
# cmake -DCMAKE_CXX_FLAGS="-fprofile-use" -B build-pgo
# cmake --build build-pgo
```

Note the extra `llvm-profdata merge` step in the Clang workflow. Raw profile files can be split across multiple runs (useful for averaging over different workloads), and `llvm-profdata merge` combines them into one `.profdata` file before the final compilation.

### Q2: Measure PGO performance improvement

After running both builds, measure with `time` and verify with `perf stat`. The branch-miss numbers are especially telling because they confirm the compiler actually reordered the checks.

```bash
# Compare execution times:
$ time ./app_nopgo     # regular -O2 build
# real 0m1.82s

$ time ./app_pgo       # PGO-optimized build
# real 0m1.45s         # ~20% faster

# Why PGO helps here:
# 1. classify(): compiler reorders branches so case 3 (50%) is checked first
# 2. switch: compiler uses computed goto for hot cases
# 3. process(): hot loop gets better register allocation
# 4. code layout: hot path stays in instruction cache

# Verify with perf stat:
$ perf stat ./app_nopgo
#  5,200,000  branch-misses  (4.2% of branches)

$ perf stat ./app_pgo
#  1,100,000  branch-misses  (0.9% of branches)  <- 4x fewer
```

A 4x reduction in branch misses is substantial. Each branch miss on a modern CPU can cost 10-20 cycles as the pipeline is flushed and refilled. Across ten million iterations, that adds up fast.

### Q3: Risks of stale profiles

PGO is data-driven, which means it's only as good as the data. If the code changes significantly after you collect a profile - new branches, different hot paths, functions that got renamed or inlined - the profile no longer matches the code. The compiler may still use it, but it'll be optimizing for a workload that no longer exists. The result can actually be slower than a plain `-O2` build in some cases.

```cpp
Stale profile problem:

v1.0 code:                  v2.0 code (changed control flow):
  if (x < 100) {              if (is_premium(x)) {    // NEW
    hot_path();   <-- 90%       premium_path();        // never profiled!
  }                           }
  Profile says: hot_path=90%  Profile: hot_path=90%... but it's gone!
  Compiler trusts this.       Compiler misoptimizes based on stale data.
```

The good news is that both GCC and Clang will warn you when the profile data no longer matches the source. Make it a habit to treat these warnings seriously.

When to regenerate profiles:

- **After any control-flow changes** (new branches, removed functions)
- **After changing hot data structures** (different memory access patterns)
- **Before every release build** (best practice)
- **NOT needed for:** comment changes, formatting, adding new cold paths

```bash
# Clang: detect stale profiles
clang++ -fprofile-instr-use=old.profdata \
        -Wprofile-instr-out-of-date \
        workload_v2.cpp
# warning: profile data may be out of date: of 10 functions,
#          3 have mismatched data that will be ignored

# GCC: similar warning
g++ -fprofile-use -Wmissing-profile workload_v2.cpp
# warning: profile count data file not found for function 'is_premium'

# CI best practice: automate PGO in pipeline
# 1. Build instrumented binary
# 2. Run benchmark suite to collect profile
# 3. Build optimized binary with fresh profile
# 4. Run tests on optimized binary
# 5. Deploy
```

Automating the full three-step PGO process in CI ensures you always ship with a fresh, accurate profile.

---

## Notes

- PGO typically gives 10-30% speedup for branch-heavy code.
- Profile data is compiler-specific (GCC profiles don't work with Clang).
- Use representative workloads for profiling; benchmark suites work well.
- Clang's IR-based PGO (`-fprofile-instr-*`) is more accurate than frontend PGO.
- BOLT (Facebook's Binary Optimization and Layout Tool) is post-link PGO for even more gains.
