# Use Bloaty McBloatface to analyze and reduce binary size

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://github.com/google/bloaty>  

---

## Topic Overview

If your binary has grown large and you have no idea what is eating the space, Bloaty McBloatface is the tool to reach for. It analyzes ELF, Mach-O, and PE binaries and breaks down size by sections, symbols, compilation units, or source files - giving you an actionable picture of exactly where the bytes are coming from.

You can install it with `sudo apt install bloaty` or build it from source. The basic usage looks like this:

```bash
Install: sudo apt install bloaty  (or build from source)
Usage:   bloaty ./myapp
         bloaty -d compileunits ./myapp
         bloaty -d symbols ./myapp
         bloaty ./after -- ./before    (diff mode)
```

The diff mode (`./after -- ./before`) is especially useful: it tells you how a change you just made affected binary size, section by section.

---

## Self-Assessment

### Q1: Run Bloaty and identify top binary size contributors

Start by building your program and then running Bloaty with no extra flags to get the section-level breakdown.

```bash
# Build a sample project
g++ -std=c++20 -O2 -g -o myapp main.cpp

# Default: show sections
bloaty ./myapp
```

You will see output broken down by ELF section. A typical result looks something like this:

```text
    FILE SIZE        VM SIZE
 --------------  --------------
  34.5%  1.20Mi  42.1%  1.20Mi    .text          (code)
  26.3%   940Ki   0.0%       0    .debug_info    (debug data)
  15.7%   560Ki   0.0%       0    .debug_str     (debug strings)
   8.2%   293Ki  10.3%   293Ki    .rodata        (read-only data)
   5.1%   182Ki   6.4%   182Ki    .eh_frame      (exception handling)
   3.8%   136Ki   4.8%   136Ki    .data          (initialized data)
   2.4%    86Ki   3.0%    86Ki    .bss           (uninitialized data)
```

Notice that `.debug_info` and `.debug_str` show a large FILE SIZE but zero VM SIZE. That is because debug data is not loaded into memory at runtime - it only lives on disk. Once you see what is large at the section level, drill down to the symbol level to find the biggest individual functions:

```bash
# Drill down by symbols (find biggest functions)
bloaty -d symbols -n 10 ./myapp
```

The output will typically show standard library templates near the top:

```cpp
# Typical top offenders:
  12.3%  std::__cxx11::basic_string<...>::_M_mutate(...)    # string ops
  8.7%   std::vector<Widget>::_M_realloc_insert(...)        # vector growth
  6.2%   __cxa_throw                                        # exception support
  4.1%   MyApp::processData()                               # your code
```

This is the key insight: your own code is often not the top contributor. It is usually the standard library template instantiations that consume the most space.

### Q2: `-O2` vs `-Os` binary size and performance

Here is a concrete program to benchmark. The goal is to compare what `-O2` (optimize for speed) and `-Os` (optimize for size) produce.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <chrono>

void process(std::vector<int>& data) {
    std::sort(data.begin(), data.end());
    auto sum = std::accumulate(data.begin(), data.end(), 0LL);
    std::cout << "Sum: " << sum << '\n';
}

int main() {
    std::vector<int> data(1'000'000);
    std::iota(data.begin(), data.end(), 0);

    auto start = std::chrono::steady_clock::now();
    process(data);
    auto end = std::chrono::steady_clock::now();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "Time: " << ms.count() << " ms\n";
}
```

Build with both flags, strip the debug info, then compare with Bloaty's diff mode:

```bash
# Build with different flags and compare:
g++ -std=c++20 -O2 -o prog_O2 prog.cpp && strip prog_O2
g++ -std=c++20 -Os -o prog_Os prog.cpp && strip prog_Os

bloaty prog_Os -- prog_O2
```

The diff output shows the `.text` section shrinking with `-Os` because the compiler avoids loop unrolling and other size-expanding optimizations:

```text
     FILE SIZE       VM SIZE
  --------------  --------------
  -18.2%  -42Ki  -18.2%  -42Ki    .text        # -Os: less code (no unrolling)
   +0.0%      0   +0.0%      0    .rodata      # usually same
  -12.3%  -42Ki  -12.3%  -42Ki    TOTAL
```

The trade-off is real and worth measuring before committing to `-Os`:

| Metric | `-O2` | `-Os` | Difference |
| --- | --- | --- | --- |
| Binary size | ~230KB | ~188KB | -18% |
| Runtime | ~45ms | ~52ms | +15% slower |
| Best for | Servers, desktop | Embedded, IoT | Depends on constraints |

### Q3: `-fvisibility=hidden` + LTO for maximum size reduction

If you want to squeeze binary size as hard as possible, the most effective combination is hidden symbol visibility plus link-time optimization. Here is how to test all four variants and compare them against a baseline:

```bash
# Baseline
g++ -std=c++20 -O2 -o baseline prog.cpp && strip baseline

# With hidden visibility (only exported symbols visible)
g++ -std=c++20 -O2 -fvisibility=hidden -o hidden prog.cpp && strip hidden

# With LTO (link-time optimization)
g++ -std=c++20 -O2 -flto -o lto prog.cpp && strip lto

# Combined: hidden + LTO
g++ -std=c++20 -O2 -fvisibility=hidden -flto -o combined prog.cpp && strip combined

# Compare all against baseline:
bloaty combined -- baseline
```

The reason these two flags work so well together is that LTO can only eliminate code it knows is unreachable, and `-fvisibility=hidden` dramatically expands the set of symbols the linker is allowed to throw away. They are synergistic:

| Technique | Size Reduction | How It Works |
| --- | --- | --- |
| `-fvisibility=hidden` | 5-15% | Hides internal symbols; linker can remove unused |
| `-flto` | 10-20% | Cross-TU inlining + dead code elimination |
| Combined | 15-30% | Synergy: LTO + fewer visible symbols = more can be removed |
| + `-Os` | 25-40% | Stack all three for maximum reduction |

To apply all of this in CMake, add the following to your `CMakeLists.txt`:

```cmake
# CMakeLists.txt for maximum size reduction:
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)  # LTO
set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)
add_compile_options(-Os)
```

---

## Notes

- Bloaty can diff two binaries: `bloaty new -- old` shows what changed.
- Use `-d compileunits` to find which `.cpp` files generate the most code.
- Template-heavy code (STL, Boost) often dominates binary size.
- Debug info (`-g`) bloats file size but not VM size (not loaded at runtime).
- `strip` removes debug symbols; use `objcopy --only-keep-debug` to save them separately.
