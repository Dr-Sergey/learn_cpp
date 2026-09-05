# Use std::inclusive_scan and exclusive_scan for prefix sum computation

**Category:** Standard Library Algorithms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/inclusive_scan>  

---

## Topic Overview

C++17 added `std::inclusive_scan` and `std::exclusive_scan` as parallelizable alternatives to `std::partial_sum`. The core difference between the two is easy to remember: "inclusive" means each output element includes the input element at the same position; "exclusive" means it does not - the input element at that position is shifted one slot to the right in the output.

### Comparison

Given input `{a, b, c, d}` and op `+`, init `0`:

| Algorithm | Output | Index 0 | Notes |
| --- | --- | --- | --- |
| `inclusive_scan` | `{a, a+b, a+b+c, a+b+c+d}` | First element | Like `partial_sum` |
| `exclusive_scan` | `{0, a, a+b, a+b+c}` | Init value | Shifted right by 1 |

A concrete picture often helps more than the table:

```cpp
Input:           [3,  1,  4,  1,  5]
inclusive_scan:  [3,  4,  8,  9, 14]  // each element includes itself
exclusive_scan:  [0,  3,  4,  8,  9]  // each element excludes itself
```

### Signatures

Here are the overloads you will reach for most often:

```cpp
#include <numeric>

// inclusive_scan
inclusive_scan(first, last, dest);                       // default: +
inclusive_scan(first, last, dest, binary_op);            // custom op
inclusive_scan(first, last, dest, binary_op, init);      // custom op + init
inclusive_scan(policy, first, last, dest, ...);          // parallel

// exclusive_scan (init is required)
exclusive_scan(first, last, dest, init);                 // default: +
exclusive_scan(first, last, dest, init, binary_op);      // custom op
exclusive_scan(policy, first, last, dest, init, ...);    // parallel
```

Both require the binary operation to be **associative** for parallel execution to be correct.

---

## Self-Assessment

### Q1: Compute a prefix sum using std::inclusive_scan and verify against manual accumulation

This example first computes a prefix sum with `inclusive_scan`, then builds the same thing by hand to confirm the results match. It also shows two other useful variations: running maximum and running product, both of which fall out naturally by just swapping the operation.

```cpp
#include <iostream>
#include <vector>
#include <numeric>

int main() {
    std::vector<int> input = {3, 1, 4, 1, 5, 9, 2, 6};

    // === inclusive_scan: prefix sum ===
    std::vector<int> prefix(input.size());
    std::inclusive_scan(input.begin(), input.end(), prefix.begin());

    std::cout << "Input:          ";
    for (int x : input) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "inclusive_scan: ";
    for (int x : prefix) std::cout << x << " ";
    std::cout << "\n";
    // inclusive_scan: 3 4 8 9 14 23 25 31

    // === Manual verification ===
    std::vector<int> manual(input.size());
    manual[0] = input[0];
    for (size_t i = 1; i < input.size(); ++i) {
        manual[i] = manual[i - 1] + input[i];
    }

    std::cout << "Manual:         ";
    for (int x : manual) std::cout << x << " ";
    std::cout << "\n";
    // Manual: 3 4 8 9 14 23 25 31

    std::cout << "Match: " << std::boolalpha << (prefix == manual) << "\n";
    // Match: true

    // === inclusive_scan with custom op: running max ===
    std::vector<int> running_max(input.size());
    std::inclusive_scan(input.begin(), input.end(), running_max.begin(),
                        [](int a, int b) { return std::max(a, b); });

    std::cout << "Running max:    ";
    for (int x : running_max) std::cout << x << " ";
    std::cout << "\n";
    // Running max: 3 3 4 4 5 9 9 9

    // === inclusive_scan with custom op: running product ===
    std::vector<long long> running_prod(input.size());
    std::inclusive_scan(input.begin(), input.end(), running_prod.begin(),
                        std::multiplies<long long>{});

    std::cout << "Running prod:   ";
    for (long long x : running_prod) std::cout << x << " ";
    std::cout << "\n";
    // Running prod: 3 3 12 12 60 540 1080 6480

    return 0;
}
```

The running-max scan is a nice example because it shows that `inclusive_scan` is not limited to addition. Any associative operation works - and `std::max` is associative.

### Q2: Use exclusive_scan for an offsets array where index 0 is always 0

The classic use case for `exclusive_scan` is computing offsets from sizes. If you have chunks of data with known sizes, the exclusive scan of those sizes gives you the start position of each chunk - and index 0 is always the `init` value (usually 0), which is exactly what you want for "where does the first chunk start?".

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <string>

int main() {
    // === Classic use case: compute byte offsets from sizes ===
    std::vector<int> sizes = {4, 2, 7, 3, 5};

    std::vector<int> offsets(sizes.size());
    std::exclusive_scan(sizes.begin(), sizes.end(), offsets.begin(), 0);

    std::cout << "Sizes:   ";
    for (int x : sizes) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "Offsets: ";
    for (int x : offsets) std::cout << x << " ";
    std::cout << "\n";
    // Sizes:   4 2 7 3 5
    // Offsets: 0 4 6 13 16
    // offsets[0] is always 0 (the init value)
    // offsets[i] = sum of sizes[0..i-1]

    // === Practical: flatten nested vectors ===
    std::vector<std::vector<int>> nested = {{10,20}, {30}, {40,50,60}, {70,80}};

    // Compute sizes
    std::vector<int> chunk_sizes;
    for (const auto& v : nested) chunk_sizes.push_back(v.size());

    // Compute start offsets
    std::vector<int> starts(chunk_sizes.size());
    std::exclusive_scan(chunk_sizes.begin(), chunk_sizes.end(), starts.begin(), 0);

    // Total size
    int total = starts.back() + chunk_sizes.back();

    // Flatten into one array
    std::vector<int> flat(total);
    for (size_t i = 0; i < nested.size(); ++i) {
        std::copy(nested[i].begin(), nested[i].end(), flat.begin() + starts[i]);
    }

    std::cout << "\nFlattened: ";
    for (int x : flat) std::cout << x << " ";
    std::cout << "\n";
    // Flattened: 10 20 30 40 50 60 70 80

    std::cout << "Chunk starts: ";
    for (int x : starts) std::cout << x << " ";
    std::cout << "\n";
    // Chunk starts: 0 2 3 6

    // === Range query with prefix sums ===
    // Sum of input[l..r] = prefix[r+1] - prefix[l]
    std::vector<int> data = {5, 3, 8, 2, 7};
    std::vector<int> psum(data.size() + 1);
    std::exclusive_scan(data.begin(), data.end(), psum.begin(), 0);
    psum.back() = psum[psum.size() - 2] + data.back();
    // Actually, easier approach for range queries:
    std::vector<int> ps(data.size() + 1, 0);
    std::inclusive_scan(data.begin(), data.end(), ps.begin() + 1);
    // ps = {0, 5, 8, 16, 18, 25}

    int l = 1, r = 3;  // sum of data[1..3] = 3+8+2 = 13
    std::cout << "\nSum[" << l << ".." << r << "] = "
              << ps[r + 1] - ps[l] << "\n";  // 13

    return 0;
}
```

The range-query trick at the end - `ps[r+1] - ps[l]` - is worth remembering. With an exclusive prefix sum (or an inclusive one shifted by one slot), any subrange sum becomes a single subtraction, which is O(1) after O(n) preprocessing.

### Q3: Parallelise a prefix scan using the par execution policy and explain the correctness requirements

To use a parallel execution policy, just pass `std::execution::par` as the first argument. The algorithm does the rest. The code below times sequential vs. parallel on 10 million doubles so you can see the speedup.

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <execution>
#include <chrono>

int main() {
    constexpr size_t N = 10'000'000;
    std::vector<double> data(N, 1.0);

    // === Sequential scan ===
    std::vector<double> result_seq(N);
    auto t1 = std::chrono::high_resolution_clock::now();
    std::inclusive_scan(std::execution::seq,
                        data.begin(), data.end(), result_seq.begin());
    auto t2 = std::chrono::high_resolution_clock::now();
    auto ms_seq = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    // === Parallel scan ===
    std::vector<double> result_par(N);
    t1 = std::chrono::high_resolution_clock::now();
    std::inclusive_scan(std::execution::par,
                        data.begin(), data.end(), result_par.begin());
    t2 = std::chrono::high_resolution_clock::now();
    auto ms_par = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    std::cout << "Sequential: " << ms_seq << " ms\n";
    std::cout << "Parallel:   " << ms_par << " ms\n";
    std::cout << "Last value: " << result_par.back() << "\n";  // 10000000

    // Verify correctness
    bool match = (result_seq == result_par);
    std::cout << "Results match: " << std::boolalpha << match << "\n";

    return 0;
}
// Compile: g++ -std=c++17 -O2 -ltbb scan_parallel.cpp
```

Parallel scans come with rules you must follow or the results will be wrong:

1. **Associativity:** The binary operation must be associative: `op(op(a,b), c) == op(a, op(b,c))`. The parallel algorithm splits the range into chunks, scans each independently, then merges. Non-associative ops (like subtraction) give wrong results.

2. **No side effects:** The operation is called from multiple threads. It must not modify shared state.

3. **Determinism:** For floating-point, parallel scan may group additions differently, causing tiny rounding differences. For integer `+`, results are exact.

4. **Init value (exclusive_scan):** Must be the identity element for the operation (0 for `+`, 1 for `*`).

---

## Notes

- **`partial_sum` vs scans:** `std::partial_sum` (C++03) is the non-parallelizable predecessor. Use `inclusive_scan` in new code - it supports execution policies and has clearer semantics.
- **In-place scan:** Source and destination can be the same range: `inclusive_scan(v.begin(), v.end(), v.begin())`.
- **`transform_inclusive_scan` / `transform_exclusive_scan`:** Apply a unary transform before scanning. Equivalent to `transform` -> `scan` but in a single pass.
- **Common use cases:** Range sum queries (games, competitive programming), histograms, load balancing offsets, parallel prefix operations.
- **Parallel scan algorithm:** Internally uses a two-pass approach - first a local scan per chunk, then an adjustment pass. Total work is O(n), span is O(log n).
