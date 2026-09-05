# Benchmark-driven testing with Google Benchmark integration

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Performance testing** ensures your code meets latency and throughput targets. The idea is not just to check that a function produces the right answer - it is to check that it does so fast enough to matter in production. Google Benchmark (gbenchmark) is the standard micro-benchmarking framework for C++. It handles warmup, statistical analysis, and output formatting automatically, so you can focus on writing the benchmark logic rather than fighting timing machinery. Integrating benchmarks into CI prevents performance regressions from sneaking in unnoticed.

### Micro-benchmark vs Macro-benchmark

These two approaches answer different questions. Micro-benchmarks tell you how fast a single function is. Macro-benchmarks tell you whether the overall system meets its SLAs. You need both, but for catching algorithmic regressions, micro-benchmarks are where Google Benchmark shines.

| Aspect | Micro-benchmark (gbenchmark) | Macro-benchmark |
| --- | --- | --- |
| **Scope** | Single function/algorithm | End-to-end system |
| **Duration** | Nanoseconds to milliseconds | Seconds to minutes |
| **Environment** | Controlled, isolated | Production-like |
| **Determinism** | High (statistical filtering) | Lower (system noise) |
| **Use case** | Algorithm choice, hot loops | Capacity planning, SLAs |

---

## Self-Assessment

### Q1: Write effective Google Benchmark tests with proper setup

**Answer:**

First, bring in Google Benchmark via CMake's FetchContent, which is the easiest way to add it without a system-level install:

```cmake
# === CMake setup ===
include(FetchContent)
FetchContent_Declare(
    benchmark
    GIT_REPOSITORY https://github.com/google/benchmark.git
    GIT_TAG        v1.8.3
)
set(BENCHMARK_ENABLE_TESTING OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(benchmark)

add_executable(my_benchmarks bench_main.cpp)
target_link_libraries(my_benchmarks benchmark::benchmark benchmark::benchmark_main)
```

Now here are several benchmark functions showing the key patterns. Notice the `DoNotOptimize` and `ClobberMemory` calls - these are critical. Without them, the compiler sees that the result of the benchmark loop is never used and optimizes the entire loop away, giving you zero nanoseconds for everything.

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <algorithm>
#include <numeric>
#include <random>
#include <string>
#include <unordered_map>
#include <map>

// === Basic benchmark: vector vs list insertion ===
static void BM_VectorPushBack(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        for (int i = 0; i < state.range(0); ++i)
            v.push_back(i);
        benchmark::DoNotOptimize(v.data());
    }
    state.SetComplexityN(state.range(0));
}
BENCHMARK(BM_VectorPushBack)
    ->RangeMultiplier(4)
    ->Range(64, 1 << 20)
    ->Complexity(benchmark::oN);

// === With reserve: prove amortized O(1) ===
static void BM_VectorReservedPushBack(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        v.reserve(state.range(0));
        for (int i = 0; i < state.range(0); ++i)
            v.push_back(i);
        benchmark::DoNotOptimize(v.data());
    }
    state.SetComplexityN(state.range(0));
}
BENCHMARK(BM_VectorReservedPushBack)
    ->RangeMultiplier(4)
    ->Range(64, 1 << 20)
    ->Complexity(benchmark::oN);

// === Comparing sort algorithms ===
static void BM_StdSort(benchmark::State& state) {
    std::mt19937 rng(42);
    std::vector<int> data(state.range(0));
    std::iota(data.begin(), data.end(), 0);
    std::shuffle(data.begin(), data.end(), rng);

    for (auto _ : state) {
        auto copy = data;
        std::sort(copy.begin(), copy.end());
        benchmark::DoNotOptimize(copy.data());
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(state.iterations() * state.range(0));
}
BENCHMARK(BM_StdSort)->RangeMultiplier(4)->Range(256, 1 << 18);

// === Map vs unordered_map lookup ===
static void BM_MapLookup(benchmark::State& state) {
    std::map<int, int> m;
    for (int i = 0; i < state.range(0); ++i)
        m[i] = i;

    int found = 0;
    for (auto _ : state) {
        auto it = m.find(state.range(0) / 2);
        benchmark::DoNotOptimize(it);
    }
}
BENCHMARK(BM_MapLookup)->RangeMultiplier(10)->Range(10, 1000000);

static void BM_UnorderedMapLookup(benchmark::State& state) {
    std::unordered_map<int, int> m;
    for (int i = 0; i < state.range(0); ++i)
        m[i] = i;

    for (auto _ : state) {
        auto it = m.find(state.range(0) / 2);
        benchmark::DoNotOptimize(it);
    }
}
BENCHMARK(BM_UnorderedMapLookup)->RangeMultiplier(10)->Range(10, 1000000);
```

The `->Range(64, 1 << 20)` calls generate multiple benchmark runs at geometrically spaced sizes, which is how you see whether your algorithm's complexity matches theory. `SetComplexityN` then lets Google Benchmark fit a curve and report the measured Big-O.

### Q2: Integrate benchmarks into CI to catch performance regressions

**Answer:**

Running benchmarks in CI requires a bit more machinery than just running tests, because you need to compare results against a previous baseline. The workflow below captures benchmark output as JSON, then uses a Python snippet to flag any benchmark that slowed down by more than 10%.

```yaml
# === GitHub Actions CI benchmark job ===
name: Benchmarks
on:
  pull_request:
    paths: ['src/**', 'benchmarks/**']

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build benchmarks
        run: |
          cmake -B build -DCMAKE_BUILD_TYPE=Release
          cmake --build build --target my_benchmarks
      - name: Run benchmarks (JSON output)
        run: |
          ./build/my_benchmarks \
            --benchmark_format=json \
            --benchmark_out=benchmark_results.json \
            --benchmark_repetitions=5 \
            --benchmark_min_time=0.5
      - name: Compare with baseline
        run: |
          # Download previous baseline
          # Use google/benchmark's compare.py tool
          python3 tools/compare.py benchmarks \
            baseline.json benchmark_results.json
      - name: Check for regressions (>10% slower)
        run: |
          python3 - <<'EOF'
          import json, sys
          with open('benchmark_results.json') as f:
              results = json.load(f)
          with open('baseline.json') as f:
              baseline = json.load(f)

          baseline_map = {b['name']: b['cpu_time']
                          for b in baseline['benchmarks']
                          if b.get('run_type') == 'iteration'}

          regressions = []
          for bench in results['benchmarks']:
              if bench.get('run_type') != 'iteration':
                  continue
              name = bench['name']
              if name in baseline_map:
                  ratio = bench['cpu_time'] / baseline_map[name]
                  if ratio > 1.10:  # 10% regression threshold
                      regressions.append(f"{name}: {ratio:.2f}x slower")

          if regressions:
              print("PERFORMANCE REGRESSIONS DETECTED:")
              for r in regressions:
                  print(f"  - {r}")
              sys.exit(1)
          print("No regressions detected.")
          EOF
```

The 10% threshold accounts for natural noise on shared CI runners. If you tighten it to 5%, you will get false positives. If you loosen it to 20%, you will miss real regressions. Tune this based on how noisy your runners are.

### Q3: Advanced benchmarking techniques

**Answer:**

The examples above cover basic cases. Here are three more patterns you will reach for regularly: fixture-based setup, custom throughput counters, multi-threaded benchmarks, and template benchmarks.

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <string>
#include <memory>

// === Fixture-based benchmark with setup/teardown ===
class DatabaseBenchmark : public benchmark::Fixture {
public:
    void SetUp(benchmark::State& state) override {
        // Expensive one-time setup per benchmark
        data_.resize(state.range(0));
        std::iota(data_.begin(), data_.end(), 0);
    }

    void TearDown(benchmark::State&) override {
        data_.clear();
    }

protected:
    std::vector<int> data_;
};

BENCHMARK_DEFINE_F(DatabaseBenchmark, LinearSearch)(benchmark::State& state) {
    for (auto _ : state) {
        auto it = std::find(data_.begin(), data_.end(), state.range(0) - 1);
        benchmark::DoNotOptimize(it);
    }
}
BENCHMARK_REGISTER_F(DatabaseBenchmark, LinearSearch)->Range(100, 100000);

// === Custom counters for throughput ===
static void BM_MemcpyThroughput(benchmark::State& state) {
    auto size = state.range(0);
    auto src = std::make_unique<char[]>(size);
    auto dst = std::make_unique<char[]>(size);
    std::memset(src.get(), 'x', size);

    for (auto _ : state) {
        std::memcpy(dst.get(), src.get(), size);
        benchmark::ClobberMemory();
    }

    state.SetBytesProcessed(state.iterations() * size);
    state.counters["size_KB"] = size / 1024.0;
}
BENCHMARK(BM_MemcpyThroughput)
    ->RangeMultiplier(4)
    ->Range(1 << 10, 1 << 24);  // 1KB to 16MB

// === Multi-threaded benchmark ===
static void BM_AtomicIncrement(benchmark::State& state) {
    static std::atomic<int> counter{0};

    if (state.thread_index() == 0)
        counter.store(0);

    for (auto _ : state) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}
BENCHMARK(BM_AtomicIncrement)->Threads(1)->Threads(2)->Threads(4)->Threads(8);

// === Template benchmarks ===
template<typename Container>
static void BM_ContainerIterate(benchmark::State& state) {
    Container c;
    for (int i = 0; i < state.range(0); ++i)
        c.insert(c.end(), i);

    for (auto _ : state) {
        int sum = 0;
        for (const auto& v : c) sum += v;
        benchmark::DoNotOptimize(sum);
    }
}
BENCHMARK_TEMPLATE(BM_ContainerIterate, std::vector<int>)->Range(64, 1 << 16);
BENCHMARK_TEMPLATE(BM_ContainerIterate, std::list<int>)->Range(64, 1 << 16);
```

The fixture pattern is particularly useful when your benchmark needs non-trivial setup (loading data, building a tree, connecting to a test database) that should not be counted in the benchmark time. The `SetUp` cost is excluded from the measured loop automatically.

---

## Notes

- Always build benchmarks with `-O2` or the Release configuration - debug builds are meaningless for performance measurement.
- `DoNotOptimize()` prevents the compiler from eliminating dead code in the benchmark loop. Never skip it when the result is not otherwise observable.
- `ClobberMemory()` forces memory writes to be visible and prevents store elimination. Use it after writes whose results are "consumed" only by the next benchmark iteration.
- Use `--benchmark_repetitions=N` with `--benchmark_report_aggregates_only` for statistical robustness across multiple runs.
- `state.SetItemsProcessed()` and `state.SetBytesProcessed()` enable throughput reporting in the output, which is much more useful than raw nanoseconds for I/O-heavy code.
- Benchmark names appear in output exactly as the function name, so use descriptive names from the start.
- CI regression detection should allow a 5-10% noise margin on shared runners - those machines have variable load.
- Use `--benchmark_filter=regex` to run specific benchmarks during development instead of waiting for the full suite.
