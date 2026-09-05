# Use clang-format to enforce consistent code style in a team

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/ClangFormat.html>  

---

## Topic Overview

clang-format reads a `.clang-format` file from your project root and applies those formatting rules automatically. For a team, the real value is that style stops being a subjective discussion and becomes a mechanical check. Nobody argues about brace placement in code review when the formatter settles it before the PR is even created.

The typical team workflow looks like this:

```cpp
Workflow:
  Developer writes code
       │
  clang-format -i *.cpp  (pre-commit hook or save action)
       │
  git push
       │
  CI: clang-format --dry-run --Werror
  ┌───┴───┐
 PASS    FAIL -> "Fix formatting before merge"
```

The formatter runs locally (either on save or as a pre-commit hook) and again in CI. If code that was not formatted somehow makes it into a PR, CI catches it.

---

## Self-Assessment

### Q1: Create a `.clang-format` file and apply it across a codebase

The `BasedOnStyle` setting is your starting point. It sets all 120+ formatting options to a known baseline, so you only need to list the options where you want to override the defaults. Starting from `LLVM` or `Google` and tweaking a few settings is far less work than specifying everything from scratch:

```yaml
# .clang-format
---
BasedOnStyle: LLVM

# Team customizations:
IndentWidth: 4
TabWidth: 4
UseTab: Never
ColumnLimit: 120
AccessModifierOffset: -4
AllowShortFunctionsOnASingleLine: Empty
AlwaysBreakTemplateDeclarations: Yes
BinPackArguments: false
BinPackParameters: false
BreakBeforeBraces: Attach
NamespaceIndentation: None
PointerAlignment: Left
SortIncludes: CaseInsensitive
SpaceAfterTemplateKeyword: true
Standard: c++20
---
```

Apply the formatter to the whole codebase in one pass:

```bash
# Apply to entire codebase:
find src include tests -name '*.cpp' -o -name '*.h' | xargs clang-format -i

# Or use git ls-files to only format tracked files:
git ls-files '*.cpp' '*.h' | xargs clang-format -i

# Preview changes without modifying:
clang-format --dry-run -Werror main.cpp
```

Here is a concrete before/after to show what the formatter actually does to messy code:

```cpp
namespace app{class Widget {int x ;double   y;
public:
Widget( int a , double b ):x( a ),y( b ){}
void   print (
) const;
};}
```

After formatting, everything is consistently indented and spaced:

```cpp
namespace app {
class Widget {
    int x;
    double y;

public:
    Widget(int a, double b) : x(a), y(b) {}
    void print() const;
};
}  // namespace app
```

The closing `}  // namespace app` comment is generated automatically by clang-format - a nice touch for long namespace blocks.

### Q2: CI step to enforce formatting

A `.clang-format` file without a CI check is just a suggestion. The check is what makes the style rule mandatory. This GitHub Actions workflow fails the build if any committed file differs from what the formatter would produce:

```yaml
# .github/workflows/format-check.yml
name: Formatting
on: [push, pull_request]

jobs:
  check-format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check clang-format
        run: |
          # Format all C++ files
          git ls-files '*.cpp' '*.h' '*.hpp' | xargs clang-format -i

          # Check if anything changed
          if ! git diff --quiet; then
            echo "::error::Code is not formatted. Run clang-format."
            git diff --stat
            git diff  # show exact changes
            exit 1
          fi
          echo "All files correctly formatted."
```

You can also enforce formatting before the push ever happens with a git pre-commit hook. The hook reformats staged files and re-stages them so the commit always contains formatted code:

```bash
#!/bin/bash
# .git/hooks/pre-commit
FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(cpp|h|hpp)$')
if [ -n "$FILES" ]; then
    echo "$FILES" | xargs clang-format -i
    echo "$FILES" | xargs git add  # re-stage formatted files
fi
```

The pre-commit hook is a good developer convenience, but the CI check is the authoritative gate. Both together give you fast feedback locally and reliable enforcement on every push.

### Q3: `BasedOnStyle` vs per-option overrides

The reason `BasedOnStyle` matters is that clang-format has well over 100 options, and all of them have defaults. Without `BasedOnStyle`, every unspecified option falls back to the LLVM defaults, which may not match what you want. With `BasedOnStyle`, you inherit a complete, consistent starting point and only override what you explicitly disagree with.

```yaml
# BasedOnStyle sets ALL options to a predefined style's defaults
# Per-option overrides then modify specific options

# Example: Start from Google style, customize a few things
---
BasedOnStyle: Google          # Sets ~120 options to Google's values
IndentWidth: 4                # Override: Google uses 2, we want 4
ColumnLimit: 100              # Override: Google uses 80
AllowShortIfStatementsOnASingleLine: Never  # Override
---

# Without BasedOnStyle, you must specify EVERY option
# clang-format has 120+ options; missing ones get LLVM defaults

# Available base styles:
#   LLVM, Google, Chromium, Mozilla, WebKit, Microsoft, GNU
```

The key differences between the popular base styles are summarized here:

| Base Style | Indent | Braces | Column Limit |
| --- | --- | --- | --- |
| LLVM | 2 | Attach | 80 |
| Google | 2 | Attach | 80 |
| Chromium | 2 | Attach | 80 |
| Mozilla | 2 | Linux (next line for functions) | 80 |
| WebKit | 4 | Attach | 0 (no limit) |
| Microsoft | 4 | Allman (next line) | 120 |

To explore a style before committing to it, dump its full configuration and compare:

```bash
# Generate the full config for a base style:
clang-format --dump-config --style=google > .clang-format
# Then edit the file to customize

# Test different styles quickly:
clang-format --style=google main.cpp | diff - <(clang-format --style=mozilla main.cpp)
```

---

## Notes

- Place `.clang-format` in the project root; clang-format walks up the directory tree from each file until it finds the config.
- Use `.clang-format-ignore` to exclude files you do not want formatted, such as generated sources or vendored third-party code.
- Use `// clang-format off` and `// clang-format on` for sections that have intentional manual formatting (lookup tables, matrices, alignment-sensitive macros).
- Integrate with editors: VS Code uses `formatOnSave`, Vim uses `:ClangFormat`, and Emacs has a clang-format plugin.
- If you are starting a new project, adopting an established base style like Google or LLVM with minimal overrides reduces the ongoing maintenance burden.
