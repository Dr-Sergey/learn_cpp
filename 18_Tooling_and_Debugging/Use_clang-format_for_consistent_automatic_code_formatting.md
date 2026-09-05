# Use clang-format for consistent automatic code formatting

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/ClangFormat.html>  

---

## Topic Overview

clang-format automatically reformats C++ source code according to a style configuration file. The real benefit is not that the output looks any particular way - it is that the decision is automated and consistent. Once you have a `.clang-format` file and your editor applies it on save, nobody on the team ever argues about brace placement or indentation again. Those discussions simply disappear because the tool settles them deterministically.

Here are the three most common ways to invoke clang-format:

```bash
# Format a file in-place:
clang-format -i main.cpp

# Check without modifying (CI mode):
clang-format --dry-run --Werror main.cpp

# Format all C++ files recursively:
find . -name '*.cpp' -o -name '*.h' | xargs clang-format -i
```

The `--dry-run --Werror` combination is what you use in CI: it exits with a non-zero code if any file would be changed, which fails the build without actually modifying any files.

---

## Self-Assessment

### Q1: Create a `.clang-format` configuration file

Place the configuration file in your project root. clang-format searches upward from the file being formatted until it finds a `.clang-format` file, so a single root config covers the whole project.

The `BasedOnStyle` key is the most important line because it sets defaults for all 120+ style options at once. You only need to list the options where you disagree with the base style:

```yaml
# .clang-format (place in project root)
---
Language: Cpp
BasedOnStyle: Google  # Start from a well-known base

# Customizations:
IndentWidth: 4
ColumnLimit: 100
AccessModifierOffset: -4
AllowShortFunctionsOnASingleLine: Inline
AlwaysBreakTemplateDeclarations: Yes
BreakBeforeBraces: Attach
PointerAlignment: Left
SortIncludes: CaseInsensitive
IncludeBlocks: Regroup
IncludeCategories:
  - Regex:    '<[^/]+>'      # system headers
    Priority: 3
  - Regex:    '<.+>'         # third-party headers
    Priority: 2
  - Regex:    '".*"'         # project headers
    Priority: 1
---
```

To see what this actually does, here is a deliberately messy file:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
#include<iostream>
#include "myheader.h"
#include<vector>
class Widget{
    int x;double y;
public:
Widget(int a,double b):x(a),y(b){}
void print(){std::cout<<x<<" "<<y<<"\n";}
};
int main(){Widget w(1,2.0);w.print();}
```

After running `clang-format -i main.cpp`, the output is consistently structured:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
#include "myheader.h"

#include <iostream>
#include <vector>

class Widget {
    int x;
    double y;

public:
    Widget(int a, double b) : x(a), y(b) {}
    void print() { std::cout << x << " " << y << "\n"; }
};

int main() {
    Widget w(1, 2.0);
    w.print();
}
```

Notice that the includes were also reordered and grouped: project headers first (from the `IncludeCategories` config), then system headers. clang-format does not just fix whitespace - it restructures includes according to your policy.

### Q2: CI step to enforce formatting

Adding a formatting check to CI is what gives the `.clang-format` file its teeth. Without a CI check, developers can skip the formatter, and the configuration file becomes irrelevant over time. With a CI check, the formatter is effectively mandatory.

```yaml
# .github/workflows/format.yml
name: Format Check

on: [push, pull_request]

jobs:
  clang-format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install clang-format
        run: sudo apt-get install -y clang-format-17
      - name: Check formatting
        run: |
          find . -name '*.cpp' -o -name '*.h' | \
            xargs clang-format-17 --dry-run --Werror
        # --dry-run: don't modify files
        # --Werror: exit with error if changes needed
        # If formatting is wrong, CI fails with a diff-like output
```

An alternative approach formats the files and then checks for a diff, which produces more readable output when the check fails:

```bash
# Format and check for diff
find . -name '*.cpp' -o -name '*.h' | xargs clang-format-17 -i
git diff --exit-code
# If there's a diff, files weren't formatted -> fail CI
```

The `git diff --exit-code` approach is particularly nice because the CI output shows exactly which lines would change, making it easy for the developer to understand what needs fixing.

### Q3: Handling macro-heavy code and `// clang-format off`

clang-format does not understand the semantics of macros. It formats them syntactically, which sometimes destroys intentional alignment in DSL-like macro invocations, lookup tables, or matrix literals. The escape hatch is to tell the formatter to leave a section alone:

```cpp
#include <iostream>

// clang-format mangles certain macro patterns:

// Example: DSL-like macros with custom formatting
// clang-format off
#define REGISTER_TEST(name, func) \
    static TestEntry entry_##name = { \
        .test_name = #name,           \
        .test_func = func,            \
        .enabled   = true             \
    }
// clang-format on

// Lookup tables with intentional alignment:
// clang-format off
const int OPCODES[] = {
    0x00, 0x01, 0x02, 0x03,
    0x10, 0x11, 0x12, 0x13,
    0x20, 0x21, 0x22, 0x23,
};
// clang-format on

// Matrix literals:
// clang-format off
const float IDENTITY_4X4[] = {
    1.0f, 0.0f, 0.0f, 0.0f,
    0.0f, 1.0f, 0.0f, 0.0f,
    0.0f, 0.0f, 1.0f, 0.0f,
    0.0f, 0.0f, 0.0f, 1.0f,
};
// clang-format on

int main() {
    std::cout << "Opcode[0]: " << OPCODES[0] << '\n';
    std::cout << "Identity[0]: " << IDENTITY_4X4[0] << '\n';
}
// Expected output:
// Opcode[0]: 0
// Identity[0]: 1
```

Use `// clang-format off` sparingly - the goal is to let the formatter do its job on as much of the codebase as possible. Good candidates for disabling the formatter are:

- Lookup tables with intentional column alignment
- Macro definitions with custom backslash alignment
- Matrix and vector literals
- Assembly blocks
- Embedded binary data or generated code

---

## Notes

- Generate a starting config from a known style: `clang-format --dump-config --style=google > .clang-format`, then edit the parts you want to change.
- Popular base styles are `LLVM`, `Google`, `Chromium`, `Mozilla`, `WebKit`, and `Microsoft`.
- In VS Code, install the "C/C++" extension and set `C_Cpp.clang_format_style` to `file` to use your project's config.
- Use `.clang-format-ignore` to exclude files you do not want formatted, such as generated code or vendored third-party sources.
- clang-format can also format Java, JavaScript, JSON, and Proto files - the same tool works across your whole repo if you use those languages.
