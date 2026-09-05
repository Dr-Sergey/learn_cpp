# Use static analysis with clang-tidy for modernization and readability checks

**Category:** Tooling and Debugging  
**Standard:** C++11  
**Reference:** <https://clang.llvm.org/extra/clang-tidy/checks/list.html>  

---

## Topic Overview

clang-tidy is both a linter and a light static analyzer. Think of it as an automated code reviewer that catches common bugs, flags opportunities to use modern C++ idioms, and enforces naming conventions - all without you having to remember to check these things manually.

What makes clang-tidy different from a simple text-based linter is that it parses your code the same way the compiler does, so it understands types, overloads, and templates. It can therefore diagnose things a grep-based tool would miss, and it can apply fixes automatically in many cases.

The checks are organized into named categories, which makes it easy to enable only the kinds of feedback you want:

| Check Category | What It Does | Examples |
| --- | --- | --- |
| `modernize-*` | Upgrade to modern C++ | `use-nullptr`, `use-override`, `use-auto` |
| `readability-*` | Improve readability | `braces-around-statements`, `identifier-naming` |
| `bugprone-*` | Find common bugs | `use-after-move`, `string-integer-assignment` |
| `performance-*` | Performance improvements | `unnecessary-copy-initialization`, `move-const-arg` |
| `cppcoreguidelines-*` | C++ Core Guidelines | `avoid-goto`, `owning-memory` |

---

## Self-Assessment

### Q1: Run clang-tidy and interpret the output

Here's some code written in an older C++ style. It has no `override` keywords, uses `NULL` instead of `nullptr`, creates objects with raw `new`, and uses iterator-based loops. clang-tidy will flag all of these.

```cpp
// old_code.cpp - has modernization opportunities
#include <iostream>
#include <vector>
#include <memory>

class Base {
public:
    virtual void process() { std::cout << "Base\n"; }
    virtual ~Base() {}
};

class Derived : public Base {
public:
    void process() { std::cout << "Derived\n"; }  // missing override
    ~Derived() {}  // missing override
};

void old_style() {
    int* p = NULL;                    // should be nullptr
    Base* b = new Derived();          // should use smart pointer
    b->process();
    delete b;

    std::vector<int> v;
    for (std::vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
        // should use range-for
    }

    typedef int MyInt;  // should use 'using'
}

int main() { old_style(); }
```

Running clang-tidy with `modernize-*` and `readability-*` checks produces detailed diagnostics with the exact location, the problem, and often a suggested fix right in the output.

```bash
# Run with modernize and readability checks:
$ clang-tidy old_code.cpp \
    --checks='modernize-*,readability-*' \
    -- -std=c++20

# Output:
# old_code.cpp:12:10: warning: prefer using 'override' ... [modernize-use-override]
#     void process() {
#                    ^    override
#
# old_code.cpp:13:5: warning: prefer using 'override' ... [modernize-use-override]
#     ~Derived() {}
#     ^           override
#
# old_code.cpp:17:16: warning: use nullptr [modernize-use-nullptr]
#     int* p = NULL;
#              ^~~~  nullptr
#
# old_code.cpp:22:5: warning: use range-based for loop [modernize-loop-convert]
#     for (std::vector<int>::iterator
#     ^~~~
#     for (auto & elem : v)
#
# old_code.cpp:26:5: warning: use 'using' instead of 'typedef' [modernize-use-using]
#     typedef int MyInt;
#     ^~~~~~~~~~~~~~~~~  using MyInt = int;
```

Each warning tells you the check that fired (in square brackets), so you can look up the full documentation for that check if you want to understand the rationale.

### Q2: Auto-fix with `--fix`

One of the most useful features is `--fix`. For many of the modernization checks, clang-tidy can apply the fix directly to the source file. You can be selective about which checks to auto-fix - it's usually safest to fix one category at a time and review the diff before committing.

```bash
# Auto-fix specific checks:
$ clang-tidy old_code.cpp \
    --checks='modernize-use-nullptr,modernize-use-override' \
    --fix \
    -- -std=c++20

# Applied fixes:
# - NULL -> nullptr
# - Added 'override' to Derived::process() and ~Derived()
```

After running the auto-fix, the affected code looks like this. The changes are minimal and safe - the meaning is identical, just expressed in modern C++.

```cpp
class Derived : public Base {
public:
    void process() override { std::cout << "Derived\n"; }  // fixed!
    ~Derived() override {}  // fixed!
};

void old_style() {
    int* p = nullptr;  // fixed!
    // ...
}
```

```bash
# Fix all modernize checks at once:
$ clang-tidy old_code.cpp --checks='modernize-*' --fix -- -std=c++20

# Dry run (show what would be fixed, don't modify):
$ clang-tidy old_code.cpp --checks='modernize-*' --fix-notes -- -std=c++20

# Fix across entire project (with compile_commands.json):
$ run-clang-tidy -p build/ -checks='modernize-*' -fix
```

### Q3: Configure `.clang-tidy` file

Rather than specifying checks on the command line every time, you put a `.clang-tidy` file in your project root. clang-tidy auto-detects it. This also lets you tune individual check parameters - for example, you can configure the exact naming convention your project uses rather than accepting the defaults.

```yaml
# .clang-tidy - project root configuration
---
Checks: >
  -*,
  modernize-*,
  readability-*,
  bugprone-*,
  performance-*,
  -modernize-use-trailing-return-type,
  -readability-magic-numbers,
  -readability-identifier-length

WarningsAsErrors: >
  bugprone-*,
  modernize-use-override

HeaderFilterRegex: 'src/.*\.h$'

CheckOptions:
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
  - key: readability-identifier-naming.FunctionCase
    value: camelBack
  - key: readability-identifier-naming.VariableCase
    value: lower_case
  - key: readability-identifier-naming.ConstantCase
    value: UPPER_CASE
  - key: modernize-use-auto.MinTypeNameLength
    value: 10
  - key: readability-function-cognitive-complexity.Threshold
    value: 25
```

The `WarningsAsErrors` section is what gives you CI enforcement: bug-prone patterns become hard errors, while style suggestions stay as warnings. The `HeaderFilterRegex` prevents clang-tidy from flooding you with warnings from third-party headers you don't control.

```bash
# Run using .clang-tidy config (auto-detected from file):
$ clang-tidy src/*.cpp -- -std=c++20

# CI integration:
$ clang-tidy --warnings-as-errors='*' src/*.cpp -- -std=c++20
# Returns non-zero exit code on any warning -> CI fails

# CMake integration:
set(CMAKE_CXX_CLANG_TIDY
    clang-tidy
    --config-file=${CMAKE_SOURCE_DIR}/.clang-tidy
)
```

The CMake integration runs clang-tidy automatically on every compilation unit during the build, which means you catch issues immediately instead of needing a separate lint step.

---

## Notes

- Start with `modernize-*` and `bugprone-*`; add more gradually.
- `readability-identifier-naming` enforces naming conventions across the project.
- Use `// NOLINT(check-name)` to suppress specific warnings on a line.
- `run-clang-tidy` parallelizes analysis across files.
- clang-tidy requires `compile_commands.json` for accurate analysis.
