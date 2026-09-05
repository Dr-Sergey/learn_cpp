# Use compile-time benchmarking to track build performance regressions

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://github.com/aras-p/ClangBuildAnalyzer>  

---

## Topic Overview

Build performance is easy to ignore until the problem is bad enough that developers start complaining. By the time a clean build takes 20 minutes, the damage is already done - and it happened one slow header at a time. Compile-time benchmarking gives you the tools to see where that time is going and catch regressions before they compound.

Clang's `-ftime-trace` flag makes every compilation produce a JSON trace file (compatible with Chrome's tracing viewer) that breaks down exactly how long each phase of compilation took, including every header inclusion and every template instantiation. ClangBuildAnalyzer aggregates those traces across your whole project into an actionable report.

```cpp
CI pipeline:
  Build with -ftime-trace
       │
  ClangBuildAnalyzer
       │
  Report: "<regex> takes 180ms per TU"
          "Widget<T> instantiated 47 times"
       │
  Compare against baseline
       │
  Alert if regression > 10%
```

---

## Self-Assessment

### Q1: Find the slowest headers and templates with ClangBuildAnalyzer

The workflow is a three-step process: build with tracing enabled, capture the trace files, then analyze them. The result is a ranked list of everything that is slowing down your builds:

```bash
# Step 1: Build with -ftime-trace
cmake -B build -DCMAKE_CXX_COMPILER=clang++ \
      -DCMAKE_CXX_FLAGS="-ftime-trace"
cmake --build build

# Step 2: Capture all trace files
ClangBuildAnalyzer --all build capture.bin

# Step 3: Analyze
ClangBuildAnalyzer --analyze capture.bin
```

The output gives you ranked lists across several dimensions - not just which files are slow, but which templates are being instantiated repeatedly and which headers are the most expensive:

```text
**** Files that took longest to compile (wall time):
  180ms  src/parser.cpp
  152ms  src/engine.cpp
   98ms  src/renderer.cpp

**** Templates that took longest to instantiate:
  45ms  std::vector<Widget> (18 times)
  32ms  std::map<std::string, Config> (12 times)
  28ms  fmt::format<...> (47 times)

**** Files that were included the most:
  47   <vector>
  42   <string>
  38   <iostream>     <-- every include costs ~15ms
  12   <regex>        <-- 180ms per include!

**** Expensive headers (time * include count):
  2160ms  <regex>       (12 includes * 180ms)
   810ms  <iostream>    (38 includes * ~21ms)
   450ms  <map>         (15 includes * ~30ms)
```

The "time * include count" metric at the bottom is the most actionable number. `<regex>` being included 12 times at 180 ms each means it contributes over 2 seconds to your total build time. Moving it from a header to a `.cpp` file would eliminate that cost from 11 of those 12 translation units.

### Q2: Measure the cost of heavy headers

If you have never thought about header compile cost before, the numbers are striking. You can measure them directly with a few minimal test files:

```bash
# Create minimal test files:
echo '#include <regex>' > test_regex.cpp
echo 'int main() {}' >> test_regex.cpp

echo '#include <iostream>' > test_iostream.cpp
echo 'int main() {}' >> test_iostream.cpp

echo 'int main() {}' > test_empty.cpp

# Measure with time:
time clang++ -std=c++20 -c test_empty.cpp     # ~0.05s baseline
time clang++ -std=c++20 -c test_iostream.cpp   # ~0.15s (+0.10s)
time clang++ -std=c++20 -c test_regex.cpp      # ~0.35s (+0.30s)
```

Here is the approximate overhead for the most common standard library headers:

| Header | Approx. Compile Overhead | Lines Pulled In |
| --- | --- | --- |
| `<array>` | ~5ms | ~2K |
| `<vector>` | ~15ms | ~10K |
| `<string>` | ~20ms | ~12K |
| `<iostream>` | ~100ms | ~25K |
| `<map>` | ~30ms | ~15K |
| `<algorithm>` | ~25ms | ~14K |
| `<regex>` | ~300ms | ~50K |
| `<variant>` | ~40ms | ~8K |

These costs multiply by the number of translation units that include the header. A widely-included header with `<regex>` in it can easily add minutes to a large project build. The fix is straightforward: move the heavy include to the `.cpp` file and keep the header lightweight:

```cpp
// BAD: includes <regex> in a widely-included header
// config.h
#include <regex>  // 300ms * N translation units = slow!

bool validate(const std::string& s) {
    return std::regex_match(s, std::regex("[a-z]+"));
}

// GOOD: move to .cpp file, keep header light
// config.h
#include <string>
bool validate(const std::string& s);  // forward declare only

// config.cpp
#include "config.h"
#include <regex>  // heavy include only in ONE TU
bool validate(const std::string& s) {
    return std::regex_match(s, std::regex("[a-z]+"));
}
```

### Q3: Forward declarations and include reduction for 20%+ compile time savings

A common symptom of include bloat is a header that is included by many files but only needs a subset of what it includes. A header might include `<algorithm>` and `<iostream>` because its member function implementations use them - but if those implementations live in the `.cpp` file, there is no reason to include those headers in the `.h` file at all.

Before the fix, every includer pays the full cost:

```cpp
// widget.h (included by 30 files)
#include <vector>     // 15ms
#include <map>        // 30ms
#include <string>     // 20ms
#include <algorithm>  // 25ms
#include <iostream>   // 100ms = 190ms per includer!

class Widget {
    std::vector<int> data_;
    std::map<std::string, int> props_;
public:
    void process();
    void print() const;
};
```

After removing the headers that are only needed in the implementation:

```cpp
// widget.h (optimized)
#include <vector>  // needed: data_ member
#include <string>  // needed: map key type
#include <map>     // needed: props_ member
// Removed <algorithm> and <iostream>: only needed in .cpp!

class Widget {
    std::vector<int> data_;
    std::map<std::string, int> props_;
public:
    void process();
    void print() const;
};

// widget.cpp
#include "widget.h"
#include <algorithm>  // moved here
#include <iostream>   // moved here

void Widget::process() { std::sort(data_.begin(), data_.end()); }
void Widget::print() const { for (auto& d : data_) std::cout << d << ' '; }
```

The impact on build time is proportional to how many files include the header:

| Metric | Before | After | Savings |
| --- | --- | --- | --- |
| Header cost per TU | 190ms | 65ms | -66% |
| Total (30 TUs) | 5.7s | 1.95s | -66% |

A two-thirds reduction in header parsing time for that one header, with no functional change to the code.

---

## Notes

- Use `include-what-you-use` (IWYU) to automate the analysis of which includes can be removed or replaced with forward declarations.
- Precompiled headers (PCH) are another strategy for when many translation units include the same heavy set of headers - PCH compiles them once and reuses the result.
- Track compile times in CI by storing build times per commit and alerting on regressions above a threshold, just as you would for runtime performance.
- Forward declare classes used only by pointer or reference in headers - you do not need the full definition unless the header uses the type's members or derives from it.
- Use `-ftime-trace-granularity=100` to capture trace events as small as 100 microseconds, which gives you a finer-grained picture of where time is spent.
