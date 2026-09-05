# Use static analysis tools (clang-tidy, cppcheck) as part of testing

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Static analysis** examines code without executing it, catching bugs, style violations, and undefined behavior at compile time. It complements dynamic testing (unit tests, sanitizers) by finding issues that may not manifest in test runs. clang-tidy and cppcheck are the two most widely used open-source C++ static analyzers.

Think of static analysis as a second set of eyes that never gets tired. It can spot a use-after-move, a narrowing conversion, or a missing `override` keyword consistently across every file in your project - things that are easy to miss in code review. The key insight is that it works without running the code, so it catches bugs in paths that your tests may never exercise.

### Tool Comparison

The two tools are genuinely complementary - they use different internal representations and catch different categories of problems. Running both together is better than either alone.

| Feature | clang-tidy | cppcheck | clang static analyzer |
| --- | --- | --- | --- |
| **Based on** | Clang AST | Own parser | Clang CFG |
| **Checks** | 400+ (style, bugs, modernize, performance) | 100+ (bugs, undefined behavior) | Deep path-sensitive analysis |
| **Auto-fix** | Yes (`--fix`) | No | No |
| **Speed** | Moderate (needs compile_commands.json) | Fast (standalone) | Slow (symbolic execution) |
| **False positives** | Low-moderate | Low | Very low |
| **C++ standard** | Full C++23 via Clang | Good C++20 support | Full via Clang |
| **Configuration** | `.clang-tidy` file | `cppcheck` flags/suppressions | `scan-build` wrapper |

---

## Self-Assessment

### Q1: Set up clang-tidy with a project-specific configuration

**Answer:**

The `.clang-tidy` file lives at your project root and is picked up automatically. The strategy shown here is "start with everything disabled, then opt into specific check families" - this avoids being flooded with warnings on an existing codebase and lets you add categories incrementally.

```yaml
# === .clang-tidy (project root) ===
---
Checks: >
  -*,
  bugprone-*,
  cppcoreguidelines-*,
  modernize-*,
  performance-*,
  readability-*,
  -modernize-use-trailing-return-type,
  -readability-magic-numbers,
  -cppcoreguidelines-avoid-magic-numbers,
  -readability-identifier-length

WarningsAsErrors: >
  bugprone-*,
  performance-move-const-arg,
  modernize-use-nullptr

HeaderFilterRegex: 'src/.*\.h$'

CheckOptions:

  - key: readability-identifier-naming.ClassCase

    value: CamelCase

  - key: readability-identifier-naming.FunctionCase

    value: lower_case

  - key: readability-identifier-naming.VariableCase

    value: lower_case

  - key: readability-identifier-naming.PrivateMemberSuffix

    value: '_'

  - key: performance-unnecessary-value-param.AllowedTypes

    value: 'std::shared_ptr;std::unique_ptr'
...
```

Integrating clang-tidy into your CMake build means it runs automatically as part of compilation. The two options below show per-target and global integration.

```cmake
# === CMakeLists.txt integration ===

# Export compile_commands.json (required by clang-tidy)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Option 1: Per-target clang-tidy
set_target_properties(my_lib PROPERTIES
    CXX_CLANG_TIDY "clang-tidy;-p;${CMAKE_BINARY_DIR}"
)

# Option 2: Global (all targets)
set(CMAKE_CXX_CLANG_TIDY
    clang-tidy
    -p ${CMAKE_BINARY_DIR}
    --warnings-as-errors=bugprone-*
)
```

You can also run it manually or on changed files only. The changed-files approach is particularly useful in CI where you only want to check what was actually modified in a pull request.

```bash
# Run manually
clang-tidy -p build src/*.cpp --fix

# Run on changed files only (faster in CI)
git diff --name-only HEAD~1 -- '*.cpp' '*.h' | \
    xargs clang-tidy -p build

# Suppress false positive inline
// NOLINTBEGIN(bugprone-narrowing-conversions)
float f = static_cast<float>(some_double);  // Intentional
// NOLINTEND(bugprone-narrowing-conversions)

// Single line suppression:
int x = 42;  // NOLINT(readability-magic-numbers)
```

Use `NOLINT` sparingly and always leave a comment explaining why. A suppression without a reason is a future maintenance problem.

### Q2: Set up cppcheck and combine with clang-tidy

**Answer:**

cppcheck can run directly on source files without needing a compilation database, which makes it fast and easy to integrate. For the most accurate results, point it at your `compile_commands.json` so it sees the same include paths and defines that your compiler does.

```bash
# === cppcheck basic usage ===
cppcheck --enable=all \
    --std=c++20 \
    --suppress=missingIncludeSystem \
    --error-exitcode=1 \
    --inline-suppr \
    --project=build/compile_commands.json \
    --output-file=cppcheck_report.txt

# With XML output for CI integration
cppcheck --enable=all \
    --xml --xml-version=2 \
    --project=build/compile_commands.json \
    2> cppcheck_results.xml
```

A custom CMake target lets you run both tools in one command, which is convenient for local development.

```cmake
# === Custom CMake target for all static analysis ===

find_program(CLANG_TIDY clang-tidy)
find_program(CPPCHECK cppcheck)

add_custom_target(static-analysis
    COMMAND ${CLANG_TIDY}
        -p ${CMAKE_BINARY_DIR}
        ${PROJECT_SOURCE_DIR}/src/*.cpp
    COMMAND ${CPPCHECK}
        --enable=all
        --std=c++20
        --error-exitcode=1
        --suppress=missingIncludeSystem
        --project=${CMAKE_BINARY_DIR}/compile_commands.json
    COMMENT "Running static analysis..."
)
```

The GitHub Actions workflow below runs both tools on every push and pull request. Block PRs on new violations from day one - it is much harder to enforce this retroactively.

```yaml
# === GitHub Actions CI ===
name: Static Analysis
on: [push, pull_request]
jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Install tools

        run: sudo apt-get install -y clang-tidy cppcheck

      - name: Configure (generate compile_commands.json)

        run: cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

      - name: clang-tidy

        run: |
          find src -name '*.cpp' | \
            xargs clang-tidy -p build --warnings-as-errors='*'

      - name: cppcheck

        run: |
          cppcheck --enable=all --std=c++20 \
            --error-exitcode=1 \
            --suppress=missingIncludeSystem \
            --project=build/compile_commands.json
```

### Q3: Show key clang-tidy checks that catch real bugs

**Answer:**

The examples below are not hypothetical edge cases - these are patterns that appear in real codebases and cause real bugs. Each one is the kind of mistake that compiles without warning but misbehaves at runtime.

```cpp
// === bugprone-use-after-move ===
std::string s = "hello";
auto s2 = std::move(s);
std::cout << s;  // WARNING: use after move

// === bugprone-dangling-handle ===
std::string_view sv = std::string("temporary");  // DANGLING!

// === performance-move-const-arg ===
void sink(std::string s);
const std::string cs = "hello";
sink(std::move(cs));  // WARNING: moving const, falls back to copy

// === modernize-use-nullptr ===
int* p = 0;   // WARNING: use nullptr instead
int* q = NULL; // WARNING: use nullptr instead

// === modernize-use-override ===
class Base { virtual void f(); };
class Derived : public Base {
    void f() {}  // WARNING: add 'override'
};

// === bugprone-narrowing-conversions ===
double d = 3.14;
int i = d;  // WARNING: narrowing conversion

// === performance-unnecessary-copy-initialization ===
const auto& vec = get_large_vector();
auto copy = vec;  // WARNING: unnecessary copy; use const auto&

// === cppcoreguidelines-slicing ===
struct Base2 { int x; virtual ~Base2() = default; };
struct Derived2 : Base2 { int y; };
void take_by_value(Base2 b) {}  // WARNING: slicing
Derived2 d2;
take_by_value(d2);  // Object sliced!

// === bugprone-exception-escape ===
void fn() noexcept {
    throw std::runtime_error("oops");  // WARNING: exception escapes noexcept
}
```

The `bugprone-dangling-handle` and `bugprone-use-after-move` checks are particularly valuable because these bugs produce undefined behavior that can be intermittent and hard to reproduce in testing. Static analysis catches them deterministically at compile time.

---

## Notes

- **Always use `CMAKE_EXPORT_COMPILE_COMMANDS=ON`** - both tools need it for accurate analysis.
- clang-tidy's `--fix` can automatically modernize code (use-nullptr, use-override, etc.).
- Use `// NOLINT` sparingly and always document the reason.
- cppcheck catches different bugs than clang-tidy - use both for defense in depth.
- `clang-tidy` checks are prefixed by category: `bugprone-`, `modernize-`, `performance-`, `readability-`.
- Start strict (`WarningsAsErrors` for `bugprone-*`), relax only for documented false positives.
- Static analysis is a CI gate, not a developer inconvenience - block PRs on new violations.
- Consider also: `include-what-you-use` (iwyu) for header hygiene.
