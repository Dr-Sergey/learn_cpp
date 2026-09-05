# Use std::copy_if, std::remove_if, and std::count_if with predicates

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/copy>  

---

## Topic Overview

These three algorithms take a **predicate** (a callable returning bool) and operate on ranges. They are the bread-and-butter tools for filtering data without writing explicit loops. Each one does a distinct job: count matching elements, selectively copy them, or rearrange the range to logically remove them.

| Algorithm | What it does | Modifies source? | Returns |
| --- | --- | --- | --- |
| `copy_if` | Copies elements satisfying predicate to output | No | Output iterator past last copied |
| `remove_if` | Moves non-matching elements to front | Yes (reorders) | Iterator to new logical end |
| `count_if` | Counts elements satisfying predicate | No | `difference_type` count |

### Signatures

```cpp
// Copy matching elements to destination
auto out = std::copy_if(first, last, dest, pred);

// "Remove" matching elements (needs .erase() afterward)
auto new_end = std::remove_if(first, last, pred);
container.erase(new_end, container.end());

// Count matching elements
auto n = std::count_if(first, last, pred);
```

All predicates must be **pure functions** - they should not modify their arguments or produce side effects, especially when used with parallel execution policies where the predicate may be called concurrently from multiple threads.

---

## Self-Assessment

### Q1: Chain copy_if and transform to filter and map a range into an output container

The two-step approach here - filter first with `copy_if`, then transform - is a common pattern. C++20 ranges let you compose these as a single lazy pipeline, but the explicit two-step version is clear and works in C++11/14/17.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <iterator>
#include <string>

int main() {
    std::vector<int> input = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // === copy_if: filter even numbers ===
    std::vector<int> evens;
    std::copy_if(input.begin(), input.end(), std::back_inserter(evens),
                 [](int x) { return x % 2 == 0; });

    std::cout << "Evens: ";
    for (int x : evens) std::cout << x << " ";
    std::cout << "\n";
    // Evens: 2 4 6 8 10

    // === Chain: filter then transform ===
    // Get squares of numbers > 5
    std::vector<int> filtered;
    std::copy_if(input.begin(), input.end(), std::back_inserter(filtered),
                 [](int x) { return x > 5; });

    std::vector<int> squares;
    std::transform(filtered.begin(), filtered.end(), std::back_inserter(squares),
                   [](int x) { return x * x; });

    std::cout << "Squares of >5: ";
    for (int x : squares) std::cout << x << " ";
    std::cout << "\n";
    // Squares of >5: 36 49 64 81 100

    // === One-pass alternative: transform + back_inserter with condition in transform ===
    // (Not clean — better to use ranges in C++20)

    // === C++20 ranges: filter | transform in a single pipeline ===
    // auto result = input
    //     | std::views::filter([](int x) { return x > 5; })
    //     | std::views::transform([](int x) { return x * x; });

    // === Practical: extract names of passing students ===
    struct Student {
        std::string name;
        int grade;
    };

    std::vector<Student> students = {
        {"Alice", 92}, {"Bob", 45}, {"Charlie", 78}, {"Dave", 33}, {"Eve", 88}
    };

    std::vector<std::string> passing_names;
    // Step 1: filter
    std::vector<Student> passing;
    std::copy_if(students.begin(), students.end(), std::back_inserter(passing),
                 [](const Student& s) { return s.grade >= 60; });
    // Step 2: transform
    std::transform(passing.begin(), passing.end(), std::back_inserter(passing_names),
                   [](const Student& s) { return s.name; });

    std::cout << "Passing: ";
    for (const auto& name : passing_names) std::cout << name << " ";
    std::cout << "\n";
    // Passing: Alice Charlie Eve

    return 0;
}
```

`std::back_inserter` is the glue that lets you use `copy_if` and `transform` without pre-sizing the destination vector. It creates an output iterator that calls `push_back` for each element written, so the destination grows automatically.

### Q2: Explain why remove_if does not reduce container size and what you must do afterward

This is the same conceptual split that applies to `std::remove` and `std::unique`: the algorithm knows about iterators, but only the container knows about its own memory. They cannot do each other's jobs.

What `remove_if` actually does:

1. Moves elements that do NOT match the predicate to the front of the range.
2. Returns an iterator to the new logical end.
3. Elements from the returned iterator to the original end are in a **moved-from** (valid but unspecified) state.
4. The container's `size()` is unchanged.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6};
    std::cout << "Before: size=" << v.size() << " -> ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Before: size=6 -> 1 2 3 4 5 6

    // Remove odds
    auto new_end = std::remove_if(v.begin(), v.end(),
                                  [](int x) { return x % 2 != 0; });

    std::cout << "After remove_if: size=" << v.size() << " -> ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // After remove_if: size=6 -> 2 4 6 4 5 6  (tail is garbage!)

    // MUST erase the tail!
    v.erase(new_end, v.end());

    std::cout << "After erase: size=" << v.size() << " -> ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // After erase: size=3 -> 2 4 6

    // === C++20 alternative: std::erase_if does both steps ===
    std::vector<int> v2 = {1, 2, 3, 4, 5, 6};
    std::erase_if(v2, [](int x) { return x % 2 != 0; });
    // v2 = {2, 4, 6}, size = 3. Done!

    return 0;
}
```

The tail after `remove_if` contains moved-from values - they are valid objects in an unspecified state, not zeroes or garbage bytes, but you cannot predict what values they hold. Iterating the full container and treating those tail elements as real data is a silent correctness bug. In C++20, prefer `std::erase_if` which does both steps atomically and eliminates this whole class of mistakes.

### Q3: Use count_if to compute a histogram of values satisfying a predicate

`count_if` is simple to understand but very versatile. It can drive a histogram, a quality report, a validation summary - anywhere you need counts broken down by condition.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>
#include <map>

int main() {
    // === Basic count_if ===
    std::vector<int> data = {1, -2, 3, -4, 5, -6, 7, 8, -9, 10};

    auto pos_count = std::count_if(data.begin(), data.end(),
                                   [](int x) { return x > 0; });
    auto neg_count = std::count_if(data.begin(), data.end(),
                                   [](int x) { return x < 0; });
    auto zero_count = std::count_if(data.begin(), data.end(),
                                    [](int x) { return x == 0; });

    std::cout << "Positive: " << pos_count << ", Negative: " << neg_count
              << ", Zero: " << zero_count << "\n";
    // Positive: 6, Negative: 4, Zero: 0

    // === Histogram: grade distribution ===
    std::vector<int> grades = {95, 82, 67, 91, 74, 55, 88, 43, 76, 99, 61, 70, 85};

    std::map<std::string, long> histogram;
    histogram["A (90-100)"] = std::count_if(grades.begin(), grades.end(),
        [](int g) { return g >= 90; });
    histogram["B (80-89)"]  = std::count_if(grades.begin(), grades.end(),
        [](int g) { return g >= 80 && g < 90; });
    histogram["C (70-79)"]  = std::count_if(grades.begin(), grades.end(),
        [](int g) { return g >= 70 && g < 80; });
    histogram["D (60-69)"]  = std::count_if(grades.begin(), grades.end(),
        [](int g) { return g >= 60 && g < 70; });
    histogram["F (<60)"]    = std::count_if(grades.begin(), grades.end(),
        [](int g) { return g < 60; });

    std::cout << "\nGrade Distribution:\n";
    for (const auto& [label, count] : histogram) {
        std::cout << "  " << label << ": " << count
                  << " " << std::string(count, '*') << "\n";
    }
    // A (90-100): 3 ***
    // B (80-89):  3 ***
    // C (70-79):  3 ***
    // D (60-69):  2 **
    // F (<60):    2 **

    // === Character frequency analysis ===
    std::string text = "Hello, World! This is a C++ example.";
    auto vowels = std::count_if(text.begin(), text.end(),
        [](char c) {
            c = std::tolower(c);
            return c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u';
        });
    auto consonants = std::count_if(text.begin(), text.end(),
        [](char c) { return std::isalpha(c) && !std::strchr("aeiouAEIOU", c); });
    auto digits = std::count_if(text.begin(), text.end(),
        [](char c) { return std::isdigit(c); });

    std::cout << "\nText analysis: vowels=" << vowels
              << " consonants=" << consonants
              << " digits=" << digits << "\n";

    return 0;
}
```

Each `count_if` call is a single clean statement that reads like a natural-language question: "how many grades are >= 90?". Compare that to a manual loop with a counter variable - the algorithm version is shorter, has no off-by-one risk, and the intent is immediately obvious.

---

## Notes

- **Predicate must be a pure function:** `count_if`, `copy_if`, and `remove_if` may call the predicate in any order. With parallel policies, it is called concurrently.
- **`copy_if` destination must have enough space:** Use `std::back_inserter` for automatic growth, or pre-allocate with `reserve()`.
- **`remove_if` on non-vector containers:** For `std::list`, use `lst.remove_if(pred)` - the member function actually erases. For `std::set`/`std::map`, use `std::erase_if` (C++20).
- **Efficiency:** `count_if` and `copy_if` are single-pass (O(n)). `remove_if` is O(n) but involves moves. All are stable (preserve relative order of kept elements).
- **C++20 ranges:** `std::ranges::copy_if`, `std::ranges::count_if` support projections for cleaner syntax.
