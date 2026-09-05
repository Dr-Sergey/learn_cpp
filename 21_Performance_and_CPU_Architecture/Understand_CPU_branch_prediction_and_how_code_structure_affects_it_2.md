# Understand CPU branch prediction and how code structure affects it (Part 2: Pipeline Cost)

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/likely>  

---

## Topic Overview

When the CPU mispredicts a branch, it must flush the speculative pipeline and restart from the correct path. On modern superscalar CPUs, this costs 10-20 cycles of wasted work. The reason this trips people up is that "10-20 cycles" sounds abstract until you connect it to what is physically happening: the CPU has already speculatively fetched, decoded, and started executing a dozen or more instructions down the wrong path. All of that work is discarded.

The picture below shows the contrast between a correct prediction (the pipeline keeps flowing) and a mispredict (everything speculatively in flight is thrown away and the pipeline stalls while it refills from the correct target):

```cpp
Pipeline on correct prediction:     Pipeline on mispredict:

  Fetch -> Decode -> Execute          Fetch -> Decode -> Execute
   [A]     [B]      [C]                [A]     [B]      [wrong!]
   [B]     [C]      [D]                [B]     [wrong!]  FLUSH
   [C]     [D]      [E]                FLUSH   FLUSH     FLUSH
   ~3 instructions in flight            restart from correct path
                                        ~15 cycles wasted!
```

The cost scales with pipeline depth. Deeper pipelines keep more speculative work in flight, so there is more to throw away when the prediction is wrong.

| CPU | Pipeline depth | Mispredict cost |
| --- | --- | --- |
| Intel Skylake | 14-19 stages | ~15 cycles |
| Intel Alder Lake | 20+ stages | ~17 cycles |
| AMD Zen 4 | 19 stages | ~13 cycles |
| Apple M2 | 16 stages | ~14 cycles |
| ARM Cortex-A76 | 13 stages | ~11 cycles |

---

## Self-Assessment

### Q1: Why mispredictions cost 10-20 cycles

The comments here do the heavy lifting. This code is set up to isolate the branch-prediction effect by comparing two inputs to the same loop - one where the branch is always taken (100% predictable) and one where it is random (50% mispredict rate). The only variable is the data.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// The mispredict cost comes from the PIPELINE DEPTH:
// 1. CPU fetches and decodes ~15 instructions ahead speculatively
// 2. On mispredict: all speculative work is thrown away
// 3. Pipeline must refill from the correct branch target
// 4. No useful work for ~15 cycles ("bubble")
//
// On a 4 GHz CPU doing 4 instructions/cycle:
//   15 cycles * 4 IPC = 60 instructions of lost throughput!
//
// For a loop with 50% mispredict rate:
//   Each iteration: ~5 instructions + 50% * 15 cycle penalty = ~12.5 cycles
//   vs predicted:   ~5 instructions + 0% penalty = ~5 cycles
//   => 2.5x slowdown from mispredictions alone

int main() {
    constexpr int N = 100'000'000;
    std::mt19937 rng(42);

    // Predictable branch: always taken
    std::vector<int> predictable(N, 200);  // all > 128

    // Unpredictable branch: 50/50
    std::vector<int> unpredictable(N);
    for (auto& x : unpredictable) x = rng() % 256;

    auto bench = [](const std::vector<int>& data, const char* label) {
        volatile long long sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int x : data) {
            if (x >= 128) sum += x;  // same branch in both cases
        }
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(predictable,   "Predictable (>99% hit)");
    bench(unpredictable, "Random (50% mispredict)");
    // Typical: Predictable ~80ms, Random ~250ms (3x slower!)
    // The only difference: branch prediction accuracy.
}
```

The 3x slowdown from a single branch in a tight loop is a striking result. The branch itself is trivial - one comparison and one conditional jump. All the cost comes from the pipeline flushes.

### Q2: Sorting inputs to minimize branch mispredictions

Sorting is a classic technique for turning an unpredictable branch into a predictable one. When data is sorted, the branch outcome changes only once - at the transition point - so the predictor makes virtually no mistakes.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>
#include <random>
#include <numeric>

int main() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 256;

    auto process = [](const std::vector<int>& v) {
        long long sum = 0;
        for (int x : v) {
            if (x >= 128) sum += x;   // conditional branch
            // Unsorted: branch outcome is random -> 50% mispredict
            // Sorted:   first half always not-taken, second half always taken
        }
        return sum;
    };

    // UNSORTED:
    auto t0 = std::chrono::high_resolution_clock::now();
    long long sum1 = 0;
    for (int r = 0; r < 10; ++r) sum1 += process(data);
    auto t1 = std::chrono::high_resolution_clock::now();

    // SORTED:
    std::sort(data.begin(), data.end());
    auto t2 = std::chrono::high_resolution_clock::now();
    long long sum2 = 0;
    for (int r = 0; r < 10; ++r) sum2 += process(data);
    auto t3 = std::chrono::high_resolution_clock::now();

    auto ms1 = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    auto ms2 = std::chrono::duration_cast<std::chrono::milliseconds>(t3 - t2);

    std::cout << "Unsorted: " << ms1.count() << " ms\n";
    std::cout << "Sorted:   " << ms2.count() << " ms\n";
    // Typical: Unsorted ~600ms, Sorted ~150ms (4x faster!)
    //
    // Why it works:
    // Unsorted: [37, 200, 15, 180, 242, 5, ...] -> branch unpredictable
    // Sorted:   [0, 1, 2, ..., 127, 128, 129, ..., 255]
    //           ^--- all not-taken ---^^--- all taken ---^
    //           Only 1 mispredict at the 127->128 transition!
}
```

Of course sorting has its own cost - O(N log N) - so this technique only makes sense when you will process the data many times, or when you can sort once and reuse. If you only pass through the data once, a branchless formulation is usually the better answer.

### Q3: Measuring with `perf stat -e branch-misses`

Here is how to run a controlled experiment with `perf stat` to confirm that the performance difference is actually coming from branch mispredictions and not something else (cache misses, instruction throughput, etc.).

```cpp
#include <iostream>
#include <vector>
#include <random>
#include <algorithm>

// Two versions of the same algorithm to compare with perf stat:

void version_unsorted() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 256;
    // NOT sorted: random branch outcomes

    volatile long long sum = 0;
    for (int rep = 0; rep < 100; ++rep) {
        for (int x : data) {
            if (x >= 128) sum += x;
        }
    }
    std::cout << "Sum: " << sum << '\n';
}

void version_sorted() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 256;
    std::sort(data.begin(), data.end());  // sorted!

    volatile long long sum = 0;
    for (int rep = 0; rep < 100; ++rep) {
        for (int x : data) {
            if (x >= 128) sum += x;
        }
    }
    std::cout << "Sum: " << sum << '\n';
}

int main() {
    // Compile two versions:
    //   g++ -O2 -DUNSORTED -o unsorted branch.cpp
    //   g++ -O2 -DSORTED   -o sorted   branch.cpp
    //
    // Measure with perf:
    //   perf stat -e branches,branch-misses ./unsorted
    //   perf stat -e branches,branch-misses ./sorted
    //
    // Expected output:
    //
    // UNSORTED:
    //   1,000,000,000  branches
    //     250,000,000  branch-misses   # 25.00%   <-- BAD!
    //   Time: ~8.5 sec
    //
    // SORTED:
    //   1,000,000,000  branches
    //         100,000  branch-misses   # 0.01%    <-- GOOD!
    //   Time: ~2.1 sec
    //
    // Key perf stat counters for branch analysis:
    //   perf stat -e branches,branch-misses,\
    //              branch-loads,branch-load-misses,\
    //              L1-icache-load-misses ./program

#ifdef SORTED
    version_sorted();
#else
    version_unsorted();
#endif
}
```

The `perf stat` output is the definitive evidence: 250 million branch misses on the unsorted run versus 100 thousand on the sorted run. That is a 2500x reduction in mispredictions, which translates directly into the 4x wall-clock speedup. When you see a loop that is slower than expected and you cannot explain it from cache behavior alone, branch misses are the next thing to check.

---

## Notes

- The mispredict cost is proportional to pipeline depth (deeper pipeline = higher cost).
- Modern TAGE predictors handle regular patterns well; truly random branches are the worst case.
- `perf stat -e branch-misses` is the primary diagnostic tool.
- Branchless alternatives (cmov, arithmetic) eliminate misprediction entirely.
- `[[likely]]` / `[[unlikely]]` affect code layout, not hardware prediction.
