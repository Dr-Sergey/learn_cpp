# Use perf flamegraphs for visual CPU profiling

**Category:** Tooling and Debugging  
**Standard:** C++23  
**Reference:** <https://www.brendangregg.com/flamegraphs.html>  

---

## Topic Overview

When you're trying to speed up a program, the first question is always the same: "Where is the time actually going?" A flamegraph answers that question visually. Each box in the chart represents a function call, and its width tells you exactly how much of the total CPU time that function consumed. The wider the box, the more time is spent there, and that is where you should focus your optimization effort.

The key insight is that you read flamegraphs from the bottom up. The bottom row is always your entry point - often `main()` - and each row above it represents a callee. If a box at the top of a tall stack is very wide, that function is burning most of your CPU. If a box near the bottom is wide and has few things stacked above it, that code is doing most of the work directly (high "self time").

```cpp
Flamegraph (read bottom-up):
┌───────────────────────────────────────────────┐
│                 main()                          │
├────────────────────────┬──────────────────────┤
│    process_data() 60%    │   sort_results() 35% │
├──────────┬─────────────┤                      │
│ parse()  │ validate()   │                      │
│   40%    │    20%       │                      │
└──────────┴─────────────┴──────────────────────┘
  ^ widest frame = most time = optimize first
```

In this example, `parse()` at 40% is your prime optimization target. `validate()` at 20% is next. The narrow `sort_results()` box on the right is proportionally smaller and probably not worth touching first.

---

## Self-Assessment

### Q1: Generate a flamegraph with `perf record` + `flamegraph.pl`

Here's a simple program with an intentional hot spot so you can see how the flamegraph picks it up. Compile it with `-g` to preserve symbol names - without that flag, the output will be full of meaningless hex addresses.

```cpp
// hotspot.cpp - compile: g++ -O2 -g -std=c++20 hotspot.cpp -o hotspot
// -g is needed for symbol resolution in perf
#include <vector>
#include <algorithm>
#include <numeric>
#include <cmath>
#include <iostream>

void expensive_parse(std::vector<double>& data) {
    for (auto& d : data)
        d = std::sin(d) * std::cos(d) * std::log(d + 1.0);
}

void cheap_validate(const std::vector<double>& data) {
    for (const auto& d : data)
        if (std::isnan(d)) throw std::runtime_error("NaN!");
}

void sort_results(std::vector<double>& data) {
    std::sort(data.begin(), data.end());
}

int main() {
    std::vector<double> data(10'000'000);
    std::iota(data.begin(), data.end(), 1.0);

    expensive_parse(data);
    cheap_validate(data);
    sort_results(data);

    std::cout << "Done. First: " << data[0] << '\n';
}
```

Three functions are called from `main()`: a heavy trigonometric computation, a cheap validation pass, and a sort. The flamegraph should clearly show `expensive_parse` dominating. Run these three shell commands in order to collect samples, process the stack traces, and render the SVG.

```bash
# Step 1: Record CPU samples with call graph
$ perf record -g --call-graph dwarf -F 999 ./hotspot
# -F 999: sample frequency (999 Hz avoids aliasing with timer)
# -g --call-graph dwarf: accurate stack traces

# Step 2: Generate flamegraph
$ perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg
# Tools from: https://github.com/brendangregg/FlameGraph

# Step 3: Open in browser
$ firefox flamegraph.svg  # or any browser
# Interactive: click to zoom into a subtree, search for function names

# Alternative: perf's built-in flamegraph (Linux 5.8+)
$ perf script report flamegraph
```

The resulting SVG is interactive - you can click any box to zoom in on that part of the call tree, and use the search box to highlight a function by name across all stacks.

### Q2: Identify the hottest function and optimize it

After generating the flamegraph from Q1, reading it gives you a clear priority list. The percentages here represent each function's share of total samples.

```cpp
Flame graph analysis:
─────────────────────────────────────────────
  main()                                  100%
  ├─ expensive_parse()                    55%  <-- WIDEST = optimize!
  │   ├─ sin()                            20%  (self-time)
  │   ├─ cos()                            18%
  │   └─ log()                            17%
  ├─ sort_results()                       40%
  │   └─ __introsort_loop()               38%  (self-time)
  └─ cheap_validate()                      5%  (ignore)

Optimization target: expensive_parse() at 55%
```

Notice that `expensive_parse` spends its time roughly evenly across three transcendental functions. If you recall your trigonometric identities, `sin(x) * cos(x) == 0.5 * sin(2x)`, which means you can replace two calls with one.

```cpp
// Optimized version: precompute sin*cos = 0.5*sin(2x)
void expensive_parse_opt(std::vector<double>& data) {
    for (auto& d : data)
        d = 0.5 * std::sin(2.0 * d) * std::log(d + 1.0);
    // Reduced from 3 trig calls to 1 (sin(2x) = 2*sin(x)*cos(x))
}
```

One trig call instead of two is a meaningful saving when you're processing ten million elements. The flamegraph is what made it obvious which function deserved this attention in the first place.

### Q3: Compare before/after flamegraphs

After optimizing, you want confirmation that you actually improved things and you didn't just shift the bottleneck somewhere else. The `difffolded.pl` script produces a differential flamegraph where blue means faster and red means slower - an at-a-glance validation of your change.

```bash
# Before optimization:
$ perf record -g --call-graph dwarf ./hotspot_before
$ perf script | stackcollapse-perf.pl > before.folded
$ flamegraph.pl before.folded > before.svg

# After optimization:
$ perf record -g --call-graph dwarf ./hotspot_after
$ perf script | stackcollapse-perf.pl > after.folded
$ flamegraph.pl after.folded > after.svg

# Differential flamegraph (red = slower, blue = faster):
$ difffolded.pl before.folded after.folded | flamegraph.pl > diff.svg

# Expected diff flamegraph:
# - expensive_parse: BLUE (reduced from 55% to 30%)
# - sort_results: wider (now proportionally larger, was 40%, now ~55%)
# - cheap_validate: unchanged

# Also compare with perf stat:
$ perf stat ./hotspot_before
# 2.45 seconds time elapsed
$ perf stat ./hotspot_after
# 1.78 seconds time elapsed  (27% faster!)
```

Notice that `sort_results` appears proportionally wider in the diff even though it didn't get slower. That's expected - once `expensive_parse` shrinks, everything else looks relatively larger. The diff flamegraph's blue color on `sort_results` confirms it's actually the same speed or faster.

---

## Notes

- Always compile with `-g` (debug symbols) for meaningful function names.
- Use `-fno-omit-frame-pointer` for reliable stack traces with `--call-graph fp`.
- `--call-graph dwarf` is more accurate but slower than `--call-graph fp`.
- Differential flamegraphs are invaluable for validating optimizations.
- Online viewer: <https://www.speedscope.app/> imports `perf script` output.
- On macOS, use Instruments.app; on Windows, use ETW + WPA.
