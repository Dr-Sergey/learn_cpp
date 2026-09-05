# Use Bloaty McBloatface to analyze binary size contributors

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://github.com/google/bloaty>  

---

## Topic Overview

Once you know your binary is large, the next question is: which symbols and which compilation units are responsible? Bloaty can drill all the way down to individual symbols and compilation units, and that level of detail is essential for tracking down the number-one cause of large C++ binaries: template instantiation bloat.

The three most useful invocations for this kind of investigation are:

```bash
# Show top symbols:
bloaty -d symbols -n 20 ./myapp

# Show top compilation units:
bloaty -d compileunits -n 10 ./myapp

# Two-level: symbols within compilation units:
bloaty -d compileunits,symbols -n 10 ./myapp
```

The two-level form is particularly revealing - it tells you not just what is large, but which `.cpp` file is responsible for generating it.

---

## Self-Assessment

### Q1: Identify which symbols consume the most space

Run Bloaty with the `-d symbols` flag and ask for the top 15 results:

```bash
bloaty -d symbols -n 15 ./myapp
```

You will typically see output like this:

```text
    FILE SIZE        VM SIZE
 --------------  --------------
  15.3%   180Ki  15.3%   180Ki   std::__cxx11::basic_string<char, ...>
   9.8%   115Ki   9.8%   115Ki   std::vector<Widget, ...>::_M_realloc_insert
   7.2%    85Ki   7.2%    85Ki   std::sort<__gnu_cxx::__normal_iterator<...>>
   5.1%    60Ki   5.1%    60Ki   nlohmann::json::parse(...)
   4.3%    51Ki   4.3%    51Ki   MyApp::processData()
```

Your own code (`MyApp::processData`) is usually near the bottom of this list. The real culprits are standard library and third-party template instantiations. There are three patterns to watch for:

- **STL symbols** (string, vector, map): each concrete type you use generates a full copy of the template body.
- **Third-party libraries**: JSON parsers, logging frameworks, and similar header-heavy libraries can pull in surprising amounts of code.
- **Exception handling**: `__cxa_throw`, `.eh_frame`, and `.gcc_except_table` can together add tens of kilobytes even when you rarely throw.

### Q2: Find and eliminate template instantiation bloat

The reason template bloat is so common is subtle: every time you use a template function with a new type, the compiler generates a complete, independent copy of the entire function body. If that body is 50 lines long and you use it with `int`, `double`, `string`, and `Widget`, you get four full copies. The fix is to factor out the type-independent logic into a non-template implementation and keep only the tiny type-conversion wrapper as a template.

Here is what that looks like:

```cpp
// BEFORE: Each type creates a full copy of the entire function
// -> log<int>, log<double>, log<string>, log<Widget> = 4 copies!

#include <iostream>
#include <string>
#include <typeinfo>

// BAD: full template - entire body duplicated per instantiation
template<typename T>
void log_value_bad(const std::string& label, const T& value) {
    std::cout << "[" << __FILE__ << ":" << __LINE__ << "] "
              << label << " = ";
    // 50 lines of formatting, buffering, flushing...
    std::cout << value << '\n';
}

// GOOD: type-erased core + thin template wrapper
void log_impl(const std::string& label, const std::string& str_value) {
    std::cout << label << " = " << str_value << '\n';
    // All 50 lines of formatting only exist ONCE
}

template<typename T>
void log_value_good(const std::string& label, const T& value) {
    // Thin wrapper: only the conversion is templated
    if constexpr (std::is_same_v<T, std::string>) {
        log_impl(label, value);
    } else {
        log_impl(label, std::to_string(value));
    }
}

int main() {
    log_value_good("count", 42);
    log_value_good("ratio", 3.14);
    log_value_good("name", std::string("Alice"));
}
// Expected output:
// count = 42
// ratio = 3.140000
// name = Alice
```

You can confirm the improvement with Bloaty's diff mode. Before the refactor, you see three equally-sized template instantiations; after, you see one shared implementation and three tiny wrappers:

```bash
# Before (full template):
bloaty -d symbols ./before | grep log_value
#   log_value<int>    2.1Ki
#   log_value<double> 2.1Ki
#   log_value<string> 2.3Ki

# After (thin wrapper + shared impl):
bloaty -d symbols ./after | grep log
#   log_impl          2.1Ki  (shared)
#   log_value<int>    0.1Ki  (tiny wrapper)
#   log_value<double> 0.1Ki
```

That is a significant reduction - and it scales up quickly when the template body is large or when there are many distinct instantiations.

### Q3: `std::function` vs template parameter binary size comparison

Another common source of size growth is using a template parameter for every callable. Each distinct lambda type produces a separate instantiation of the entire function. `std::function` solves this by using type erasure - all callables go through one shared code path. The cost is a virtual-dispatch-like indirection at runtime.

Here is a side-by-side comparison:

```cpp
#include <iostream>
#include <functional>
#include <vector>

// Version A: std::function (type-erased, ONE instantiation)
void process_func(const std::vector<int>& data,
                  std::function<int(int)> transform) {
    for (auto x : data)
        std::cout << transform(x) << ' ';
    std::cout << '\n';
}

// Version B: template (separate instantiation per callable type)
template<typename F>
void process_tmpl(const std::vector<int>& data, F transform) {
    for (auto x : data)
        std::cout << transform(x) << ' ';
    std::cout << '\n';
}

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // std::function: one instantiation, but ~400 bytes runtime overhead
    process_func(v, [](int x) { return x * 2; });
    process_func(v, [](int x) { return x + 10; });

    // template: two instantiations, but inline-able (faster)
    process_tmpl(v, [](int x) { return x * 2; });
    process_tmpl(v, [](int x) { return x + 10; });
}
// Expected output:
// 2 4 6 8 10
// 11 12 13 14 15
// 2 4 6 8 10
// 11 12 13 14 15
```

The trade-off is a classic size-vs-speed choice:

| Approach | Code Size | Runtime Speed | Use When |
| --- | --- | --- | --- |
| `std::function` | Smaller (1 instantiation) | Slower (indirect call) | Binary size matters, many callers |
| Template | Larger (N instantiations) | Faster (inlined) | Hot path, few distinct callers |

The right answer depends on the context. For a function called from a hundred different places across a large codebase, `std::function` can be a significant win. For a hot inner loop with a single caller, a template is better.

---

## Notes

- Bloaty's diff mode (`bloaty new -- old`) is essential for tracking size regressions.
- Common bloat sources: `<iostream>` (~100KB), `<regex>` (~300KB), exception tables.
- Use `extern template` to prevent unwanted instantiations in headers.
- `-ffunction-sections -fdata-sections -Wl,--gc-sections` removes unused functions.
- Consider thin template wrappers around type-erased implementations.
