# Use compiler diagnostics: -ftemplate-backtrace-limit and concept diagnostics

**Category:** Tooling and Debugging  
**Standard:** C++20  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html>  

---

## Topic Overview

If you have ever compiled a piece of code that uses templates and then stared at a wall of error text that stretches past the terminal window, you already know the problem this section addresses. C++ template errors are notoriously verbose because every error comes with a full chain of instantiation notes - the compiler faithfully telling you how it got from your code to the place where something went wrong.

There are compiler flags that let you control how much of that chain you see. The key ones are:

| Flag | Compiler | Effect |
| --- | --- | --- |
| `-ftemplate-backtrace-limit=N` | GCC/Clang | Show N levels of template instantiation (0 = all) |
| `-fconcepts-diagnostics-depth=N` | GCC | Show N levels of concept constraint failures |
| `-fdiagnostics-show-template-tree` | Clang | Tree view of template type differences |

The default limit is 10, which truncates the backtrace in complex cases. Setting it to 0 turns off the limit entirely. You will see much more output, but that output contains the full path from your call site down to the broken instantiation, which is often exactly what you need to locate the root cause.

---

## Self-Assessment

### Q1: `-ftemplate-backtrace-limit=0` for full instantiation chain

Here is a common scenario: you try to sort a container whose element type does not satisfy `operator<`. The error comes from deep inside `<algorithm>`, and by default you only see a few levels of it. To understand how the compiler got there, you want the full chain.

```cpp
#include <vector>
#include <algorithm>
#include <list>

struct Widget {
    int x;
    // No operator< defined!
};

int main() {
    std::vector<Widget> v = {{3}, {1}, {2}};
    std::sort(v.begin(), v.end());  // ERROR deep inside <algorithm>
}
// Default compilation (limit=10):
//   error: no match for 'operator<' in ...
//   note: ... 5 more levels truncated ...
//
// With -ftemplate-backtrace-limit=0:
//   Shows the FULL chain:
//   main() -> std::sort() -> __sort() -> __move_median_to_first()
//          -> __comp() -> operator<(Widget, Widget) <- NOT FOUND
//
// The full chain helps you understand WHERE the error originates.
```

The default output tells you that `operator<` was not found somewhere inside the standard library. The full-backtrace output shows you the complete call chain and makes it obvious which step required the comparison. When you are debugging, that context is the difference between "something is wrong" and "I know exactly what to fix."

```bash
# Compile with unlimited backtrace:
g++ -std=c++20 -ftemplate-backtrace-limit=0 sort_error.cpp

# Clang equivalent:
clang++ -std=c++20 -ftemplate-backtrace-limit=0 sort_error.cpp

# CMake:
target_compile_options(myapp PRIVATE -ftemplate-backtrace-limit=0)
```

You would typically add this flag during active debugging and remove it from your default build so that normal compilations stay readable.

### Q2: SFINAE error vs concept constraint violation

This example shows one of the most practical reasons to prefer concepts over SFINAE in C++20. Both approaches restrict a template to certain types, but the quality of the error message they produce is vastly different.

```cpp
#include <type_traits>
#include <concepts>
#include <iostream>
#include <string>

// ========= SFINAE version =========
template<typename T,
         typename = std::enable_if_t<std::is_arithmetic_v<T>>>
T add_sfinae(T a, T b) { return a + b; }

// Error when called with std::string:
//   add_sfinae(std::string("a"), std::string("b"));
// GCC SFINAE error:
//   error: no matching function for call to 'add_sfinae(std::string, std::string)'
//   note: candidate: template<class T, class> T add_sfinae(T, T)
//   note: template argument deduction/substitution failed:
//   note: ... std::enable_if_t<false> is not a type ...
//   ^^^^^^^^^^ CRYPTIC! What constraint failed? Why?

// ========= Concept version =========
template<std::integral T>    // clear constraint
T add_concept(T a, T b) { return a + b; }

// Error when called with std::string:
//   add_concept(std::string("a"), std::string("b"));
// GCC concept error:
//   error: no matching function for call to 'add_concept(std::string, std::string)'
//   note: the required expression 'std::integral<T>' is not satisfied
//   note: because 'std::is_integral_v<std::string>' evaluated to 'false'
//   ^^^^^^^^^^ CLEAR! Tells you exactly which constraint and why.

int main() {
    std::cout << add_sfinae(1, 2) << '\n';    // OK
    std::cout << add_concept(3, 4) << '\n';   // OK
}
// Expected output:
// 3
// 7
```

The reason SFINAE errors are so hard to read is that SFINAE works by silently discarding candidates when substitution fails. The compiler does tell you about the failure, but it has to express it in terms of internal substitution mechanics - `enable_if_t<false>` is not a type - rather than in terms of your intent. Concepts, on the other hand, are designed to carry human-readable meaning. When a concept check fails, the compiler can say exactly which named constraint was not satisfied and why. For anyone reading the error message six months from now, that is a huge improvement.

### Q3: `-fconcepts-diagnostics-depth` for nested constraints

Concepts can be composed: one concept can depend on another, which depends on another still. When you have a constraint failure inside a chain like that, the default diagnostic depth of 1 tells you only that the outermost concept was not satisfied. To understand which inner constraint broke, you need to increase the depth.

```cpp
#include <concepts>
#include <ranges>
#include <vector>
#include <iostream>

// Deeply nested concept:
template<typename T>
concept Printable = requires(T t, std::ostream& os) {
    { os << t } -> std::same_as<std::ostream&>;
};

template<typename R>
concept PrintableRange = std::ranges::range<R> &&
    Printable<std::ranges::range_value_t<R>>;

template<PrintableRange R>
void print_all(const R& range) {
    for (const auto& x : range)
        std::cout << x << ' ';
    std::cout << '\n';
}

struct NoPrint { int x; };  // no operator<<

int main() {
    std::vector<int> v = {1, 2, 3};
    print_all(v);  // OK

    // std::vector<NoPrint> np = {{1}, {2}};
    // print_all(np);  // ERROR:
    // Default depth=1: "PrintableRange<vector<NoPrint>> not satisfied"
    // With -fconcepts-diagnostics-depth=3:
    //   because 'Printable<NoPrint>' is not satisfied
    //     because the expression 'os << t' is not valid
    //       because no operator<< matches 'ostream& << NoPrint'
}
// Expected output:
// 1 2 3
```

At depth 1, the message is technically correct but practically useless - it tells you the outer constraint failed but not why. At depth 3, you see the full chain: `PrintableRange` failed because `Printable<NoPrint>` failed because there is no matching `operator<<`. That last step is the actionable information. The deeper diagnostics give you the full story.

```bash
# Increase concept diagnostic depth:
g++ -std=c++20 -fconcepts-diagnostics-depth=3 concepts.cpp

# Clang has good concept diagnostics by default
# For template diff tree (Clang only):
clang++ -std=c++20 -fdiagnostics-show-template-tree concepts.cpp
```

---

## Notes

- `-ftemplate-backtrace-limit=0` is useful for debugging but produces very long output - only turn it on when you need it.
- Concepts (C++20) provide dramatically better error messages than SFINAE, and that improvement in readability is one of the strongest practical arguments for adopting them.
- GCC's `-fconcepts-diagnostics-depth` defaults to 1; increase it when you have nested constraints and the top-level message is not enough to identify the problem.
- Clang's `-fdiagnostics-show-template-tree` shows type diffs as a tree, which is helpful when two template types are nearly identical but differ in one argument.
- Consider adding these diagnostic flags only for development builds, not for release builds where compile time matters more than error verbosity.
