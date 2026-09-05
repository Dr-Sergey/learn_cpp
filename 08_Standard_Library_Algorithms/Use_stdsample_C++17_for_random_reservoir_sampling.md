# Use std::sample (C++17) for random reservoir sampling

**Category:** Standard Library Algorithms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/sample>  

---

## Topic Overview

`std::sample` (C++17) selects `n` elements uniformly at random from a range, writing them to an output iterator. It implements **reservoir sampling**, meaning it requires only a single pass through the input range - perfect for streams or forward-only ranges where you cannot rewind.

### Signature

```cpp
#include <algorithm>
#include <random>

template<class PopIter, class SampleIter, class Distance, class URBG>
SampleIter sample(PopIter first, PopIter last,
                  SampleIter out, Distance n, URBG&& g);
```

### Key Properties

| Property | Detail |
| --- | --- |
| Passes | Single pass through input |
| Stability | Preserves relative order (for forward iterators) |
| Uniformity | Each element has equal probability n/N of being selected |
| Input requirement | Input: at least InputIterator; Output: at least OutputIterator |
| If n >= N | Returns all elements |

---

## Self-Assessment

### Q1: Sample 10 elements uniformly at random from a 1000-element range using std::sample

The call is straightforward: you provide the population range, a back inserter for the results, how many elements you want, and a random engine. The selected elements arrive in their original relative order.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <random>
#include <numeric>

int main() {
    // Create a 1000-element range
    std::vector<int> population(1000);
    std::iota(population.begin(), population.end(), 1);  // 1..1000

    // Random engine
    std::mt19937 rng(42);  // fixed seed for reproducibility

    // === Sample 10 elements ===
    std::vector<int> sample_result;
    std::sample(population.begin(), population.end(),
                std::back_inserter(sample_result),
                10, rng);

    std::cout << "Sample of 10 from 1000: ";
    for (int x : sample_result) std::cout << x << " ";
    std::cout << "\n";
    // e.g.: 37 143 250 389 412 567 623 741 856 991

    // Note: elements appear in their original relative order!
    // This is guaranteed for forward+ iterators.

    // === Sample from a string ===
    std::string alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    std::string random_letters;
    std::sample(alphabet.begin(), alphabet.end(),
                std::back_inserter(random_letters),
                5, rng);
    std::cout << "Random letters: " << random_letters << "\n";

    // === Multiple samples show different results ===
    for (int trial = 0; trial < 3; ++trial) {
        std::vector<int> s;
        std::sample(population.begin(), population.end(),
                    std::back_inserter(s), 5, rng);
        std::cout << "Trial " << trial << ": ";
        for (int x : s) std::cout << x << " ";
        std::cout << "\n";
    }

    // === If n >= population size, returns everything ===
    std::vector<int> small = {10, 20, 30};
    std::vector<int> over_sample;
    std::sample(small.begin(), small.end(),
                std::back_inserter(over_sample), 100, rng);
    std::cout << "\nOver-sample (n=100, size=3): ";
    for (int x : over_sample) std::cout << x << " ";
    std::cout << "\n";
    // 10 20 30 - returns all

    return 0;
}
```

The output order being preserved is an often-missed detail: if you sample from a sorted vector, the result is also sorted. This is because the reservoir algorithm works by deciding whether each element "makes the cut" as it is seen, not by shuffling afterward.

### Q2: Explain why std::sample only requires a single pass through the input range

This is the clever part. The reason you don't need to know the total size upfront - or rewind the range - comes down to a mathematical invariant that Vitter's Algorithm R maintains at every step.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <random>
#include <iterator>
#include <numeric>

// Demonstration of the reservoir sampling algorithm that std::sample uses

void reservoir_sample_manual(const std::vector<int>& stream, int k) {
    std::mt19937 rng(123);
    std::vector<int> reservoir(stream.begin(), stream.begin() + k);

    // === Reservoir sampling algorithm (Vitter's Algorithm R) ===
    // 1. Fill reservoir with first k elements
    // 2. For each subsequent element i (from k to n-1):
    //    - Generate random j in [0, i]
    //    - If j < k, replace reservoir[j] with stream[i]

    for (size_t i = k; i < stream.size(); ++i) {
        std::uniform_int_distribution<size_t> dist(0, i);
        size_t j = dist(rng);
        if (j < static_cast<size_t>(k)) {
            reservoir[j] = stream[i];
        }
    }

    std::cout << "Manual reservoir sample: ";
    for (int x : reservoir) std::cout << x << " ";
    std::cout << "\n";
}

int main() {
    std::vector<int> data(100);
    std::iota(data.begin(), data.end(), 1);

    reservoir_sample_manual(data, 5);

    // === Why single-pass works ===
    // At each step i, every element seen so far has equal probability k/i
    // of being in the reservoir. This is provable by induction.
    //
    // Key insight: we don't need to know the total size N upfront!
    // This means std::sample works with:
    //   - Forward iterators (single pass)
    //   - Streams of unknown length
    //   - Generators that can't be rewound
    //
    // Contrast with shuffle: shuffle needs random access (requires
    // knowing the size to pick random positions)

    // === std::sample with forward-only range ===
    // Works just fine - single pass is all it needs
    std::vector<int> result;
    std::mt19937 rng(42);
    std::sample(data.begin(), data.end(), std::back_inserter(result), 5, rng);

    std::cout << "std::sample: ";
    for (int x : result) std::cout << x << " ";
    std::cout << "\n";

    return 0;
}
```

The invariant is: after processing element `i`, every element seen so far has exactly `k/i` probability of being in the reservoir. When element `i+1` arrives, it gets a `k/(i+1)` chance by being placed with probability `k/(i+1)` and displacing a random existing reservoir entry. The math works out so the invariant holds at every step - which is why you never need a second pass.

### Q3: Compare std::sample with a Fisher-Yates shuffle for selecting a random subset

Both approaches produce a uniform random subset, but they have different tradeoffs. The right choice depends on whether your data is mutable, whether K is much smaller than N, and whether you need order preserved in the output.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <random>
#include <numeric>
#include <chrono>

int main() {
    constexpr int N = 1'000'000;
    constexpr int K = 100;
    std::mt19937 rng(42);

    std::vector<int> data(N);
    std::iota(data.begin(), data.end(), 0);

    // === Method 1: std::sample - O(N) single pass, O(K) output space ===
    auto t1 = std::chrono::high_resolution_clock::now();
    std::vector<int> sampled;
    std::sample(data.begin(), data.end(), std::back_inserter(sampled), K, rng);
    auto t2 = std::chrono::high_resolution_clock::now();
    auto ms_sample = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    // === Method 2: Partial Fisher-Yates shuffle - O(K) swaps ===
    // shuffle first K elements, then take them
    auto data_copy = data;
    t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < K; ++i) {
        std::uniform_int_distribution<int> dist(i, N - 1);
        std::swap(data_copy[i], data_copy[dist(rng)]);
    }
    std::vector<int> shuffled(data_copy.begin(), data_copy.begin() + K);
    t2 = std::chrono::high_resolution_clock::now();
    auto ms_shuffle = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    std::cout << "std::sample:           " << ms_sample << " us\n";
    std::cout << "Partial Fisher-Yates:  " << ms_shuffle << " us\n";

    // === Comparison ===
    // | Feature                | std::sample      | Fisher-Yates partial |
    // |------------------------|------------------|---------------------|
    // | Time                   | O(N)             | O(K)                |
    // | Space                  | O(K) output      | O(N) (needs copy)   |
    // | Modifies input?        | No               | Yes (shuffles)      |
    // | Preserves order?       | Yes (forward+)   | No                  |
    // | Works with streams?    | Yes              | No (needs random access) |
    // | When K << N            | Slower           | Faster              |
    // | When immutable input   | Preferred        | Needs copy          |

    std::cout << "\nSample (order preserved): ";
    for (int i = 0; i < 10 && i < K; ++i) std::cout << sampled[i] << " ";
    std::cout << "...\n";

    std::cout << "Shuffle (random order):   ";
    for (int i = 0; i < 10 && i < K; ++i) std::cout << shuffled[i] << " ";
    std::cout << "...\n";

    return 0;
}
```

Notice that partial Fisher-Yates is faster when K is tiny relative to N (only K swaps needed), but it requires a mutable copy of the entire N-element input. `std::sample` scans the whole range but needs no extra memory beyond the K-element output and never touches the source. When the input is a stream or a read-only range, `std::sample` is the only option.

---

## Notes

- `std::sample` preserves **relative order** of selected elements when using forward+ iterators. For input iterators, order may not be preserved.
- Always use a proper random engine (`std::mt19937`), not `rand()`.
- `std::sample` does NOT modify the input range - it's a non-mutating algorithm.
- For **K << N** with random-access ranges, a partial Fisher-Yates shuffle is often faster (O(K) vs O(N)), but it requires a mutable copy of the data.
- If you need **random order** in the output, call `std::shuffle` on the sample result.
