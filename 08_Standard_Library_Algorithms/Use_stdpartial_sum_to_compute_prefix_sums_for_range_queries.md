# Use std::partial_sum to compute prefix sums for range queries

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/partial_sum>  

---

## Topic Overview

`std::partial_sum` (from `<numeric>`) computes running totals - each output element is the sum (or custom operation) of all preceding input elements including itself. Think of it as asking "what is the cumulative total up to and including this position?"

Here's the idea visualized:

```text
Input:        [3,  1,  4,  1,  5]
partial_sum:  [3,  4,  8,  9, 14]
              [a, a+b, a+b+c, ...]
```

### The O(1) Range Query Trick

Here's where `partial_sum` becomes genuinely powerful. Once you have a prefix sum array, you can answer "what is the sum of elements from index L to R?" in **O(1)** per query - no matter how large the array is:

$$\text{sum}(L, R) = \text{prefix}[R+1] - \text{prefix}[L]$$

The reason this works is simple: `prefix[R+1]` is the total from index 0 through R, and `prefix[L]` is the total from index 0 through L-1, so subtracting them leaves exactly the slice you want.

### Signatures

The function lives in `<numeric>` and supports both a default addition operation and a custom binary operation:

```cpp
#include <numeric>
partial_sum(first, last, dest);             // default: +
partial_sum(first, last, dest, binary_op);  // custom operation
```

---

## Self-Assessment

### Q1: Compute prefix sums with std::partial_sum and use them for O(1) range-sum queries

The trick here is allocating the prefix array with size `n+1` and starting the `partial_sum` output at `prefix.begin() + 1`. That way `prefix[0]` stays zero, and the range query formula `prefix[R+1] - prefix[L]` works cleanly for any L.

```cpp
#include <iostream>
#include <vector>
#include <numeric>

int main() {
    std::vector<int> data = {5, 3, 8, 2, 7, 4, 1, 6};

    // === Build prefix sum array ===
    // Use size+1 to have prefix[0] = 0 for cleaner range query formula
    std::vector<int> prefix(data.size() + 1, 0);
    std::partial_sum(data.begin(), data.end(), prefix.begin() + 1);

    std::cout << "Data:   ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "Prefix: ";
    for (int x : prefix) std::cout << x << " ";
    std::cout << "\n";
    // Data:   5 3 8 2 7 4 1 6
    // Prefix: 0 5 8 16 18 25 29 30 36

    // === O(1) range sum query ===
    // sum(L, R) = prefix[R+1] - prefix[L]

    auto range_sum = [&prefix](int l, int r) -> int {
        return prefix[r + 1] - prefix[l];
    };

    // Sum of data[1..3] = 3 + 8 + 2 = 13
    std::cout << "\nsum(1,3) = " << range_sum(1, 3) << "\n";  // 13

    // Sum of data[0..7] = entire array = 36
    std::cout << "sum(0,7) = " << range_sum(0, 7) << "\n";  // 36

    // Sum of data[4..6] = 7 + 4 + 1 = 12
    std::cout << "sum(4,6) = " << range_sum(4, 6) << "\n";  // 12

    // Sum of single element data[2] = 8
    std::cout << "sum(2,2) = " << range_sum(2, 2) << "\n";  // 8

    // === Practical: running balance ===
    std::vector<int> transactions = {100, -30, 50, -20, 80, -40};
    std::vector<int> balance(transactions.size());
    std::partial_sum(transactions.begin(), transactions.end(), balance.begin());

    std::cout << "\nTransaction history:\n";
    for (size_t i = 0; i < transactions.size(); ++i) {
        std::cout << "  " << (transactions[i] >= 0 ? "+" : "")
                  << transactions[i] << "  balance=" << balance[i] << "\n";
    }
    // +100 balance=100
    // -30  balance=70
    // +50  balance=120
    // ...

    return 0;
}
```

Notice that the lambda `range_sum` is O(1) - it does exactly one subtraction regardless of how large L-to-R is. All the work happened once at setup time.

### Q2: Show the difference between partial_sum (inclusive) and exclusive_scan for the same data

The distinction between inclusive and exclusive scans trips people up at first. Inclusive means the current element is included in the sum at that position. Exclusive means the sum at position `i` does not include `data[i]` - it only includes what came before. You can think of exclusive scan as "the total before I arrived."

```cpp
#include <iostream>
#include <vector>
#include <numeric>

int main() {
    std::vector<int> data = {3, 1, 4, 1, 5};

    // === partial_sum (inclusive) ===
    std::vector<int> inclusive(data.size());
    std::partial_sum(data.begin(), data.end(), inclusive.begin());

    // === exclusive_scan ===
    std::vector<int> exclusive(data.size());
    std::exclusive_scan(data.begin(), data.end(), exclusive.begin(), 0);

    // === inclusive_scan (C++17 - same result as partial_sum for +) ===
    std::vector<int> inc_scan(data.size());
    std::inclusive_scan(data.begin(), data.end(), inc_scan.begin());

    std::cout << "Data:            ";
    for (int x : data)      std::cout << x << "  ";
    std::cout << "\n";

    std::cout << "partial_sum:     ";
    for (int x : inclusive)  std::cout << x << "  ";
    std::cout << "\n";

    std::cout << "inclusive_scan:  ";
    for (int x : inc_scan)   std::cout << x << "  ";
    std::cout << "\n";

    std::cout << "exclusive_scan:  ";
    for (int x : exclusive)  std::cout << x << "  ";
    std::cout << "\n";

    //                       3   1   4   1   5
    // partial_sum:          3   4   8   9  14   (includes current element)
    // inclusive_scan:       3   4   8   9  14   (same as partial_sum for +)
    // exclusive_scan:       0   3   4   8   9   (excludes current, starts with init=0)

    // === Key difference ===
    // inclusive: output[i] = data[0] + data[1] + ... + data[i]
    // exclusive: output[i] = init + data[0] + data[1] + ... + data[i-1]
    //
    // exclusive[i] = inclusive[i-1]  (shifted right by one, with init at position 0)

    // === When to use which ===
    // Inclusive (partial_sum): running totals, cumulative statistics
    // Exclusive (exclusive_scan): offset/index arrays where position 0 = 0

    // Example: byte offsets for variable-length records
    std::vector<int> record_sizes = {10, 5, 8, 3, 12};
    std::vector<int> offsets(record_sizes.size());
    std::exclusive_scan(record_sizes.begin(), record_sizes.end(), offsets.begin(), 0);

    std::cout << "\nRecord offsets: ";
    for (int x : offsets) std::cout << x << " ";
    std::cout << "\n";
    // 0 10 15 23 26  - offset[0] = 0, always

    return 0;
}
```

The byte-offset example at the end is a great real-world use case for exclusive scan: record 0 starts at offset 0, record 1 starts after record 0, and so on. With an inclusive scan the first record would incorrectly have an offset equal to its own size.

### Q3: Use parallel prefix sum via std::inclusive_scan with std::execution::par

`std::partial_sum` cannot be parallelized because it enforces strict left-to-right execution - each result depends on the previous one. `std::inclusive_scan` with an execution policy breaks that constraint by allowing the implementation to use a parallel up-sweep/down-sweep algorithm.

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <execution>
#include <chrono>

int main() {
    constexpr size_t N = 5'000'000;
    std::vector<double> data(N, 1.0);

    // === partial_sum (NOT parallelizable - strict left-to-right) ===
    std::vector<double> ps_result(N);
    auto t1 = std::chrono::high_resolution_clock::now();
    std::partial_sum(data.begin(), data.end(), ps_result.begin());
    auto t2 = std::chrono::high_resolution_clock::now();
    auto ms_ps = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    // === inclusive_scan with seq (equivalent to partial_sum) ===
    std::vector<double> seq_result(N);
    t1 = std::chrono::high_resolution_clock::now();
    std::inclusive_scan(std::execution::seq,
                        data.begin(), data.end(), seq_result.begin());
    t2 = std::chrono::high_resolution_clock::now();
    auto ms_seq = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    // === inclusive_scan with par (PARALLEL!) ===
    std::vector<double> par_result(N);
    t1 = std::chrono::high_resolution_clock::now();
    std::inclusive_scan(std::execution::par,
                        data.begin(), data.end(), par_result.begin());
    t2 = std::chrono::high_resolution_clock::now();
    auto ms_par = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    std::cout << "partial_sum:         " << ms_ps << " ms\n";
    std::cout << "inclusive_scan(seq): " << ms_seq << " ms\n";
    std::cout << "inclusive_scan(par): " << ms_par << " ms\n";
    std::cout << "Last value: " << par_result.back() << "\n";  // 5000000

    // Verify correctness
    bool match = (ps_result.back() == par_result.back());
    std::cout << "Results match: " << std::boolalpha << match << "\n";

    // === Why partial_sum can't be parallelized ===
    // partial_sum guarantees left-to-right execution:
    //   result[0] = data[0]
    //   result[1] = result[0] + data[1]  <- depends on result[0]
    //   result[2] = result[1] + data[2]  <- depends on result[1]
    //
    // inclusive_scan allows arbitrary grouping (like reduce),
    // enabling a parallel up-sweep/down-sweep algorithm.

    return 0;
}
// Compile: g++ -std=c++17 -O2 -ltbb prefix_sum.cpp
```

The comments in the code spell out exactly why `partial_sum` is inherently sequential. If you need parallelism and are on C++17+, reach for `std::inclusive_scan` with `std::execution::par` instead.

---

## Notes

- **`partial_sum` is in `<numeric>`**, not `<algorithm>`.
- **In-place OK:** Source and destination can overlap: `partial_sum(v.begin(), v.end(), v.begin())`.
- **Custom binary ops:** `partial_sum(first, last, dest, multiplies<>{})` gives running products.
- **Prefer `inclusive_scan`** in new C++17+ code - it supports execution policies and is the modern replacement.
- **Range query formula:** With prefix sum `P` where `P[0] = 0`: `sum(L..R) = P[R+1] - P[L]`. This is O(n) setup, O(1) per query - very useful in competitive programming and real-time systems.
- **2D prefix sums:** Extend the technique to matrices for O(1) sub-rectangle sum queries.
