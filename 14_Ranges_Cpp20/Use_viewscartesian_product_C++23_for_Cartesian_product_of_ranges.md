# Use views::cartesian_product (C++23) for Cartesian product of ranges

**Category:** Ranges Cpp20  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/cartesian_product_view>  

---

## Topic Overview

`views::cartesian_product` generates all **combinations** of elements from two or more ranges, yielding tuples. For ranges `R1 = {a,b}` and `R2 = {1,2,3}`, it produces `{(a,1), (a,2), (a,3), (b,1), (b,2), (b,3)}`.

Think of it as those nested `for` loops you always end up writing when you need every pairing of two (or more) sets - except here it's lazy, composable, and expressed in a single line.

### How It Works

Here is the layout of how those combinations are arranged:

```cpp
R1 = { A, B }       R2 = { 1, 2, 3 }

cartesian_product(R1, R2):
  (A, 1)  (A, 2)  (A, 3)    <- R1[0] x all of R2
  (B, 1)  (B, 2)  (B, 3)    <- R1[1] x all of R2

Total: |R1| x |R2| = 2 x 3 = 6 tuples
```

The **last range varies fastest** (like nested for-loops where the outermost loop is the first range). So R1 steps slowly while R2 cycles through all its values for each R1 element.

### Signature

The call is straightforward - pass two or more ranges and you get back a range of tuples:

```cpp
views::cartesian_product(R1, R2, ...Rn);
// Returns a range of tuple<range_reference_t<R1>, range_reference_t<R2>, ...>
```

### Range Category

The resulting range category depends on what you feed in. If the table feels like a lot, the short version is: your result is at most as capable as your weakest input range.

| Input ranges | Resulting category |
| --- | --- |
| All random-access + sized | **random-access** + sized |
| All bidirectional (+ common) | **bidirectional** |
| All forward | **forward** |
| Any input-only | **input** |

The first range only needs to be `forward_range`; subsequent ranges must be `forward_range` at minimum.

### Key Properties

- **Lazy**: tuples are generated on demand, no intermediate storage.
- **Multi-dimensional**: supports 2, 3, or more ranges.
- Produces `std::tuple` of references to elements.

---

## Self-Assessment

### Q1: Generate all (x, y) pairs for two ranges using `views::cartesian_product`

Here you can see the basic pattern: two ranges go in, every combination comes out. Notice how structured bindings make destructuring each tuple painless.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    std::vector<int> xs = {1, 2, 3};
    std::string ys = "AB";

    // Generate all (x, y) pairs
    for (auto [x, y] : std::views::cartesian_product(xs, ys)) {
        std::cout << '(' << x << ',' << y << ") ";
    }
    std::cout << '\n';

    // With three ranges: all (row, col, layer) combinations
    auto rows = std::views::iota(0, 2);
    auto cols = std::views::iota(0, 3);
    auto layers = std::views::iota(0, 2);

    int count = 0;
    for (auto [r, c, l] : std::views::cartesian_product(rows, cols, layers)) {
        ++count;
    }
    std::cout << "3D grid cells: " << count << '\n';
}
// Expected output:
// (1,A) (1,B) (2,A) (2,B) (3,A) (3,B)
// 3D grid cells: 12
```

Notice that `ys` (the last range) varies fastest - so you get `(1,A), (1,B)` before moving on to `(2,A), (2,B)`. The three-range example just multiplies the sizes: 2 x 3 x 2 = 12 combinations, none of them stored at once.

- `cartesian_product(xs, ys)` yields each `(int, char)` combination.
- Structured bindings `auto [x, y]` destructure each tuple.
- The last range (`ys`) varies fastest, so we get `(1,A), (1,B), (2,A), (2,B), ...`.
- With 3 ranges of sizes 2x3x2, we get 12 combinations.

### Q2: Show that `cartesian_product` produces a random-access range when both inputs are random-access

This one is worth understanding deeply because the random-access behavior enables O(1) indexing - the view can jump to element `[7]` without iterating all the previous elements.

```cpp
#include <array>
#include <concepts>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> a = {10, 20, 30};
    std::array<int, 4> b = {1, 2, 3, 4};

    auto product = std::views::cartesian_product(a, b);

    // Both inputs are random-access + sized -> product is random-access + sized
    static_assert(std::ranges::random_access_range<decltype(product)>);
    static_assert(std::ranges::sized_range<decltype(product)>);

    std::cout << "Total combinations: " << std::ranges::size(product) << '\n';

    // Random access: jump to element [7] directly (0-indexed)
    auto it = std::ranges::begin(product);
    it += 7;  // direct jump, O(1)
    auto [x, y] = *it;
    std::cout << "Element [7]: (" << x << ", " << y << ")\n";

    // Verify by counting: (10,1)(10,2)(10,3)(10,4)(20,1)(20,2)(20,3)(20,4)
    //                      [0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]
    // Element [7] = (20, 4)

    // With a filter view (non-random-access), category degrades
    auto filtered_a = a | std::views::filter([](int n) { return n > 10; });
    auto product2 = std::views::cartesian_product(filtered_a, b);
    static_assert(std::ranges::bidirectional_range<decltype(product2)>);
    static_assert(!std::ranges::random_access_range<decltype(product2)>);
    std::cout << "Filtered product is bidirectional, not random-access\n";
}
// Expected output:
// Total combinations: 12
// Element [7]: (20, 4)
// Filtered product is bidirectional, not random-access
```

The reason `it += 7` works in O(1) is that the view can decompose index 7 as `(7 / 4, 7 % 4)` = row 1, column 3, then directly point into `a[1]` and `b[3]`. As soon as you introduce a `filter_view` (which can't tell you its element positions in O(1)), that arithmetic breaks down and the whole product degrades to bidirectional.

- `vector` and `array` are both random-access + sized, so the product inherits random-access.
- `size()` is O(1): it simply multiplies the sizes of each input range.
- Iterator arithmetic (`it += 7`) works in O(1) by computing `(7 / 4, 7 % 4)` = row 1, col 3.
- When a filter view is used (which is only bidirectional), the product downgrades to bidirectional.

### Q3: Use `cartesian_product` to compute the Kronecker product of two small matrices lazily

This example shows `cartesian_product` doing real mathematical work - generating all index pairs for a matrix product without any explicit nested loops.

```cpp
#include <array>
#include <iostream>
#include <ranges>

int main() {
    // Matrix A (2x2) stored as flat array, row-major
    std::array<int, 4> A = {1, 2, 3, 4};   // [[1,2],[3,4]]
    constexpr int A_rows = 2, A_cols = 2;

    // Matrix B (2x2)
    std::array<int, 4> B = {0, 5, 6, 7};   // [[0,5],[6,7]]
    constexpr int B_rows = 2, B_cols = 2;

    // Kronecker product: for each (i,j) in A and (k,l) in B,
    // result[i*B_rows+k][j*B_cols+l] = A[i][j] * B[k][l]

    auto A_indices = std::views::cartesian_product(
        std::views::iota(0, A_rows), std::views::iota(0, A_cols));
    auto B_indices = std::views::cartesian_product(
        std::views::iota(0, B_rows), std::views::iota(0, B_cols));

    // Lazily compute all Kronecker product entries
    constexpr int K_rows = A_rows * B_rows;
    constexpr int K_cols = A_cols * B_cols;

    std::array<int, K_rows * K_cols> K{};

    for (auto [ai, aj] : A_indices) {
        for (auto [bi, bj] : B_indices) {
            int row = ai * B_rows + bi;
            int col = aj * B_cols + bj;
            K[row * K_cols + col] = A[ai * A_cols + aj] * B[bi * B_cols + bj];
        }
    }

    // Print the 4x4 Kronecker product
    std::cout << "Kronecker product (A (x) B):\n";
    for (int r = 0; r < K_rows; ++r) {
        for (int c = 0; c < K_cols; ++c)
            std::cout << K[r * K_cols + c] << '\t';
        std::cout << '\n';
    }
}
// Expected output:
// Kronecker product (A (x) B):
// 0	5	0	10
// 6	7	12	14
// 0	15	0	20
// 18	21	24	28
```

The key idea here is that `cartesian_product(iota(0,2), iota(0,2))` replaces two nested `for(int i=0; i<2; ++i)` loops cleanly. The view is lazy - index pairs are generated as needed, not stored in any intermediate container.

- `cartesian_product(iota(0,2), iota(0,2))` generates all (row, col) index pairs for a 2x2 matrix.
- The nested loop iterates all `(ai,aj)` x `(bi,bj)` combinations to compute each Kronecker entry.
- The view is lazy - index pairs are generated on the fly, not stored in a container.
- This replaces four nested `for(int i=0; i<N; ++i)` loops with more expressive range-based iteration.

---

## Notes

- `cartesian_product` with a single range returns a range of 1-tuples (rarely useful).
- With zero-sized ranges, the product is empty.
- The **first range** is iterated once; subsequent ranges may be iterated multiple times (once per element of the preceding range).
- Equivalent nested loop: `for (auto& x : R1) for (auto& y : R2) ...` but `cartesian_product` can be composed with other views.
- Memory usage is O(1) - only iterators are stored, not elements.
