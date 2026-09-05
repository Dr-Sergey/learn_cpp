# Use std::transform_reduce for parallel map-reduce operations

**Category:** Standard Library Algorithms  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/transform_reduce>  

---

## Topic Overview

`std::transform_reduce` fuses a **map** step (unary or binary transform) with a **reduce** step (associative+commutative fold) into a single algorithm that the implementation can parallelise and vectorise when given an execution policy. The fusion matters because it means no intermediate container of transformed values needs to exist - everything is computed in a single pass.

### Overloads

| Overload | Parameters | Equivalent to |
| --- | --- | --- |
| **Unary** | `(first, last, init, reduce_op, transform_op)` | `reduce(transform(each), init, reduce_op)` |
| **Binary (inner product)** | `(first1, last1, first2, init, reduce_op, transform_op)` | `reduce(transform(a[i],b[i]), init, reduce_op)` |
| **Default inner product** | `(first1, last1, first2, init)` | like `std::inner_product` |

All overloads accept an optional execution policy as the first argument.

### Key Rules

1. **`reduce_op`** must be associative and commutative - the library may evaluate in any order or partition.
2. Neither `reduce_op` nor `transform_op` may invalidate iterators or modify elements.
3. The default reduce is `std::plus<>()` and default transform is `std::multiplies<>()` (inner product).

### Comparison with Related Algorithms

| Algorithm | Introduced | Parallel? | Notes |
| --- | --- | --- | --- |
| `std::accumulate` | C++98 | No | Left fold, order guaranteed |
| `std::inner_product` | C++98 | No | Binary transform + left fold |
| `std::reduce` | C++17 | Yes | Parallel fold (no transform) |
| `std::transform_reduce` | C++17 | Yes | Parallel map+fold |

### Core Syntax

Here's the essential pattern - a sum of squares and a dot product - before diving into the exercises:

```cpp
#include <numeric>
#include <execution>
#include <vector>
#include <cmath>
#include <iostream>

int main() {
    std::vector<double> v{3.0, 4.0};

    // --- Unary transform_reduce: sum of squares ---
    double sum_sq = std::transform_reduce(
        v.begin(), v.end(),
        0.0,                          // init
        std::plus<>(),                // reduce
        [](double x){ return x*x; }  // transform
    );
    std::cout << "sum of squares = " << sum_sq << "\n";
    // Output: sum of squares = 25

    // L2 norm
    std::cout << "L2 norm = " << std::sqrt(sum_sq) << "\n";
    // Output: L2 norm = 5

    // --- Binary transform_reduce: dot product ---
    std::vector<double> a{1,2,3}, b{4,5,6};
    double dot = std::transform_reduce(a.begin(), a.end(), b.begin(), 0.0);
    std::cout << "dot product = " << dot << "\n";
    // Output: dot product = 32

    // --- Parallel version ---
    double par_dot = std::transform_reduce(
        std::execution::par_unseq,
        a.begin(), a.end(), b.begin(), 0.0);
    std::cout << "parallel dot = " << par_dot << "\n";
    // Output: parallel dot = 32
}
```

Notice that the binary overload with just four arguments (no explicit `reduce_op` or `transform_op`) defaults to inner product behavior - multiply elements pairwise, then sum. This is intentional: it makes `transform_reduce` a parallelisable drop-in for `std::inner_product`.

---

## Self-Assessment

### Q1: Compute the L2 norm of a vector using transform_reduce with a squared element transform

The L2 norm requires squaring each element and summing the results before taking a square root. `transform_reduce` does the squaring and summing in a single pass without allocating a temporary vector of squares.

```cpp
#include <numeric>
#include <vector>
#include <cmath>
#include <iostream>

int main() {
    std::vector<double> v{1.0, 2.0, 3.0, 4.0, 5.0};

    // Map each element to x*x, then reduce with addition
    double sum_sq = std::transform_reduce(
        v.begin(), v.end(),
        0.0,                          // init (identity for +)
        std::plus<>(),                // reduce op
        [](double x) { return x * x; } // transform op
    );

    double l2 = std::sqrt(sum_sq);
    std::cout << "sum of squares = " << sum_sq << "\n";
    std::cout << "L2 norm        = " << l2    << "\n";
    // Output:
    // sum of squares = 55
    // L2 norm        = 7.41620
}
```

The argument order is worth memorizing: `(range, init, reduce_op, transform_op)`. Many people accidentally swap `reduce_op` and `transform_op`. A good mnemonic is to read it inside-out - the transform is the innermost operation (applied to each element first), and the reduce is the outer accumulation.

---

### Q2: Parallelize the operation with std::execution::par_unseq and verify correctness

Adding `std::execution::par_unseq` as the first argument tells the implementation it may split the work across threads and apply SIMD vectorization within each chunk. The correctness question is subtle: floating-point addition is not mathematically associative, so different groupings can produce slightly different results.

```cpp
#include <numeric>
#include <execution>
#include <vector>
#include <cmath>
#include <iostream>
#include <cassert>

int main() {
    // Build a large vector for meaningful parallelism
    constexpr int N = 1'000'000;
    std::vector<double> v(N);
    for (int i = 0; i < N; ++i)
        v[i] = static_cast<double>(i + 1);  // 1..1000000

    // Sequential version
    double seq = std::transform_reduce(
        v.begin(), v.end(),
        0.0,
        std::plus<>(),
        [](double x) { return x * x; }
    );

    // Parallel + vectorised version
    double par = std::transform_reduce(
        std::execution::par_unseq,          // execution policy
        v.begin(), v.end(),
        0.0,
        std::plus<>(),
        [](double x) { return x * x; }
    );

    std::cout << "sequential = " << seq << "\n";
    std::cout << "parallel   = " << par << "\n";

    // Floating-point results may differ slightly due to reduction order
    double rel_err = std::abs(seq - par) / seq;
    std::cout << "relative error = " << rel_err << "\n";
    assert(rel_err < 1e-10);  // practically identical

    std::cout << "L2 norm = " << std::sqrt(par) << "\n";
}
```

The reason this trips people up is that they expect exact equality but get a tiny rounding difference. This is not a bug - it's a consequence of the library being allowed to combine partial results in any order. The relative error shown here is typically on the order of 1e-13 or smaller: negligible for engineering use, but something to be aware of if you are comparing outputs bit-for-bit.

---

### Q3: Explain why transform_reduce requires associativity and commutativity for parallel safety

**Associativity** means `(a op b) op c == a op (b op c)`. This is needed because the library splits the range into chunks and reduces each chunk independently, then combines chunk results. Different groupings must produce the same answer.

**Commutativity** means `a op b == b op a`. This is needed because there is **no guarantee** on the order in which elements are fed to the reduction - chunks may finish in any order, and within a chunk SIMD lanes may be combined in any order.

**Counter-example - subtraction is NOT safe:**

```cpp
#include <numeric>
#include <execution>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v{10, 3, 2, 1};

    // Sequential accumulate with minus: ((((0-10)-3)-2)-1) = -16
    int acc = std::accumulate(v.begin(), v.end(), 0, std::minus<>());
    std::cout << "accumulate (left fold): " << acc << "\n";
    // Output: accumulate (left fold): -16

    // transform_reduce with minus - UNDEFINED grouping!
    // The library may compute (10-3) - (2-1) = 6, or (10-2) - (3-1) = 6,
    // or many other groupings. Result is unpredictable.
    int tr = std::transform_reduce(
        std::execution::par,
        v.begin(), v.end(),
        0,
        std::minus<>(),   // NOT associative+commutative
        [](int x){ return x; }
    );
    std::cout << "transform_reduce (parallel minus): " << tr << "\n";
    // Output: UNSPECIFIED - could be anything
}
```

**Safe operations for the reduce step:**

- `std::plus<>()` - addition
- `std::multiplies<>()` - multiplication
- `std::bit_and<>()`, `std::bit_or<>()`, `std::bit_xor<>()`
- `std::min`, `std::max` (via lambda wrappers)
- Any user-defined associative+commutative binary op

**Unsafe operations:**

- Subtraction, division, string concatenation (non-commutative), floating-point addition (technically non-associative, but accepted in practice with understanding that results may vary by a few ULP).

---

## Notes

- `transform_reduce` lives in `<numeric>`, not `<algorithm>`.
- The binary overload `(first1, last1, first2, init)` defaults to inner product - equivalent to `std::inner_product` but parallelisable.
- For very small ranges the overhead of thread dispatch may make the parallel version slower - profile before switching.
- `std::reduce` is the special case where the transform is the identity function.
- Combine with `std::execution::par` (parallel but not vectorised) when the transform has side effects that must not be interleaved within a thread (rare - ideally transforms are pure).
- On MSVC, link with `/openmp` or use the default thread pool. On GCC/libstdc++ link with `-ltbb`.
