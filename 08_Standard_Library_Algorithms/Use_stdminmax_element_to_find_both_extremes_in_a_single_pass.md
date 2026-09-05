# Use std::minmax_element to find both extremes in a single pass

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/minmax_element>  

---

## Topic Overview

`std::minmax_element` returns a pair of iterators to the smallest and largest elements in a single pass, using only ~3n/2 comparisons instead of the 2n required by separate `min_element` + `max_element` calls. When you need both extremes, this is always the right choice.

```cpp
#include <algorithm>
auto [min_it, max_it] = std::minmax_element(first, last);
auto [min_it, max_it] = std::minmax_element(first, last, comp);
```

### Comparison Count

| Approach | Comparisons | Passes |
| --- | --- | --- |
| `min_element` + `max_element` | 2(n-1) | 2 |
| `minmax_element` | <=3⌊n/2⌋ | 1 |

The algorithm works by comparing pairs of elements: first comparing the pair against each other, then comparing the smaller with the current min and the larger with the current max. This gives 3 comparisons per 2 elements, which is provably optimal for finding both extremes simultaneously.

---

## Self-Assessment

### Q1: Replace two separate min_element/max_element calls with one minmax_element call

The old way - calling `min_element` and then `max_element` - scans the data twice and performs 2(n-1) comparisons. The single `minmax_element` call does the same job in one pass with fewer comparisons. The structured binding `auto [lo, hi] = ...` makes the result easy to unpack.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    std::vector<int> data = {7, 2, 9, 1, 5, 8, 3, 6, 4};

    // === Old way: two passes ===
    auto min_it = std::min_element(data.begin(), data.end());
    auto max_it = std::max_element(data.begin(), data.end());
    std::cout << "Two-pass: min=" << *min_it << " max=" << *max_it << "\n";
    // min=1 max=9

    // === New way: single pass ===
    auto [lo, hi] = std::minmax_element(data.begin(), data.end());
    std::cout << "One-pass: min=" << *lo << " at index " << (lo - data.begin())
              << ", max=" << *hi << " at index " << (hi - data.begin()) << "\n";
    // One-pass: min=1 at index 3, max=9 at index 2

    // === With custom comparator ===
    std::vector<std::string> words = {"banana", "apple", "cherry", "date", "elderberry"};
    auto [shortest, longest] = std::minmax_element(
        words.begin(), words.end(),
        [](const std::string& a, const std::string& b) {
            return a.size() < b.size();
        });
    std::cout << "Shortest: \"" << *shortest << "\" (" << shortest->size() << " chars)\n";
    std::cout << "Longest:  \"" << *longest << "\" (" << longest->size() << " chars)\n";
    // Shortest: "date" (4 chars)
    // Longest:  "elderberry" (10 chars)

    // === On structs with projection (C++20 ranges) ===
    // auto [youngest, oldest] = std::ranges::minmax_element(people, {}, &Person::age);

    return 0;
}
```

Notice that `minmax_element` returns iterators, not values. You dereference them with `*lo` and `*hi` to get the actual values, and you can compute the index by subtracting `data.begin()` as shown above.

### Q2: Show the comparison count: minmax_element does 3n/2 comparisons vs 2n for separate calls

This example uses a counting comparator to measure the actual number of comparisons both approaches make. The ratio converges toward 0.75 as n grows - exactly the 3/4 you would predict from the 3n/2 vs 2n formula.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>

// Counting comparator
struct CountingLess {
    int& count;
    bool operator()(int a, int b) { ++count; return a < b; }
};

int main() {
    for (int n : {10, 20, 50, 100, 1000}) {
        std::vector<int> data(n);
        std::iota(data.begin(), data.end(), 0);

        // === Count comparisons for separate min + max ===
        int count_separate = 0;
        CountingLess cmp_sep{count_separate};
        std::min_element(data.begin(), data.end(), cmp_sep);
        std::max_element(data.begin(), data.end(), cmp_sep);

        // === Count comparisons for minmax_element ===
        int count_combined = 0;
        CountingLess cmp_comb{count_combined};
        std::minmax_element(data.begin(), data.end(), cmp_comb);

        double ratio = static_cast<double>(count_combined) / count_separate;

        std::cout << "n=" << n
                  << "  separate=" << count_separate
                  << "  minmax=" << count_combined
                  << "  ratio=" << ratio
                  << "  savings=" << (1.0 - ratio) * 100 << "%\n";
    }
    // Expected output (approximately):
    // n=10   separate=18  minmax=13  ratio=0.72  savings=28%
    // n=20   separate=38  minmax=28  ratio=0.74  savings=26%
    // n=50   separate=98  minmax=73  ratio=0.74  savings=26%
    // n=100  separate=198 minmax=148 ratio=0.75  savings=25%
    // n=1000 separate=1998 minmax=1498 ratio=0.75  savings=25%
    //
    // The ratio converges to 3/4 (= 3n/2 / 2n) -> 25% fewer comparisons

    return 0;
}
```

Here is how the algorithm achieves 3n/2 comparisons:

1. Process elements in pairs.
2. Compare the pair against each other (1 comparison).
3. Compare the smaller of the pair with current minimum (1 comparison).
4. Compare the larger of the pair with current maximum (1 comparison).
5. Total: 3 comparisons per 2 elements = 3n/2.

This is actually the theoretically optimal number of comparisons for this problem - you cannot find both min and max using fewer comparisons in the worst case.

### Q3: Use minmax_element to implement auto-scaling for a chart's y-axis

A practical application: when rendering a chart, you need to know the data range to set appropriate axis bounds. `minmax_element` gives you both endpoints in one pass, which is then used to calculate nice round tick intervals.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <cmath>
#include <string>
#include <iomanip>

struct ChartAxis {
    double min_val;
    double max_val;
    double tick_interval;
    int num_ticks;
};

// Auto-scale y-axis to nice round numbers
ChartAxis auto_scale_axis(const std::vector<double>& data) {
    if (data.empty()) return {0, 1, 0.2, 5};

    auto [min_it, max_it] = std::minmax_element(data.begin(), data.end());
    double data_min = *min_it;
    double data_max = *max_it;

    // Add 10% padding
    double range = data_max - data_min;
    if (range == 0) range = 1.0;
    double padding = range * 0.1;
    double axis_min = data_min - padding;
    double axis_max = data_max + padding;

    // Round to nice tick intervals
    double raw_interval = (axis_max - axis_min) / 5;
    double mag = std::pow(10, std::floor(std::log10(raw_interval)));
    double normalized = raw_interval / mag;

    double nice_interval;
    if (normalized <= 1.5)      nice_interval = 1.0 * mag;
    else if (normalized <= 3.5) nice_interval = 2.0 * mag;
    else if (normalized <= 7.5) nice_interval = 5.0 * mag;
    else                        nice_interval = 10.0 * mag;

    axis_min = std::floor(axis_min / nice_interval) * nice_interval;
    axis_max = std::ceil(axis_max / nice_interval) * nice_interval;

    return {axis_min, axis_max, nice_interval,
            static_cast<int>((axis_max - axis_min) / nice_interval)};
}

int main() {
    std::vector<double> temperatures = {18.3, 22.1, 19.7, 25.8, 21.4, 15.9, 24.6, 20.0};

    auto axis = auto_scale_axis(temperatures);

    std::cout << "Data range: [" << *std::min_element(temperatures.begin(), temperatures.end())
              << ", " << *std::max_element(temperatures.begin(), temperatures.end()) << "]\n";
    std::cout << "Axis range: [" << axis.min_val << ", " << axis.max_val << "]\n";
    std::cout << "Tick interval: " << axis.tick_interval << "\n";
    std::cout << "Ticks: ";
    for (int i = 0; i <= axis.num_ticks; ++i) {
        std::cout << std::fixed << std::setprecision(0)
                  << axis.min_val + i * axis.tick_interval << " ";
    }
    std::cout << "\n";

    // Simple ASCII chart
    auto [lo, hi] = std::minmax_element(temperatures.begin(), temperatures.end());
    double lo_val = *lo, hi_val = *hi;
    constexpr int width = 40;

    std::cout << "\nTemperature chart:\n";
    for (size_t i = 0; i < temperatures.size(); ++i) {
        int bar_len = static_cast<int>(
            (temperatures[i] - lo_val) / (hi_val - lo_val) * width);
        std::cout << "Day " << (i + 1) << " " << std::fixed << std::setprecision(1)
                  << temperatures[i] << " |" << std::string(bar_len, '#') << "\n";
    }

    return 0;
}
```

The ASCII bar chart at the end uses `minmax_element` a second time to normalize each bar length as a fraction of the full data range. Both `lo_val` and `hi_val` are needed to compute that normalization - exactly the use case this algorithm is designed for.

---

## Notes

- **Empty ranges:** Undefined behavior if the range is empty. Check with `if (!v.empty())` first.
- **Equal elements:** If multiple elements tie for min or max, `minmax_element` returns the first min and the **last** max. This differs from separate calls to `min_element` (first) and `max_element` (first).
- **`std::minmax`** (without `_element`) works on values, not iterators: `auto [lo, hi] = std::minmax(a, b)`.
- **C++20 ranges:** `std::ranges::minmax_element(v)` - no begin/end. Supports projections.
- **`std::minmax({...})`** works with initializer lists: `auto [lo, hi] = std::minmax({3, 1, 4, 1, 5})`.
- **Performance:** The 3n/2 algorithm matters for expensive comparisons (e.g., string comparison). For integers, the cache benefit of a single pass also helps.
