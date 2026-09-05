# Write branchless code using conditional moves and arithmetic tricks

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://godbolt.org>  

---

## Topic Overview

Every time the CPU encounters a branch (an `if`/`else`), its branch predictor makes a guess about which way the branch will go and speculatively executes instructions along that path. When the guess is wrong, the CPU has to throw away all that speculative work and start over - a penalty of roughly 15-20 cycles. For random or unpredictable data, this can be the biggest bottleneck in a tight loop.

Branchless code avoids the problem entirely by replacing the conditional jump with a conditional move (`cmov`), which selects between two values without any pipeline disruption. The penalty drops from 15-20 cycles (when mispredicted) to a flat ~2 cycles every time.

```asm
Branch version:                     Branchless (cmov):
  cmp eax, ebx                        cmp eax, ebx
  jl .take_a                          cmovl ecx, eax    ; select a if a<b
  mov ecx, ebx                        cmovge ecx, ebx   ; select b otherwise
  jmp .done                           ; NO jump, NO prediction needed!
  .take_a:
  mov ecx, eax
  .done:
  // 15-20 cycle penalty if mispredicted  // Always ~2 cycles
```

The table below is the key decision guide. Notice that the break-even point matters: if a branch is highly predictable, the predictor costs almost nothing, and switching to branchless actually makes things slower.

| Pattern | Branch cost | Branchless cost | Use branchless? |
| --- | --- | --- | --- |
| Predictable (95%+ one way) | ~0 cycles | ~2 cycles | No |
| Unpredictable (50/50) | ~10 cycles avg | ~2 cycles | Yes |
| Both paths expensive | ~10 + path_cost | both_costs | No |

---

## Self-Assessment

### Q1: Branchless min and verify on Compiler Explorer

The three versions here demonstrate the progression from explicit branch to ternary to arithmetic trick. The ternary is the most practical - compilers at `-O2` reliably convert `a < b ? a : b` to a `cmov` instruction. The arithmetic version is more fragile (it overflows for large integers) but illustrates the underlying bit-manipulation principle.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// Branch version: if/else
int min_branch(int a, int b) {
    if (a < b) return a;  // cmp + jl (branch)
    return b;
    // Assembly:
    //   cmp edi, esi
    //   jl .L2          <- BRANCH (mispredicts ~50% for random data)
    //   mov eax, esi
    //   ret
    //   .L2: mov eax, edi
    //   ret
}

// Branchless version 1: ternary (compiler often uses cmov)
int min_ternary(int a, int b) {
    return a < b ? a : b;
    // Assembly (with -O2):
    //   cmp edi, esi
    //   cmovle eax, edi  <- CONDITIONAL MOVE (no branch!)
    //   ret
}

// Branchless version 2: arithmetic trick
int min_arith(int a, int b) {
    // x86 trick: uses the sign bit of (a - b)
    int diff = a - b;
    return b + (diff & (diff >> 31));
    // If a < b: diff < 0, diff>>31 = -1 (all 1s), diff & -1 = diff, b + diff = a
    // If a >= b: diff >= 0, diff>>31 = 0, diff & 0 = 0, b + 0 = b
    // WARNING: overflow if a-b overflows int range!
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<int> a(N), b(N);
    std::mt19937 rng(42);
    for (int i = 0; i < N; ++i) {
        a[i] = rng() % 1000;  // random: branch predictor fails ~50%
        b[i] = rng() % 1000;
    }

    auto bench = [&](auto fn, const char* label) {
        volatile int result = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) result = fn(a[i], b[i]);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(min_branch,  "Branch  ");
    bench(min_ternary, "Ternary ");
    bench(min_arith,   "Arith   ");
    // Random data: Branch ~350ms, Ternary ~150ms, Arith ~160ms
    // Sorted data: Branch ~120ms (predictor succeeds), Ternary ~150ms
}
// Verify on godbolt.org: compile with -O2, look for cmovl (branchless)
```

The sorted-data row in the comment is the critical counterpoint. When data is sorted, the branch predictor sees "take left branch every time" and achieves near-perfect prediction. In that case, the branch version at ~120ms beats the branchless ternary at ~150ms. This is why you should always measure before converting branches to branchless form.

### Q2: `std::min` compiles to cmov

You don't always have to write branchless code explicitly. The standard library functions `std::min`, `std::max`, and `std::clamp` already compile to `cmov` instructions at `-O2`. This is free branchless code that you get just by using the standard library correctly.

```cpp
#include <iostream>
#include <algorithm>  // std::min
#include <vector>
#include <chrono>
#include <random>

// std::min is typically branchless at -O2!
int use_std_min(int a, int b) {
    return std::min(a, b);
    // GCC -O2 assembly:
    //   cmp edi, esi
    //   mov eax, esi
    //   cmovle eax, edi    <- cmov! branchless!
    //   ret
    //
    // Clang -O2 may also use:
    //   cmp edi, esi
    //   cmovl esi, edi
    //   mov eax, esi
    //   ret
}

// std::max is also branchless:
int use_std_max(int a, int b) {
    return std::max(a, b);
    // cmp + cmovge -> branchless
}

// std::clamp is two cmovs:
int use_std_clamp(int val, int lo, int hi) {
    return std::clamp(val, lo, hi);
    // cmp + cmovl + cmp + cmovg -> two conditional moves
}

// Absolute value: also branchless at -O2
int abs_branchless(int x) {
    return x < 0 ? -x : x;
    // Assembly:
    //   mov eax, edi
    //   neg eax
    //   cmovs eax, edi  <- conditional move on sign flag
    //   ret
    // Alternative: (x ^ (x >> 31)) - (x >> 31)  [two's complement trick]
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 2000 - 1000;

    volatile int r = 0;
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 1; i < N; ++i)
        r = std::min(static_cast<int>(r), data[i]);
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    std::cout << "std::min chain: " << ms.count() << " ms\n";  // ~120ms
    std::cout << "Result: " << r << '\n';  // minimum element
}
```

The practical takeaway here is that you can often get branchless benefits just by preferring `std::min`/`std::max`/`std::clamp` over explicit `if` statements - no intrinsics, no bit tricks, just standard C++.

### Q3: When branchless code is slower

The reason this trips people up is that "branchless is always faster" sounds like a rule, but it isn't. The fundamental issue is that a conditional move always evaluates both sides, whereas a correctly predicted branch only executes the side you take. When the branch is predictable or when one of the paths is expensive, the branch wins.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>
#include <string>

// Case 1: PREDICTABLE branches are FASTER than branchless
bool mostly_true(int x) { return x > 0; }  // 99% true

void branch_predictable(const int* data, int n) {
    volatile int sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] > 0)  // 99% taken -> predictor almost perfect
            sum += data[i];
        // Branch: ~0 cycles (99% predicted correctly)
        // cmov: ~2 cycles EVERY time -> worse!
    }
}

// Case 2: EXPENSIVE side effects -- can't use branchless
void expensive_paths(const std::vector<int>& keys,
                     const std::vector<std::string>& cache) {
    for (int k : keys) {
        if (k < static_cast<int>(cache.size())) {
            // Cheap path: read from cache
            volatile auto& s = cache[k];
        } else {
            // Expensive path: would need DB lookup, allocation, etc.
            // Branchless would execute BOTH paths -> wasteful!
        }
    }
}

// Case 3: Floating-point -- no cmov (uses SSE min/max instead)
float fmin_branch(float a, float b) {
    return a < b ? a : b;
    // Uses: vminss xmm0, xmm0, xmm1 (SSE min, always branchless)
    // Compiler ALWAYS makes this branchless for float!
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<int> mostly_pos(N);
    std::mt19937 rng(42);
    // 99% positive values -> branch is predictable
    for (auto& x : mostly_pos)
        x = (rng() % 100 < 99) ? (rng() % 1000 + 1) : -(rng() % 1000);

    std::cout << "When NOT to use branchless code:\n";
    std::cout << "  1. Highly predictable branch (>95% one direction)\n";
    std::cout << "     -> predictor costs ~0, cmov costs ~2 cycles\n";
    std::cout << "  2. Paths with side effects or expensive work\n";
    std::cout << "     -> cmov evaluates BOTH sides always\n";
    std::cout << "  3. Floating-point comparisons\n";
    std::cout << "     -> compiler already uses minss/maxss (branchless)\n";
    std::cout << "\nProfile first: perf stat -e branch-misses ./app\n";
    std::cout << "If branch-miss rate < 1%: branch is fine, don't optimize\n";
}
```

The workflow the last two lines describe is the right one: run `perf stat -e branch-misses,branches ./app` first and look at the misprediction rate. If it's below about 1%, your predictor is doing a great job and branchless code will only hurt. If it's above 5-10%, you have a candidate for conversion.

---

## Notes

- `std::min`, `std::max`, `std::clamp` all compile to cmov at `-O2`.
- Use Compiler Explorer to verify: look for `cmov` in the assembly.
- Branchless wins when branch misprediction rate > 5-10%.
- `perf stat -e branch-misses,branches` tells you the misprediction rate.
- For SIMD: all comparisons are inherently branchless (mask + blend).
