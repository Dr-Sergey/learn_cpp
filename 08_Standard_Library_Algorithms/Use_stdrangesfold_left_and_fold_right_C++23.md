# Use std::ranges::fold_left and fold_right (C++23)

**Category:** Standard Library Algorithms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges/fold_left>  

---

## Topic Overview

`std::ranges::fold_left` and `fold_right` (C++23) are the ranges-based equivalents of functional programming's fold/reduce operations. They replace `std::accumulate` and `std::reduce` with a cleaner, more expressive API that works directly with ranges.

### fold_left vs fold_right

The difference is all about associativity - which elements get combined first:

```cpp
fold_left(range, init, op):    ((((init op a) op b) op c) op d)
fold_right(range, init, op):   (a op (b op (c op (d op init))))
```

For commutative, associative operations like addition or multiplication, the result is the same either way. For non-commutative operations like subtraction or string concatenation, the direction matters a lot.

### Family of Fold Functions (C++23)

C++23 provides a whole family so you can handle edge cases like empty ranges cleanly:

| Function | Init value | Direction | Notes |
| --- | --- | --- | --- |
| `fold_left(range, init, op)` | Required | Left -> right | Like `accumulate` |
| `fold_right(range, init, op)` | Required | Right -> left | Reversed |
| `fold_left_first(range, op)` | Uses first elem | Left -> right | Returns `optional` |
| `fold_right_last(range, op)` | Uses last elem | Right -> left | Returns `optional` |
| `fold_left_with_iter(range, init, op)` | Required | Left -> right | Also returns end iter |
| `fold_left_first_with_iter(range, op)` | Uses first elem | Left -> right | Returns iter + optional |

### vs std::accumulate / std::reduce

Here's how the three main options compare:

| Feature | `accumulate` | `reduce` | `fold_left` |
| --- | --- | --- | --- |
| Header | `<numeric>` | `<numeric>` | `<algorithm>` |
| Takes range | No (iterators) | No | Yes |
| Parallelizable | No | Yes | No |
| Return type | Same as init | Same as init | Deduced |
| Init required | Yes | Optional | Yes (`fold_left_first` = No) |

---

## Self-Assessment

### Q1: Compute the sum of a range using std::ranges::fold_left and compare with std::accumulate

The key practical advantage of `fold_left` over `accumulate` is return type deduction. With `accumulate`, the result type is forced to match the `init` value's type - passing `0` (int) when your data is `double` silently truncates. `fold_left` deduces from the operation itself:

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <algorithm>
#include <string>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5};

    // === std::accumulate (pre-C++23) ===
    int sum_acc = std::accumulate(nums.begin(), nums.end(), 0);
    std::cout << "accumulate: " << sum_acc << "\n";  // 15

    // === std::ranges::fold_left (C++23) ===
    int sum_fold = std::ranges::fold_left(nums, 0, std::plus<>{});
    std::cout << "fold_left:  " << sum_fold << "\n";  // 15

    // === Key advantage: works directly with ranges ===
    // No need for begin()/end()
    auto product = std::ranges::fold_left(nums, 1, std::multiplies<>{});
    std::cout << "product:    " << product << "\n";  // 120

    // === Return type is deduced (not forced to match init) ===
    // accumulate with int init truncates!
    std::vector<double> prices = {1.5, 2.3, 4.7};
    int bad_sum = std::accumulate(prices.begin(), prices.end(), 0);  // BUG: int init!
    double good_sum = std::ranges::fold_left(prices, 0.0, std::plus<>{});
    std::cout << "\naccumulate(int init): " << bad_sum << "\n";   // 7 (truncated!)
    std::cout << "fold_left(0.0):       " << good_sum << "\n";   // 8.5

    // === String concatenation ===
    std::vector<std::string> words = {"Hello", " ", "World", "!"};
    auto sentence = std::ranges::fold_left(words, std::string{}, std::plus<>{});
    std::cout << "\nConcatenated: " << sentence << "\n";  // Hello World!

    // === With lambda ===
    auto max_val = std::ranges::fold_left(nums, INT_MIN,
        [](int acc, int x) { return std::max(acc, x); });
    std::cout << "Max: " << max_val << "\n";  // 5

    return 0;
}
```

The `accumulate` with `int` init truncating `double` data is a real bug that has appeared in production code. `fold_left` with `0.0` sidesteps it entirely because the operation `0.0 + 1.5` produces a `double`.

### Q2: Explain why fold_right processes elements in reverse and when that matters

For subtraction, the direction completely changes the result. This example makes that concrete:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>

int main() {
    std::vector<int> nums = {1, 2, 3, 4};

    // === fold_left: left-associative ===
    // ((((0 - 1) - 2) - 3) - 4) = -10
    int left = std::ranges::fold_left(nums, 0, std::minus<>{});
    std::cout << "fold_left(minus):  " << left << "\n";  // -10

    // === fold_right: right-associative ===
    // (1 - (2 - (3 - (4 - 0)))) = -2
    int right = std::ranges::fold_right(nums, 0, std::minus<>{});
    std::cout << "fold_right(minus): " << right << "\n";  // -2

    // For commutative+associative ops (like +, *), direction doesn't matter
    int left_sum = std::ranges::fold_left(nums, 0, std::plus<>{});
    int right_sum = std::ranges::fold_right(nums, 0, std::plus<>{});
    std::cout << "\nfold_left(+):  " << left_sum << "\n";   // 10
    std::cout << "fold_right(+): " << right_sum << "\n";    // 10

    // === When fold_right matters: building right-associative structures ===
    // Example: building a nested string like "1-(2-(3-(4-0)))"
    auto build_expr = [](int x, std::string acc) -> std::string {
        return std::to_string(x) + "-(" + acc + ")";
    };

    auto expr = std::ranges::fold_right(nums, std::string("0"), build_expr);
    std::cout << "\nExpression: " << expr << "\n";
    // 1-(2-(3-(4-(0))))

    // === Practical: building a linked-list-style structure from right ===
    // fold_right naturally builds right-recursive structures:
    //   cons(1, cons(2, cons(3, nil)))  <- fold_right
    //   snoc(snoc(snoc(nil, 1), 2), 3) <- fold_left

    // === fold_right NOTE on argument order ===
    // fold_left:  op(accumulator, element)
    // fold_right: op(element, accumulator)  <- reversed!

    return 0;
}
```

The argument order reversal is the reason this trips people up. In `fold_left`, the accumulator is the first argument to `op`. In `fold_right`, the *element* comes first and the accumulator is second. When you write a lambda for `fold_right`, you need to flip the parameter order.

### Q3: Use fold_left_first to fold without providing an initial value

`fold_left_first` is useful when there's no natural identity value for your operation, or when you want the first element to be the seed. The trade-off is that it returns `std::optional` to safely handle the empty-range case:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <optional>
#include <string>
#include <functional>
#include <ranges>
#include <utility>

int main() {
    std::vector<int> nums = {3, 1, 4, 1, 5, 9};

    // === fold_left_first: uses first element as init ===
    // Returns std::optional because the range might be empty
    auto max_opt = std::ranges::fold_left_first(nums,
        [](int a, int b) { return std::max(a, b); });

    if (max_opt) {
        std::cout << "Max: " << *max_opt << "\n";  // 9
    }

    // Equivalent to fold_left(tail, head, op) but handles empty ranges

    // === Empty range returns nullopt ===
    std::vector<int> empty;
    auto result = std::ranges::fold_left_first(empty, std::plus<>{});
    std::cout << "Empty has value: " << std::boolalpha
              << result.has_value() << "\n";  // false

    // === fold_left_with_iter: returns both result and end iterator ===
    auto [iter, sum] = std::ranges::fold_left_with_iter(nums, 0, std::plus<>{});
    std::cout << "\nSum: " << sum << "\n";                              // 23
    std::cout << "Iterator at end: " << (iter == nums.end()) << "\n";  // true

    // === Practical: finding min and max in a single pass ===
    auto minmax = std::ranges::fold_left_first(nums,
        [](std::pair<int,int> acc, int x) -> std::pair<int,int> {
            return {std::min(acc.first, x), std::max(acc.second, x)};
        });
    // Note: fold_left_first deduces from first element,
    // so we need the first op call to produce a pair
    // Better approach for minmax: use ranges::minmax directly

    // === fold_left_first for string joining ===
    std::vector<std::string> words = {"one", "two", "three"};
    auto joined = std::ranges::fold_left_first(words,
        [](std::string a, const std::string& b) { return a + ", " + b; });
    std::cout << "Joined: " << joined.value_or("(empty)") << "\n";
    // Joined: one, two, three

    return 0;
}
```

The string joining example is a clean real-world use: you don't need a separate empty-string seed because the first word naturally seeds the accumulator. And `.value_or("(empty)")` handles the empty-range case without an `if` check.

---

## Notes

- **C++23 required.** Compiler support: GCC 13+, Clang 17+, MSVC 19.37+.
- `fold_left` is in `<algorithm>`, **not** `<numeric>` (unlike `accumulate`/`reduce`).
- **Return type deduction:** `fold_left` deduces the return type from `op(init, elem)`, avoiding the classic `accumulate` trap where `int` init truncates `double` data.
- **`fold_left_first`** returns `std::optional` to safely handle empty ranges - no undefined behavior.
- **`fold_right`** note: `op(element, accumulator)` - the argument order is reversed compared to `fold_left`'s `op(accumulator, element)`.
- For parallelizable folds, use `std::reduce` or `std::transform_reduce` instead - fold functions are inherently sequential.
