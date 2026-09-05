# Use `constexpr` Containers and Algorithms (C++20)

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm>  

---

## Topic Overview

### What Changed in C++20

C++20 made `std::vector` and `std::string` usable in `constexpr` contexts, and marked most `<algorithm>` functions as `constexpr`. This enables sophisticated compile-time computation with familiar standard library tools - the same code you'd write for runtime just works at compile time.

Here's a taste of what that looks like:

```cpp
consteval auto compute() {
    std::vector<int> v = {5, 3, 1, 4, 2};
    std::sort(v.begin(), v.end());       // constexpr sort!
    return v.front();                    // returns 1
}

static_assert(compute() == 1);
```

That's a runtime sort being called at compile time. No special machinery needed.

### Constexpr Allocation Rule (Transient Allocation)

There is one important constraint: memory allocated during constant evaluation must be **deallocated before the evaluation completes**. This is called a **transient allocation**, and it's the rule that makes `constexpr std::vector` possible without the binary containing heap pointers.

```cpp
constexpr int ok() {
    std::vector<int> v = {1, 2, 3};  // allocates
    int sum = v[0] + v[1] + v[2];
    return sum;                       // v is destroyed -> memory freed -> OK
}

// This would FAIL:
// constexpr std::vector<int> bad = {1, 2, 3};  // ERROR: allocation would persist!
```

The reason the second form fails is that a `constexpr` global variable needs to exist in the binary's read-only data segment. A `std::vector` contains a pointer to heap memory. There is no heap at compile time - the "allocator" during constant evaluation is a bookkeeping mechanism inside the compiler, not a real allocator. So the vector's internal buffer has nowhere to live once compilation finishes. The transient rule captures this: use the vector for its convenience during the computation, but don't try to carry it across the evaluation boundary.

### What's `constexpr` in C++20

| Component | `constexpr` in C++20? |
| --- | --- |
| `std::array` | Yes (since C++11/14) |
| `std::vector` | Yes (new in C++20) |
| `std::string` | Yes (new in C++20) |
| `std::sort`, `std::find`, `std::copy` | Yes |
| `std::lower_bound`, `std::upper_bound` | Yes |
| `std::accumulate`, `std::transform` | Yes |
| `new`/`delete` | Yes (transient only) |

---

## Self-Assessment

### Q1: Create a `constexpr std::vector` inside a `consteval` function and verify the result at compile time

The key pattern here is: do the complex, flexible work inside the `consteval` function using `std::vector`, then return a simple scalar or `std::array` as the result. The vector lives only during the evaluation and gets freed when the function returns.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>

// === consteval function using std::vector internally ===
consteval int sum_of_sorted_top3() {
    std::vector<int> v = {15, 3, 42, 8, 27, 1, 99, 11};

    // Sort descending
    std::sort(v.begin(), v.end(), std::greater<>{});

    // Sum top 3
    int sum = 0;
    for (int i = 0; i < 3 && i < static_cast<int>(v.size()); ++i) {
        sum += v[i];
    }
    return sum;
    // v's memory is freed here - transient allocation satisfied
}

// === consteval: find median of a collection ===
consteval double median_of(std::initializer_list<int> init) {
    std::vector<int> v(init);
    std::sort(v.begin(), v.end());

    auto n = v.size();
    if (n % 2 == 1) {
        return static_cast<double>(v[n / 2]);
    } else {
        return (v[n / 2 - 1] + v[n / 2]) / 2.0;
    }
}

// === consteval: unique elements count ===
consteval std::size_t count_unique(std::initializer_list<int> init) {
    std::vector<int> v(init);
    std::sort(v.begin(), v.end());
    auto it = std::unique(v.begin(), v.end());
    return static_cast<std::size_t>(it - v.begin());
}

// All verified at compile time
static_assert(sum_of_sorted_top3() == 99 + 42 + 27);  // 168
static_assert(median_of({5, 2, 8, 1, 9}) == 5.0);
static_assert(median_of({4, 2, 6, 8}) == 5.0);
static_assert(count_unique({1, 2, 2, 3, 3, 3, 4}) == 4);

int main() {
    constexpr int top3 = sum_of_sorted_top3();
    std::cout << "Sum of top 3: " << top3 << "\n";       // 168

    constexpr double med = median_of({10, 20, 30, 40, 50});
    std::cout << "Median: " << med << "\n";                // 30

    constexpr auto uniq = count_unique({1, 1, 2, 3, 3});
    std::cout << "Unique count: " << uniq << "\n";         // 3

    return 0;
}
```

**Expected output:**

```text
Sum of top 3: 168
Median: 30
Unique count: 3
```

### Q2: Explain why `constexpr` allocation requires that memory is deallocated before constant evaluation ends

This example explores the transient allocation rule in depth and shows the workaround pattern - sort using a `std::vector` internally, then copy the result into a `std::array` before returning:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <array>
#include <algorithm>
#include <cstddef>

// === The Rule: Transient constexpr allocation ===
// All memory allocated during constant evaluation MUST be freed
// before the evaluation finishes. Why?

// Reason 1: No runtime allocator at compile time
// The "allocator" during constant evaluation is a compile-time bookkeeping
// system. It cannot persist allocations into the final binary - there's no
// runtime heap address to embed in the executable.

// === VALID: transient allocation (freed before eval ends) ===
consteval int valid_vector_use() {
    std::vector<int> v = {10, 20, 30};  // allocates
    int total = 0;
    for (int x : v) total += x;
    return total;                        // v destroyed -> memory freed
}

static_assert(valid_vector_use() == 60);

// === VALID: string used transiently ===
consteval int valid_string_use() {
    std::string s = "hello world";       // allocates if > SSO
    return static_cast<int>(s.size());   // s destroyed -> freed
}

static_assert(valid_string_use() == 11);

// === INVALID: non-transient allocation ===
// constexpr std::vector<int> bad_global = {1, 2, 3};
// ERROR: allocation is not transient - vector's heap memory would
//        need to persist into the binary, which is impossible

// === VALID workaround: produce a fixed-size result ===
consteval auto vector_to_array() {
    std::vector<int> v = {5, 3, 1, 4, 2};
    std::sort(v.begin(), v.end());  // sort in vector

    // Copy to array (non-allocating, can persist)
    std::array<int, 5> result{};
    for (std::size_t i = 0; i < v.size(); ++i) {
        result[i] = v[i];
    }
    return result;
    // v freed here -> transient, result is std::array -> no allocation
}

constexpr auto sorted = vector_to_array();
static_assert(sorted[0] == 1);
static_assert(sorted[4] == 5);

int main() {
    std::cout << "=== Transient Allocation Rule ===\n\n";

    std::cout << "valid_vector_use() = " << valid_vector_use() << "\n";
    std::cout << "valid_string_use() = " << valid_string_use() << "\n";

    std::cout << "\nSorted array: ";
    for (int x : sorted) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "\n=== Why This Rule Exists ===\n";
    std::cout << "1. Compile-time 'heap' is bookkeeping - no real allocator\n";
    std::cout << "2. The binary has .data/.rodata/.bss - no heap section\n";
    std::cout << "3. A constexpr vector would need a pointer to heap memory\n";
    std::cout << "   that doesn't exist at build time\n";
    std::cout << "4. Solution: use vector/string transiently, output to\n";
    std::cout << "   std::array or scalar for the final constexpr result\n";

    return 0;
}
```

**Expected output:**

```text
=== Transient Allocation Rule ===

valid_vector_use() = 60
valid_string_use() = 11

Sorted array: 1 2 3 4 5

=== Why This Rule Exists ===

1. Compile-time 'heap' is bookkeeping - no real allocator
2. The binary has .data/.rodata/.bss - no heap section
3. A constexpr vector would need a pointer to heap memory
   that doesn't exist at build time
4. Solution: use vector/string transiently, output to
   std::array or scalar for the final constexpr result
```

### Q3: Write a `constexpr` binary search over a sorted `std::array` using `std::lower_bound`

`std::lower_bound` is `constexpr` in C++20, which means you can write a compile-time binary search over a sorted `constexpr std::array`. The `static_assert` checks at the bottom prove these run at compile time:

```cpp
#include <iostream>
#include <array>
#include <algorithm>
#include <cstddef>

// === Compile-time sorted array ===
constexpr std::array<int, 10> data = {2, 5, 8, 12, 16, 23, 38, 56, 72, 91};

// === constexpr binary search using std::lower_bound ===
constexpr bool contains(int value) {
    auto it = std::lower_bound(data.begin(), data.end(), value);
    return it != data.end() && *it == value;
}

// === constexpr index finder ===
constexpr int index_of(int value) {
    auto it = std::lower_bound(data.begin(), data.end(), value);
    if (it != data.end() && *it == value) {
        return static_cast<int>(it - data.begin());
    }
    return -1;  // not found
}

// === constexpr count elements in range [lo, hi) ===
constexpr int count_in_range(int lo, int hi) {
    auto first = std::lower_bound(data.begin(), data.end(), lo);
    auto last = std::lower_bound(data.begin(), data.end(), hi);
    return static_cast<int>(last - first);
}

// === Verify at compile time ===
static_assert(contains(5));
static_assert(contains(56));
static_assert(!contains(10));
static_assert(!contains(100));

static_assert(index_of(2) == 0);
static_assert(index_of(91) == 9);
static_assert(index_of(23) == 5);
static_assert(index_of(99) == -1);

static_assert(count_in_range(10, 50) == 3);   // 12, 16, 23
static_assert(count_in_range(0, 100) == 10);  // all elements

// === More complex: constexpr sorted array generation + search ===
template <std::size_t N>
consteval auto make_sorted_squares() {
    std::array<int, N> arr{};
    for (std::size_t i = 0; i < N; ++i) {
        arr[i] = static_cast<int>(i * i);
    }
    // Already sorted since i^2 is monotonic for i >= 0
    return arr;
}

constexpr auto squares = make_sorted_squares<20>();

constexpr bool is_perfect_square(int n) {
    auto it = std::lower_bound(squares.begin(), squares.end(), n);
    return it != squares.end() && *it == n;
}

static_assert(is_perfect_square(0));
static_assert(is_perfect_square(1));
static_assert(is_perfect_square(4));
static_assert(is_perfect_square(16));
static_assert(is_perfect_square(361));  // 19^2
static_assert(!is_perfect_square(2));
static_assert(!is_perfect_square(15));

int main() {
    std::cout << "=== Binary Search at Compile Time ===\n";
    std::cout << "contains(5):  " << contains(5) << "\n";     // 1
    std::cout << "contains(10): " << contains(10) << "\n";    // 0
    std::cout << "index_of(23): " << index_of(23) << "\n";    // 5
    std::cout << "index_of(99): " << index_of(99) << "\n";    // -1
    std::cout << "count_in_range(10,50): " << count_in_range(10, 50) << "\n";  // 3

    std::cout << "\n=== Perfect Square Check ===\n";
    for (int i = 0; i <= 25; ++i) {
        if (is_perfect_square(i)) {
            std::cout << i << " is a perfect square\n";
        }
    }

    return 0;
}
```

**Expected output:**

```text
=== Binary Search at Compile Time ===
contains(5):  1
contains(10): 0
index_of(23): 5
index_of(99): -1
count_in_range(10,50): 3

=== Perfect Square Check ===
0 is a perfect square
1 is a perfect square
4 is a perfect square
9 is a perfect square
16 is a perfect square
25 is a perfect square
```

---

## Notes

- C++20 made `std::vector`, `std::string`, and most `<algorithm>` functions `constexpr`.
- **Transient allocation rule:** All heap allocations during constant evaluation must be freed before the evaluation ends.
- Pattern: use `std::vector`/`std::string` internally in `consteval` functions, return `std::array` or scalar as the persistent result.
- `std::sort`, `std::lower_bound`, `std::accumulate`, `std::transform`, `std::unique`, etc. all work at compile time.
- `std::array` has been `constexpr`-friendly since C++14 - it's the preferred container for compile-time results.
- In C++23, even more containers and algorithms become `constexpr`.
