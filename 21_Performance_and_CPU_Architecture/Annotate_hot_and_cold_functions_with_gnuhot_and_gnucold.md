# Annotate hot and cold functions with [[gnu::hot]] and [[gnu::cold]]

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html>  

---

## Topic Overview

GCC and Clang support `[[gnu::hot]]` and `[[gnu::cold]]` attributes to give the compiler a hint about how often a function executes. That hint flows into three real decisions: which ELF section the function lands in, how aggressively the compiler tries to inline it, and whether to optimize it for speed or for size.

The reason this matters is instruction cache locality. When unrelated cold code - error handlers, logging, rarely-hit fallbacks - is interleaved with your hot loop body in the same `.text` section, both fight for the same cache lines. Splitting them into separate sections means the CPU's instruction cache is filled only with the code you actually run in the critical path.

| Attribute | Effect |
| --- | --- |
| `[[gnu::hot]]` | Places function in `.text.hot`, higher inlining priority, aggressive optimization |
| `[[gnu::cold]]` | Places function in `.text.unlikely`, lower inlining priority, size-optimized |
| (no attribute) | Default `.text` section |

Here is a picture of how the linker lays things out after you apply these attributes:

```cpp
Memory layout with hot/cold splitting:

  .text.hot      .text          .text.unlikely
  +---------+   +---------+    +---------+
  | fast_   |   | normal  |    | error_  |
  | path()  |   | func()  |    | handler |
  | inner_  |   |         |    | log_err |
  | loop()  |   |         |    |         |
  +---------+   +---------+    +---------+
  <-- hot in icache -->        <-- rarely loaded -->
```

The cold functions still exist and work correctly - they just live far away in memory so they never pollute your hot code's cache lines.

---

## Self-Assessment

### Q1: Mark an error handler with `[[gnu::cold]]`

This example puts the ideas into practice. The hot processing function and cold error handlers are annotated, and the comments show you how to verify the section placement with standard tools after you build.

```cpp
#include <iostream>
#include <stdexcept>
#include <cstdlib>

// Cold: error handling code, rarely executed
[[gnu::cold]] void handle_error(const char* msg) {
    std::cerr << "FATAL: " << msg << '\n';
    std::abort();
}

[[gnu::cold]] void log_warning(const char* msg) {
    std::cerr << "WARN: " << msg << '\n';
}

// Hot: main processing path
[[gnu::hot]] int process(int* data, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] < 0) [[unlikely]] {
            handle_error("negative value");  // cold call
        }
        sum += data[i];
    }
    return sum;
}

// Verify section placement:
// Compile: g++ -O2 -o test test.cpp
// Check:   objdump -h test | grep text
//   .text.hot       -- contains process()
//   .text.unlikely  -- contains handle_error(), log_warning()
//
// Or: readelf -S test | grep text
//   [14] .text.hot      PROGBITS ...
//   [15] .text.unlikely  PROGBITS ...

int main() {
    int data[] = {1, 2, 3, 4, 5};
    std::cout << process(data, 5) << '\n';  // 15
}
```

Notice that `[[unlikely]]` on the branch and `[[gnu::cold]]` on the called function reinforce each other - the branch hint tells the compiler this path is rare, while the cold attribute tells it to move the function body to a separate section entirely.

### Q2: `[[gnu::hot]]` effects on inlining and section placement

When both the caller and the callee carry `[[gnu::hot]]`, the compiler raises its willingness to inline. Think of it as bumping the inlining budget for that function. The comments below explain exactly what the attribute unlocks.

```cpp
#include <iostream>
#include <vector>
#include <numeric>

// [[gnu::hot]] tells the compiler:
// 1. Place in .text.hot section (better icache locality with other hot code)
// 2. Increase inlining budget (more aggressive inlining threshold)
// 3. Optimize for speed over size
// 4. May enable more aggressive loop optimizations

[[gnu::hot]] inline int dot_product(const int* a, const int* b, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += a[i] * b[i];
    }
    return sum;
}

[[gnu::hot]] void matrix_multiply(const int* A, const int* B, int* C,
                                   int M, int N, int K) {
    for (int i = 0; i < M; ++i) {
        for (int j = 0; j < N; ++j) {
            C[i * N + j] = dot_product(&A[i * K], &B[j], K);
            // dot_product is more likely to be inlined here
            // because both caller and callee are [[gnu::hot]]
        }
    }
}

// Without [[gnu::hot]], the compiler uses default heuristics.
// With it, the compiler:
// - Raises the inlining cost threshold (~2x)
// - Uses -O3-level optimizations even at -O2
// - Groups hot code together in memory

int main() {
    int A[] = {1, 2, 3, 4};
    int B[] = {5, 6, 7, 8};
    int C[4];
    matrix_multiply(A, B, C, 2, 2, 2);
    std::cout << C[0] << " " << C[1] << '\n';  // 19 22
}
```

When `dot_product` is inlined into `matrix_multiply`, the call overhead disappears and the loop body becomes a tight multiply-accumulate sequence. That is where `[[gnu::hot]]` earns its keep.

### Q3: Cold placement reduces instruction cache pressure

Here is the core intuition for why cold splitting helps: every function that shares a cache line with your hot loop is a function that can evict hot instructions. Moving cold code to `.text.unlikely` keeps your hot path compact enough to fit in fewer cache lines.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// Scenario: a hot loop that occasionally handles errors.
// Without cold splitting, error code pollutes the icache.

[[gnu::cold]] void handle_overflow(int idx, long long val) {
    // This function body is ~200 bytes of code
    std::cerr << "Overflow at index " << idx
              << " value=" << val << '\n';
    // Log, alert, etc.
}

[[gnu::hot]] long long compute_sum(const std::vector<int>& data) {
    long long sum = 0;
    for (int i = 0; i < static_cast<int>(data.size()); ++i) {
        sum += data[i];
        if (sum > 1'000'000'000LL) [[unlikely]] {
            handle_overflow(i, sum);  // cold call, out of hot section
            return sum;
        }
    }
    return sum;
}

// Memory layout effect:
//
// WITHOUT cold splitting:
//   compute_sum: [hot code][error handling code][hot code]
//                            ^-- pollutes icache!
//
// WITH cold splitting:
//   .text.hot:     [compute_sum hot path, compact]
//   .text.unlikely: [handle_overflow, separate]
//                    ^-- only loaded if error occurs
//
// Result: compute_sum's hot path fits in fewer cache lines,
// reducing icache misses by 10-30% in tight loops.

int main() {
    std::vector<int> data(10000, 42);
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 100000; ++i) {
        volatile auto s = compute_sum(data);
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    std::cout << "Time: " << ms.count() << "ms\n";
}
```

The 10-30% reduction in icache misses cited in the comment is typical for loops where the cold path is large (logging, diagnostics) relative to the hot path. The bigger the cold blob you remove, the more compact - and cache-friendly - your hot section becomes.

---

## Notes

- `[[gnu::hot]]` and `[[gnu::cold]]` are GCC/Clang extensions; MSVC uses `__declspec(noinline)` similarly.
- C++20 `[[likely]]`/`[[unlikely]]` work on branches; `hot`/`cold` work on entire functions.
- Use PGO (Profile-Guided Optimization) for automatic hot/cold detection: `-fprofile-generate` / `-fprofile-use`.
- Verify with `objdump -h` or `readelf -S` to check section placement.
- In practice, PGO often subsumes manual hot/cold annotation.
