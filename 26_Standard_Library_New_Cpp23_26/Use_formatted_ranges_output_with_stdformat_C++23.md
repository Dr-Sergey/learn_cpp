# Use Formatted Ranges Output with `std::format` (C++23)

**Category:** Standard Library New Cpp23 26  
**Standard:** C++23  
**Reference:** [cppreference — range formatting](https://en.cppreference.com/w/cpp/utility/format/ranges)  

---

## Topic Overview

C++23 extends `std::format` and `std::print` to natively support formatting of ranges and containers. Before C++23, printing a `std::vector`, `std::map`, or `std::set` required manual loops or custom `operator<<` overloads. Now, any range that models `std::ranges::input_range` can be formatted directly - and the output is immediately readable with sensible defaults.

The default range formatting uses square brackets with comma separation for sequences and curly braces for sets/maps. Strings and character ranges use special representations. The `?s` format spec enables debug-escaped string output.

| Container | Default format output | With format spec |
| --- | --- | --- |
| `vector<int>{1,2,3}` | `[1, 2, 3]` | `::#x` -> `[0x1, 0x2, 0x3]` |
| `set<int>{1,2,3}` | `{1, 2, 3}` | N/A |
| `map<string,int>` | `{"a": 1, "b": 2}` | N/A |
| `pair<int,int>` | `(1, 2)` | N/A |
| `tuple<int,string,double>` | `(1, "hello", 3.14)` | N/A |
| `string("hello")` | `hello` | `?s` -> `"hello"` |
| `vector<string>` | `["a", "b"]` | Debug strings by default |

The format specifiers for ranges follow this pattern:

```cpp
{:[fill][align][width][n][range-type][:element-format-spec]}
```

Where `n` suppresses the brackets, and the inner spec after `:` applies to each element:

```cpp
std::format("{::>5}", vec)    //  each element right-aligned in width 5
std::format("{:n}", vec)      //  no surrounding brackets: "1, 2, 3"
```

---

## Self-Assessment

### Q1: How do you format vectors, maps, and sets with `std::format` and `std::print`

The most common use case is just printing a container for debugging or logging. Here is the full range of built-in support, from simple vectors to nested containers:

```cpp
#include <format>
#include <print>
#include <vector>
#include <map>
#include <set>
#include <string>
#include <tuple>
#include <utility>

int main() {
    // Vector formatting
    std::vector<int> nums = {10, 20, 30, 40};
    std::println("{}", nums);             // [10, 20, 30, 40]
    std::println("{::06d}", nums);        // [000010, 000020, 000030, 000040]
    std::println("{::#x}", nums);         // [0xa, 0x14, 0x1e, 0x28]
    std::println("{:n}", nums);           // 10, 20, 30, 40  (no brackets)

    // Map formatting
    std::map<std::string, int> scores = {
        {"Alice", 95}, {"Bob", 87}, {"Charlie", 92}
    };
    std::println("{}", scores);
    // {"Alice": 95, "Bob": 87, "Charlie": 92}

    // Set formatting
    std::set<int> unique = {3, 1, 4, 1, 5, 9};
    std::println("{}", unique);           // {1, 3, 4, 5, 9}

    // Pair and tuple
    auto p = std::pair{42, "hello"};
    std::println("{}", p);                // (42, "hello")

    auto t = std::tuple{1, 3.14, "world"};
    std::println("{}", t);                // (1, 3.14, "world")

    // Nested containers
    std::vector<std::vector<int>> matrix = {{1, 2}, {3, 4}, {5, 6}};
    std::println("{}", matrix);           // [[1, 2], [3, 4], [5, 6]]

    // String debug formatting
    std::vector<std::string> words = {"hello", "world"};
    std::println("{}", words);            // ["hello", "world"]
    std::println("{:?s}", "tab\there");   // "tab\there" (escaped)
}
```

Notice that different container types get different bracket styles automatically. Sequences get `[]`, sets get `{}`, pairs and tuples get `()`. These match the conventional notation for each type.

---

### Q2: How do you apply per-element format specifications and align range output

The double-colon syntax (`::spec`) is the key to per-element formatting. The first colon opens the range format spec, and the second colon introduces the element spec. It looks a little odd at first but becomes natural quickly:

```cpp
#include <format>
#include <print>
#include <vector>
#include <string>

int main() {
    std::vector<double> values = {1.5, 22.333, 0.007, 100.0};

    // Per-element formatting
    // Fixed precision for each element
    std::println("{::.2f}", values);      // [1.50, 22.33, 0.01, 100.00]

    // Scientific notation
    std::println("{::.3e}", values);      // [1.500e+00, 2.233e+01, 7.000e-03, 1.000e+02]

    // Width and alignment of the whole range
    std::vector<int> small = {1, 2, 3};
    std::println("[{:>30}]", std::format("{}", small));
    // Right-align the entire formatted range in a 30-char field

    // No-brackets ('n' flag) for CSV-like output
    std::println("{:n:>5}", small);       // "    1,     2,     3"
    // Each element right-aligned in width 5, no surrounding brackets

    // Hex dump of bytes
    std::vector<uint8_t> bytes = {0xDE, 0xAD, 0xBE, 0xEF};
    std::println("{:n::02X}", bytes);     // DE, AD, BE, EF

    // Formatting a range of strings with padding
    std::vector<std::string> names = {"Al", "Bob", "Charlie"};
    std::println("{::<10}", names);       // [Al........, Bob......., Charlie...]
    // Each name left-aligned, padded with dots

    // Joining with custom separator via ranges
    // Note: std::format does not support custom separators directly
    // but you can use the 'n' flag and wrap:
    auto csv = std::format("{:n}", small);   // "1, 2, 3"
    std::println("CSV: {}", csv);
}
```

---

### Q3: How do you write a custom `std::formatter` for your own range-like type

When you have a custom type, you can specialize `std::formatter` for it. Once you do, range formatting picks up your formatter automatically - so a `vector<YourType>` will use your custom formatting for each element without any extra work:

```cpp
#include <format>
#include <print>
#include <vector>
#include <string>
#include <ranges>
#include <cstddef>
#include <string_view>

// Custom type that is range-like
template <typename T>
struct Matrix {
    std::vector<std::vector<T>> data;
    std::size_t rows() const { return data.size(); }
    std::size_t cols() const { return data.empty() ? 0 : data[0].size(); }
};

// Custom formatter for Matrix
template <typename T>
struct std::formatter<Matrix<T>> {
    // Stored format spec for elements
    std::formatter<T> element_fmt;
    bool pretty = false;

    constexpr auto parse(std::format_parse_context& ctx) {
        auto it = ctx.begin();
        if (it != ctx.end() && *it == 'p') {
            pretty = true;
            ++it;
        }
        // Delegate remaining spec to element formatter
        ctx.advance_to(it);
        return element_fmt.parse(ctx);
    }

    auto format(const Matrix<T>& m, std::format_context& ctx) const {
        auto out = ctx.out();
        if (pretty) {
            out = std::format_to(out, "Matrix {}x{}:\n", m.rows(), m.cols());
            for (const auto& row : m.data) {
                out = std::format_to(out, "  |");
                for (const auto& val : row) {
                    out = std::format_to(out, " ");
                    out = element_fmt.format(val, ctx);
                }
                out = std::format_to(out, " |\n");
            }
        } else {
            out = std::format_to(out, "{}", m.data);
        }
        return out;
    }
};

// Custom formatter for an enum using range formatting
enum class Color { Red, Green, Blue };

template <>
struct std::formatter<Color> : std::formatter<std::string_view> {
    auto format(Color c, std::format_context& ctx) const {
        std::string_view name;
        switch (c) {
            case Color::Red:   name = "Red";   break;
            case Color::Green: name = "Green"; break;
            case Color::Blue:  name = "Blue";  break;
        }
        return std::formatter<std::string_view>::format(name, ctx);
    }
};

int main() {
    // Default range formatting picks up the custom Color formatter
    std::vector<Color> palette = {Color::Red, Color::Green, Color::Blue};
    std::println("{}", palette);
    // [Red, Green, Blue]

    // Custom Matrix formatter
    Matrix<int> m{{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}};
    std::println("{}", m);              // [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
    std::println("{:p}", m);
    // Matrix 3x3:
    //   | 1 2 3 |
    //   | 4 5 6 |
    //   | 7 8 9 |

    Matrix<double> md{{{1.0, 2.5}, {3.7, 4.2}}};
    std::println("{:p.1f}", md);
    // Matrix 2x2:
    //   | 1.0 2.5 |
    //   | 3.7 4.2 |
}
```

The `Color` formatter example is especially worth studying. Inheriting from `std::formatter<std::string_view>` is a common pattern that lets you reuse all the width/alignment/fill machinery of the string formatter while just overriding what the displayed value actually is.

---

## Notes

- Range formatting is in `<format>` (C++23). Works with `std::format`, `std::format_to`, `std::print`, and `std::println`.
- Default brackets: `[]` for sequences, `{}` for associative containers, `()` for pairs/tuples.
- Use `:n` to suppress brackets, `::spec` to apply a format spec to each element.
- Strings inside ranges are automatically quoted and escaped in the debug representation.
- The `?` type specifier enables debug output: `?s` for escaped strings, `?` for chars.
- Custom types can be made formattable by specializing `std::formatter` - range formatting then works automatically with containers of those types.
- Nested ranges format recursively: `vector<vector<int>>` prints as `[[1, 2], [3, 4]]`.
- Feature-test macro: `__cpp_lib_format_ranges >= 202207L`.
- Compilers: GCC 13+, Clang 17+ (partial), MSVC 19.34+ (VS 2022 17.4).
