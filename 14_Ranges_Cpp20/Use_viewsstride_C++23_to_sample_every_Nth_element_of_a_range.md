# Use views::stride (C++23) to sample every Nth element of a range

**Category:** Ranges Cpp20  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/stride_view>  

---

## Topic Overview

`views::stride(n)` advances the iterator by `n` steps on each increment, effectively selecting every Nth element from the source range. It's the cleanest way to downsample a sequence - think of it as "pick one, skip n-1, repeat."

### How stride Works

The diagram below makes the skip pattern concrete. With `stride(3)`, you land on indices 0, 3, 6, 9 - everything in between is silently stepped over.

```cpp
Source:  [0] [1] [2] [3] [4] [5] [6] [7] [8] [9]

stride(3):
  [0]         [3]         [6]         [9]
   ^    skip   ^    skip   ^    skip   ^
```

### Properties

| Property | Value |
| --- | --- |
| Lazy | Yes - no intermediate storage |
| Category | Same as source (preserves random-access, bidirectional, etc.) |
| Sized | If source is sized: ceil(size / n) |
| `stride(1)` | Identity - returns every element |

### Comparison with chunk

These two views are often confused. The distinction is simple: `stride` picks one element per group, while `chunk` gives you the whole group.

| Operation | What it produces |
| --- | --- |
| `stride(3)` | Every 3rd element: `[0, 3, 6, 9]` |
| `chunk(3)` | Groups of 3: `[[0,1,2], [3,4,5], [6,7,8], [9]]` |

`stride` selects one element per group; `chunk` returns entire groups.

### Pre-C++23 Alternative

If you're stuck on C++20, you can approximate `stride` with `filter` and `enumerate`, though it's considerably more verbose and loses iterator category information.

```cpp
// Manual stride with filter + enumerate
auto manual_stride = v
    | views::enumerate
    | views::filter([](auto pair) { return std::get<0>(pair) % 3 == 0; })
    | views::elements<1>;
```

---

## Self-Assessment

### Q1: Downsample a vector of sensor readings using `views::stride(4)` to take every 4th reading

This is a classic signal processing pattern. You have more data than you need and you want to thin it out by picking one sample per group. Here's how `stride` handles that.

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    // Simulated sensor data: 20 readings
    std::vector<double> readings = {
        1.0, 1.1, 1.2, 1.3,   // samples 0-3
        2.0, 2.1, 2.2, 2.3,   // samples 4-7
        3.0, 3.1, 3.2, 3.3,   // samples 8-11
        4.0, 4.1, 4.2, 4.3,   // samples 12-15
        5.0, 5.1, 5.2, 5.3,   // samples 16-19
    };

    // Downsample: take every 4th reading
    auto downsampled = readings | std::views::stride(4);

    std::cout << "Original: " << readings.size() << " readings\n";
    std::cout << "Downsampled (stride 4): ";
    for (double val : downsampled)
        std::cout << val << ' ';
    std::cout << '\n';

    // Different stride values
    for (int s : {1, 2, 5, 10}) {
        auto strided = readings | std::views::stride(s);
        std::cout << "Stride " << s << " (" << std::ranges::distance(strided) << " elements): ";
        for (double v : strided)
            std::cout << v << ' ';
        std::cout << '\n';
    }
}
// Expected output:
// Original: 20 readings
// Downsampled (stride 4): 1 2 3 4 5
// Stride 1 (20 elements): 1 1.1 1.2 1.3 2 2.1 2.2 2.3 3 3.1 3.2 3.3 4 4.1 4.2 4.3 5 5.1 5.2 5.3
// Stride 2 (10 elements): 1 1.2 2 2.2 3 3.2 4 4.2 5 5.2
// Stride 5 (4 elements): 1 2.1 3.2 4.3
// Stride 10 (2 elements): 1 3
```

`stride(4)` selects elements at indices 0, 4, 8, 12, 16 - one per group of 4. The result count is always ceil(N/n), and the view is lazy so it simply advances the iterator by 4 each step rather than building any intermediate collection.

### Q2: Combine `views::stride` with `views::transform` for a decimation filter

This example introduces an important distinction in signal processing: naive decimation (just pick every Nth sample) versus averaging decimation (average each group first, then pick one). `stride` handles the first; combining `chunk` with `transform` handles the second.

```cpp
#include <cmath>
#include <iostream>
#include <numeric>
#include <ranges>
#include <vector>

int main() {
    // Signal: 24 samples of a noisy sine wave
    std::vector<double> signal(24);
    for (int i = 0; i < 24; ++i)
        signal[i] = std::sin(i * 0.5) + (i % 3) * 0.1;  // sine + noise

    // Decimation filter: average groups of 4 samples, then take every 4th
    // Step 1: Chunk into groups of 4
    // Step 2: Average each group
    // Step 3: Result is the decimated signal

    std::cout << "Original (" << signal.size() << " samples): ";
    for (double s : signal)
        std::cout << std::round(s * 100) / 100 << ' ';
    std::cout << '\n';

    // Simple decimation (just stride, no averaging)
    auto simple = signal | std::views::stride(4);
    std::cout << "Simple stride-4 (" << std::ranges::distance(simple) << " samples): ";
    for (double s : simple)
        std::cout << std::round(s * 100) / 100 << ' ';
    std::cout << '\n';

    // Averaging decimation using chunk + transform
    auto averaged = signal
        | std::views::chunk(4)
        | std::views::transform([](auto chunk) {
              double sum = 0;
              int count = 0;
              for (double x : chunk) { sum += x; ++count; }
              return sum / count;
          });

    std::cout << "Averaged chunks (" << std::ranges::distance(averaged) << " samples): ";
    for (double s : averaged)
        std::cout << std::round(s * 100) / 100 << ' ';
    std::cout << '\n';
}
// Expected output:
// Original (24 samples): ... (24 values)
// Simple stride-4 (6 samples): ... (6 values)
// Averaged chunks (6 samples): ... (6 values, smoother)
```

**Simple decimation** with `stride(4)` picks every 4th sample - fast but potentially lossy if the discarded samples carry signal. **Averaging decimation** with `chunk(4) | transform(average)` averages each group before downsampling, which acts as a basic low-pass filter and preserves more information. Both produce 6 output samples from 24 inputs (a 4x reduction). The `chunk | transform` approach is the standard decimation filter pattern in signal processing.

### Q3: Explain the iterator category of `stride_view`

`stride_view` is unusual in that it preserves whatever iterator category the source has. If you feed it a random-access range, you get a random-access stride view. If you feed it a forward-only range, you get a forward-only stride view. The reason is that `stride_view`'s `operator++` just calls `++` on the underlying iterator `n` times (or uses `+= n` for random access), so it naturally inherits whatever capabilities the underlying iterator already has.

| Source category | stride_view category |
| --- | --- |
| random_access | random_access |
| bidirectional | bidirectional |
| forward | forward |
| input | input |

The code below uses `static_assert` to prove these claims at compile time, not just at runtime.

```cpp
#include <concepts>
#include <forward_list>
#include <iostream>
#include <list>
#include <ranges>
#include <vector>

int main() {
    std::vector<int>      vec = {1, 2, 3, 4, 5, 6, 7, 8};
    std::list<int>        lst = {1, 2, 3, 4, 5, 6, 7, 8};
    std::forward_list<int> fwd = {1, 2, 3, 4, 5, 6, 7, 8};

    // vector (random_access) -> stride is random_access
    auto sv = vec | std::views::stride(3);
    static_assert(std::ranges::random_access_range<decltype(sv)>);
    static_assert(std::ranges::sized_range<decltype(sv)>);
    std::cout << "vector | stride(3): random_access, size=" << std::ranges::size(sv) << '\n';

    // O(1) random access on stride view
    std::cout << "  sv[0]=" << sv[0] << " sv[1]=" << sv[1] << " sv[2]=" << sv[2] << '\n';

    // list (bidirectional) -> stride is bidirectional
    auto sl = lst | std::views::stride(2);
    static_assert(std::ranges::bidirectional_range<decltype(sl)>);
    static_assert(!std::ranges::random_access_range<decltype(sl)>);
    std::cout << "list | stride(2): bidirectional\n";

    // forward_list (forward) -> stride is forward
    auto sf = fwd | std::views::stride(2);
    static_assert(std::ranges::forward_range<decltype(sf)>);
    static_assert(!std::ranges::bidirectional_range<decltype(sf)>);
    std::cout << "forward_list | stride(2): forward\n";
}
// Expected output:
// vector | stride(3): random_access, size=3
//   sv[0]=1 sv[1]=4 sv[2]=7
// list | stride(2): bidirectional
// forward_list | stride(2): forward
```

Notice that `sv[0]`, `sv[1]`, `sv[2]` work because the underlying `vector` supports O(1) index access. `stride_view`'s `operator[]` maps `sv[i]` to `source[i * n]` directly, which is also O(1). That convenience disappears for `list` because lists don't support index access, and `stride_view` makes no attempt to fake it.

---

## Notes

- `stride(n)` requires `n >= 1`. `stride(1)` is the identity (every element).
- For random-access ranges, `operator[]` on stride_view is O(1): `sv[i]` maps to `source[i * n]`.
- `stride` + `chunk`: `stride(n)` picks one per group; `chunk(n)` gives all per group. They're complementary.
- Pre-C++23 workaround: use a custom iterator adapter or `filter` with an index counter.
