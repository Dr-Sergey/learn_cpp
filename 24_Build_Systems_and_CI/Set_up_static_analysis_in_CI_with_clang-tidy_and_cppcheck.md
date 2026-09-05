# Set up static analysis in CI with clang-tidy and cppcheck

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/extra/clang-tidy/>  

---

## Topic Overview

Static analysis finds bugs, style violations, and potential security issues **without running the code**. Running clang-tidy and cppcheck in CI catches problems before they reach code review, which is much cheaper than finding them in production. Both tools complement each other well: clang-tidy is AST-based (deep analysis using the full Clang frontend), while cppcheck uses its own parser and finds different categories of bugs. Running both together gives you broader coverage than either tool alone.

### Tool Comparison

| Feature | clang-tidy | cppcheck |
| --- | --- | --- |
| Parser | Clang AST (full C++ parser) | Custom parser |
| Checks | 400+ checks (modernize, bugprone, performance, ...) | Type errors, leaks, null ptrs, style |
| Configuration | `.clang-tidy` file | `--enable=all`, XML config |
| Speed | Moderate (full parse) | Fast (lightweight) |
| False positive rate | Low-medium | Medium |
| Fix-it hints | Yes (`--fix`) | No |
| Requires compile_commands.json | Yes | Optional (recommended) |

### Integration Options

There are three main ways to run clang-tidy. Each has different trade-offs for speed and integration:

```cpp
Option 1: CMAKE_CXX_CLANG_TIDY (runs during build)
  cmake -DCMAKE_CXX_CLANG_TIDY="clang-tidy;-warnings-as-errors=*"

Option 2: run-clang-tidy.py (runs separately, parallelized)
  run-clang-tidy -p build/ -j$(nproc)

Option 3: CI step with compile_commands.json
  cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
  clang-tidy -p build/ src/*.cpp
```

---

## Self-Assessment

### Q1: Add a CI step that runs clang-tidy on all changed files using run-clang-tidy.py

**Answer:**

The first thing you need is a `.clang-tidy` configuration file at the project root. This tells clang-tidy which checks to run and how to report them. The example below starts from a clean slate (`-*` disables all checks) and then enables specific categories.

```yaml
# .clang-tidy file content:
Checks: >
  -*,
  bugprone-*,
  modernize-*,
  performance-*,
  readability-*,
  cppcoreguidelines-*,
  -modernize-use-trailing-return-type,
  -readability-identifier-length

WarningsAsErrors: >
  bugprone-*,
  performance-*

HeaderFilterRegex: '^(src|include)/.*'

CheckOptions:
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
  - key: readability-identifier-naming.FunctionCase
    value: camelBack
```

With the config file in place, here is the GitHub Actions workflow that runs clang-tidy in CI. It supports two modes: running on all files (thorough) or only on changed files (faster for large projects):

```yaml
# .github/workflows/static-analysis.yml
name: Static Analysis
on: [pull_request]

jobs:
  clang-tidy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for git diff

      - name: Install clang-tidy
        run: |
          sudo apt-get update
          sudo apt-get install -y clang-tidy-17 clang-tools-17

      - name: Generate compile_commands.json
        run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_CXX_STANDARD=20

      # Option A: Run on ALL files
      - name: Run clang-tidy (all files)
        run: run-clang-tidy-17 -p build/ -j$(nproc)

      # Option B: Run only on CHANGED files (faster for large projects)
      - name: Get changed C++ files
        id: changed
        run: |
          FILES=$(git diff --name-only --diff-filter=ACMR origin/main -- '*.cpp' '*.h' '*.hpp' | tr '\n' ' ')
          echo "files=$FILES" >> $GITHUB_OUTPUT

      - name: Run clang-tidy on changed files
        if: steps.changed.outputs.files != ''
        run: |
          clang-tidy-17 -p build/ ${{ steps.changed.outputs.files }} \
            --warnings-as-errors='bugprone-*,performance-*'
```

**Explanation:**

- `run-clang-tidy` with `-j` parallelizes across all source files - it's substantially faster than running `clang-tidy` in a shell loop.
- `CMAKE_EXPORT_COMPILE_COMMANDS=ON` generates `compile_commands.json`, which tells clang-tidy the exact compiler flags used for each file. Without this, clang-tidy doesn't know your include paths, standard version, or defines.
- Running only on changed files keeps PR feedback fast for large monorepos where a full run would take too long.
- `--warnings-as-errors` makes specific check categories fail the CI job, turning them from suggestions into hard requirements.

### Q2: Configure cppcheck --enable=all and interpret its XML output in a CI report

**Answer:**

cppcheck is fast and finds a different class of bugs than clang-tidy - especially value-flow analysis issues like null pointer dereferences and memory leaks. Running both tools together means you catch more bugs for the same CI cost.

```bash
# ======= Install cppcheck =======
sudo apt-get install -y cppcheck
# Or build from source for latest version:
# git clone https://github.com/danmar/cppcheck.git
# cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j$(nproc)

# ======= Run cppcheck with XML output =======
cppcheck \
    --enable=all \
    --suppress=missingIncludeSystem \
    --std=c++20 \
    --language=c++ \
    --project=build/compile_commands.json \
    --xml \
    --output-file=cppcheck-report.xml \
    2>&1

# ======= Sample XML output =======
# <?xml version="1.0" encoding="UTF-8"?>
# <results version="2">
#   <errors>
#     <error id="nullPointer" severity="error"
#            msg="Null pointer dereference: ptr"
#            verbose="Null pointer dereference"
#            file0="src/main.cpp">
#       <location file="src/main.cpp" line="42" column="5"/>
#     </error>
#     <error id="memleak" severity="error"
#            msg="Memory leak: buffer"
#            file0="src/parser.cpp">
#       <location file="src/parser.cpp" line="88" column="5"/>
#     </error>
#   </errors>
# </results>
```

Here is the full CI integration that runs cppcheck, converts the output to HTML, and uploads it as an artifact so you can browse the results:

```yaml
# CI integration with HTML report
name: cppcheck
on: [pull_request]

jobs:
  cppcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install cppcheck
        run: sudo apt-get install -y cppcheck

      - name: Generate compile commands
        run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

      - name: Run cppcheck
        run: |
          cppcheck \
            --enable=all \
            --suppress=missingIncludeSystem \
            --std=c++20 \
            --project=build/compile_commands.json \
            --xml \
            --output-file=cppcheck-report.xml \
            --error-exitcode=1 \
            2>&1

      - name: Convert to HTML
        if: always()
        run: |
          cppcheck-htmlreport \
            --file=cppcheck-report.xml \
            --report-dir=cppcheck-html \
            --source-dir=.

      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: cppcheck-report
          path: cppcheck-html/

      # Parse XML for PR comment
      - name: Check for errors
        if: always()
        run: |
          ERRORS=$(grep -c '<error ' cppcheck-report.xml || true)
          echo "cppcheck found $ERRORS issues"
          if [ "$ERRORS" -gt 0 ]; then
            echo "::error::cppcheck found $ERRORS issues"
          fi
```

**cppcheck flags explained:**

- `--enable=all` enables all check categories: style, performance, portability, information, and warning. You can narrow this down if certain categories produce too many false positives for your codebase.
- `--suppress=missingIncludeSystem` suppresses noisy "missing system include" warnings that aren't actionable.
- `--error-exitcode=1` makes cppcheck return exit code 1 if it finds errors, which fails the CI job. Without this, cppcheck always exits 0 and the CI job passes even when there are problems.
- `--project=compile_commands.json` uses the exact compile flags from CMake, giving cppcheck the same include paths and defines your actual build uses.

### Q3: Use cmake -DCMAKE_CXX_CLANG_TIDY for integrated analysis during build

**Answer:**

Integrating clang-tidy directly into the build via `CMAKE_CXX_CLANG_TIDY` is convenient because you don't need a separate analysis step - every file that compiles also gets analyzed. The trade-off is that it slows the build significantly, so you'll want to keep it off by default and only enable it in CI.

```cmake
# ======= CMakeLists.txt - integrated clang-tidy =======
cmake_minimum_required(VERSION 3.21)
project(MyProject CXX)

# Option to enable clang-tidy during build
option(ENABLE_CLANG_TIDY "Run clang-tidy during build" OFF)

if(ENABLE_CLANG_TIDY)
    find_program(CLANG_TIDY_EXE NAMES clang-tidy clang-tidy-17)
    if(CLANG_TIDY_EXE)
        set(CMAKE_CXX_CLANG_TIDY
            ${CLANG_TIDY_EXE}
            -warnings-as-errors=bugprone-*,performance-*
            --extra-arg=-std=c++20
        )
        message(STATUS "clang-tidy enabled: ${CLANG_TIDY_EXE}")
    else()
        message(WARNING "clang-tidy requested but not found!")
    endif()
endif()

add_library(mylib src/mylib.cpp)
target_include_directories(mylib PUBLIC include/)
target_compile_features(mylib PUBLIC cxx_std_20)

add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE mylib)
```

Here is how you enable it for CI and what the output looks like during a build:

```bash
# Build with clang-tidy integrated:
cmake -B build \
    -DENABLE_CLANG_TIDY=ON \
    -DCMAKE_BUILD_TYPE=Release

cmake --build build --parallel $(nproc) 2>&1

# Output during compilation:
# [1/4] Building CXX object src/mylib.cpp.o
# src/mylib.cpp:15:5: warning: use 'nullptr' instead of '0' [modernize-use-nullptr]
#     int* p = 0;
#              ^
#              nullptr
# src/mylib.cpp:22:3: error: use of 'push_back' with temporary [performance-move-const-arg]
#   vec.push_back(std::move(str));
#   ^
# FAILED: src/mylib.cpp.o

# ======= From command line without CMakeLists.txt changes =======
cmake -B build \
    -DCMAKE_CXX_CLANG_TIDY="clang-tidy;-warnings-as-errors=*" \
    -DCMAKE_BUILD_TYPE=Release
```

You can disable clang-tidy on a per-target basis, which is useful for vendor or third-party code that you don't want to analyze:

```cmake
# Only run clang-tidy on your own targets, not vendored libs
add_library(vendor_lib STATIC vendor/lib.cpp)
set_target_properties(vendor_lib PROPERTIES CXX_CLANG_TIDY "")  # Disable

add_library(mylib src/mylib.cpp)
# mylib inherits CMAKE_CXX_CLANG_TIDY (enabled)
```

**Trade-offs:**

- `CMAKE_CXX_CLANG_TIDY` runs analysis on every compilation - this slows the build by 2-4x because clang-tidy does a full parse on top of the normal compilation pass.
- This is a reasonable cost in CI (catch everything), but too slow for local iterative development where you're recompiling frequently.
- Use the `ENABLE_CLANG_TIDY` option to keep it off by default and on in CI. That way developers don't have to pay the cost unless they opt in.
- The `--fix` flag can auto-fix issues, but don't use it in CI - it modifies source files, which would create a messy interaction with the checkout and diff.

---

## Notes

- **`compile_commands.json`** is required for clang-tidy - always set `CMAKE_EXPORT_COMPILE_COMMANDS=ON` in any build where you want to run analysis tools. Without it, clang-tidy can't find your headers.
- **`.clang-tidy`** file in the project root configures checks for the whole project. You can also put `.clang-tidy` files in subdirectories to override the root config for specific parts of the codebase.
- **NOLINTBEGIN/NOLINTEND** suppresses clang-tidy for specific code blocks, which is useful for generated code or intentional patterns that trigger false positives.
- **`iwyu`** (include-what-you-use) is another useful static analysis tool that finds unnecessary `#include` directives, which can speed up compilation.
- **Combine both tools:** clang-tidy catches modernization issues and AST-level bugs; cppcheck catches value-flow bugs and leaks. They genuinely find different things, so running both is worth the CI cost.
