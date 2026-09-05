# Use std::iota to fill a range with incrementing values

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/iota>  

---

## Topic Overview

`std::iota` (from `<numeric>`) assigns sequentially increasing values to a range, starting from a given value and applying `operator++` for each subsequent element. It is the idiomatic replacement for a manual fill loop whenever you want a sequence like `0, 1, 2, 3, ...`.

```cpp
#include <numeric>
std::vector<int> v(5);
std::iota(v.begin(), v.end(), 0);  // v = {0, 1, 2, 3, 4}
```

### iota vs Manual Loop

All three approaches below produce the same sequence. `std::iota` is preferred for clarity, and `std::views::iota` is preferred in C++20 range pipelines where you do not want to allocate a container at all:

```cpp
// Manual loop:
for (int i = 0; i < n; ++i) v[i] = i;

// std::iota (more expressive):
std::iota(v.begin(), v.end(), 0);

// C++20 views::iota (lazy, no allocation):
for (int i : std::views::iota(0, n)) { ... }
```

---

## Self-Assessment

### Q1: Fill a `vector<int>` with 0..N-1 using std::iota as an alternative to a manual loop

This example shows the basic usage side by side with the manual loop to make the equivalence concrete. Notice that you can start from any value - `iota` just increments from wherever you tell it to start.

```cpp
#include <iostream>
#include <vector>
#include <numeric>

int main() {
    constexpr int N = 10;

    // === Manual loop ===
    std::vector<int> manual(N);
    for (int i = 0; i < N; ++i) manual[i] = i;

    // === std::iota ===
    std::vector<int> iota_vec(N);
    std::iota(iota_vec.begin(), iota_vec.end(), 0);

    std::cout << "Manual: ";
    for (int x : manual) std::cout << x << " ";
    std::cout << "\n";
    // 0 1 2 3 4 5 6 7 8 9

    std::cout << "iota:   ";
    for (int x : iota_vec) std::cout << x << " ";
    std::cout << "\n";
    // 0 1 2 3 4 5 6 7 8 9

    // === Starting from a different value ===
    std::vector<int> from_10(5);
    std::iota(from_10.begin(), from_10.end(), 10);
    std::cout << "From 10: ";
    for (int x : from_10) std::cout << x << " ";
    std::cout << "\n";
    // 10 11 12 13 14

    // === Works on arrays too ===
    int arr[5];
    std::iota(std::begin(arr), std::end(arr), 100);
    std::cout << "Array: ";
    for (int x : arr) std::cout << x << " ";
    std::cout << "\n";
    // 100 101 102 103 104

    // === Fill with floating point ===
    std::vector<double> dv(5);
    std::iota(dv.begin(), dv.end(), 0.5);
    std::cout << "Doubles: ";
    for (double x : dv) std::cout << x << " ";
    std::cout << "\n";
    // 0.5 1.5 2.5 3.5 4.5

    return 0;
}
```

The floating-point case is a little surprising at first - it works because `operator++` on a `double` simply adds 1.0.

### Q2: Use std::iota with a custom iterator to generate a range of enum values

`std::iota` calls `operator++` on the value type, so any type with a prefix increment operator works - including enums, as long as you define `operator++` for them. This is a clean way to initialize a container with every value in an enum without writing a loop.

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <string_view>

// === Enum with operator++ for use with iota ===
enum class Color : int { Red = 0, Green, Blue, Yellow, Cyan, Magenta, COUNT };

// Enable operator++ for the enum
Color& operator++(Color& c) {
    c = static_cast<Color>(static_cast<int>(c) + 1);
    return c;
}

std::string_view color_name(Color c) {
    switch (c) {
        case Color::Red:     return "Red";
        case Color::Green:   return "Green";
        case Color::Blue:    return "Blue";
        case Color::Yellow:  return "Yellow";
        case Color::Cyan:    return "Cyan";
        case Color::Magenta: return "Magenta";
        default:             return "Unknown";
    }
}

int main() {
    // Fill vector with all enum values using iota
    constexpr int count = static_cast<int>(Color::COUNT);
    std::vector<Color> all_colors(count);
    std::iota(all_colors.begin(), all_colors.end(), Color::Red);

    std::cout << "All colors:\n";
    for (Color c : all_colors) {
        std::cout << "  " << static_cast<int>(c) << ": " << color_name(c) << "\n";
    }
    // 0: Red
    // 1: Green
    // 2: Blue
    // 3: Yellow
    // 4: Cyan
    // 5: Magenta

    // === Alternative: fill with just a subset ===
    std::vector<Color> first_three(3);
    std::iota(first_three.begin(), first_three.end(), Color::Red);

    std::cout << "First three: ";
    for (Color c : first_three) std::cout << color_name(c) << " ";
    std::cout << "\n";
    // Red Green Blue

    return 0;
}
```

The key thing to define is prefix `operator++` that casts through `int`. Without that, the compiler will reject the `iota` call because the built-in `++` on a scoped enum is not defined.

### Q3: Combine iota and shuffle to generate a random permutation of indices

The `iota` + `shuffle` combination is the standard C++ idiom for generating a random permutation. Fill the range with known values first, then scramble it with a proper random number generator. The quiz-ordering example below is typical of how this shows up in real applications.

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <algorithm>
#include <random>
#include <string>

int main() {
    // === Generate random permutation of [0..N-1] ===
    constexpr int N = 8;
    std::vector<int> indices(N);
    std::iota(indices.begin(), indices.end(), 0);

    std::mt19937 rng(12345);
    std::shuffle(indices.begin(), indices.end(), rng);

    std::cout << "Random permutation: ";
    for (int x : indices) std::cout << x << " ";
    std::cout << "\n";
    // e.g.: 3 7 0 5 1 6 4 2

    // === Practical: randomize quiz question order ===
    std::vector<std::string> questions = {
        "What is RAII?",
        "Explain move semantics",
        "What is ADL?",
        "Define SFINAE",
        "What is std::optional?"
    };

    std::vector<int> order(questions.size());
    std::iota(order.begin(), order.end(), 0);
    std::shuffle(order.begin(), order.end(), rng);

    std::cout << "\nQuiz (randomized):\n";
    for (int i = 0; i < static_cast<int>(order.size()); ++i) {
        std::cout << "  Q" << (i + 1) << ": " << questions[order[i]] << "\n";
    }

    // === Fisher-Yates shuffle is what std::shuffle does internally ===
    // iota + shuffle is the standard idiom for random permutations

    return 0;
}
```

Notice that the original `questions` vector is never touched. The `order` array holds the permuted indices, and you just use them to index into `questions`. This separation of "what order" from "the data itself" is exactly the indirect-sort pattern from Q1 applied to randomization.

---

## Notes

- **Header:** `std::iota` is in `<numeric>`, not `<algorithm>`.
- **Complexity:** Exactly `last - first` increments and assignments.
- **Requirement:** The value type must support copy assignment and prefix `operator++`.
- **C++20 `std::views::iota`:** Generates values lazily (no container needed). `std::views::iota(0, 10)` produces 0..9 without allocation.
- **C++23 `std::ranges::iota`:** Eager version with range interface.
- **Name origin:** From APL's iota (ι) function for index generation.
