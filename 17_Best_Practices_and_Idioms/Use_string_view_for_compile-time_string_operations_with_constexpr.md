# Use string_view for compile-time string operations with constexpr

**Category:** Best Practices and Idioms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string_view>  

---

## Topic Overview

`std::string_view` is `constexpr`-friendly since C++17, allowing string parsing, splitting, and comparisons at compile time. Unlike `std::string`, it never allocates - it is just a pointer and a length into existing storage. That zero-allocation property is what makes it usable in `constexpr` contexts where heap allocation is forbidden.

```cpp
constexpr std::string_view greeting = "Hello, World!";
constexpr auto comma_pos = greeting.find(',');        // 5 (at compile time!)
constexpr auto hello = greeting.substr(0, comma_pos); // "Hello"
static_assert(hello == "Hello");
```

---

## Self-Assessment

### Q1: Write a constexpr function that splits a `string_view` at a delimiter

The template parameter `N` tells the function how many pieces to split into, which lets the result fit in a `std::array` - also a compile-time-friendly container. The whole function executes at compile time when called with a string literal, which means errors in your format strings can become compile errors.

```cpp
#include <array>
#include <iostream>
#include <string_view>

template<size_t N>
constexpr std::array<std::string_view, N>
split(std::string_view sv, char delim) {
    std::array<std::string_view, N> result{};
    size_t idx = 0;
    while (!sv.empty() && idx < N) {
        auto pos = sv.find(delim);
        if (pos == std::string_view::npos) {
            result[idx++] = sv;
            break;
        }
        result[idx++] = sv.substr(0, pos);
        sv.remove_prefix(pos + 1);
    }
    return result;
}

int main() {
    // Compile-time split!
    constexpr auto parts = split<4>("one,two,three,four", ',');
    static_assert(parts[0] == "one");
    static_assert(parts[1] == "two");
    static_assert(parts[2] == "three");
    static_assert(parts[3] == "four");

    for (auto p : parts)
        std::cout << '[' << p << "] ";
    std::cout << '\n';

    // Splitting a path
    constexpr auto path_parts = split<3>("usr/local/bin", '/');
    static_assert(path_parts[0] == "usr");
    static_assert(path_parts[1] == "local");
    static_assert(path_parts[2] == "bin");

    for (auto p : path_parts)
        std::cout << '/' << p;
    std::cout << '\n';
}
// Expected output:
// [one] [two] [three] [four]
// /usr/local/bin
```

The `static_assert` calls confirm the splits are correct at compile time - the program would not even compile if any assertion failed. There is zero runtime cost for the parsing itself.

### Q2: Use `string_view` comparisons in `static_assert`

`string_view` comparisons are `constexpr`, which means you can validate configuration values, version strings, and build settings at compile time rather than discovering bad values at startup.

```cpp
#include <iostream>
#include <string_view>

// Compile-time configuration validation
constexpr std::string_view APP_NAME = "MyApp";
constexpr std::string_view VERSION = "2.5.1";
constexpr std::string_view BUILD_TYPE = "Release";

// Validate at compile time!
static_assert(!APP_NAME.empty(), "APP_NAME must not be empty");
static_assert(VERSION.size() == 5, "VERSION must be X.Y.Z format");
static_assert(BUILD_TYPE == "Release" || BUILD_TYPE == "Debug",
              "BUILD_TYPE must be Release or Debug");

constexpr bool starts_with(std::string_view sv, std::string_view prefix) {
    return sv.substr(0, prefix.size()) == prefix;
}

constexpr bool ends_with(std::string_view sv, std::string_view suffix) {
    return sv.size() >= suffix.size() &&
           sv.substr(sv.size() - suffix.size()) == suffix;
}

static_assert(starts_with(VERSION, "2."));
static_assert(ends_with("config.json", ".json"));
static_assert(!ends_with("config.json", ".xml"));

constexpr bool contains(std::string_view sv, std::string_view needle) {
    return sv.find(needle) != std::string_view::npos;
}

static_assert(contains("Hello, World!", "World"));
static_assert(!contains("Hello, World!", "Planet"));

int main() {
    std::cout << APP_NAME << " v" << VERSION << " (" << BUILD_TYPE << ")\n";
}
// Expected output:
// MyApp v2.5.1 (Release)
```

Note that C++20 added `starts_with` and `ends_with` as member functions directly on `string_view`, so the helper functions above would be unnecessary in C++20. The pattern of using `static_assert` with `string_view` comparisons is still valuable in both standards.

### Q3: Compile-time state machine with `constexpr string_view` parsing

This example pushes `constexpr` string processing to its logical limit: a full state machine that validates whether a string looks like a float literal. Every `static_assert` in `main` runs at compile time. If you wrote `is_valid_float("3.")` expecting true, the build would fail with a clear message before you ever run the program.

```cpp
#include <iostream>
#include <string_view>

enum class State { Start, Number, Dot, Fraction, End, Error };

constexpr State advance(State s, char c) {
    switch (s) {
        case State::Start:
            if (c >= '0' && c <= '9') return State::Number;
            return State::Error;
        case State::Number:
            if (c >= '0' && c <= '9') return State::Number;
            if (c == '.') return State::Dot;
            return State::Error;
        case State::Dot:
            if (c >= '0' && c <= '9') return State::Fraction;
            return State::Error;
        case State::Fraction:
            if (c >= '0' && c <= '9') return State::Fraction;
            return State::Error;
        default:
            return State::Error;
    }
}

constexpr bool is_valid_float(std::string_view sv) {
    if (sv.empty()) return false;
    State state = State::Start;
    for (char c : sv) {
        state = advance(state, c);
        if (state == State::Error) return false;
    }
    return state == State::Fraction;  // must end in fraction part
}

// All validated at COMPILE TIME!
static_assert(is_valid_float("3.14"));
static_assert(is_valid_float("0.5"));
static_assert(is_valid_float("123.456"));
static_assert(!is_valid_float(".5"));      // no leading digit
static_assert(!is_valid_float("3."));      // no fraction digit
static_assert(!is_valid_float("abc"));
static_assert(!is_valid_float(""));

int main() {
    constexpr auto test = "42.0";
    if constexpr (is_valid_float(test))
        std::cout << test << " is a valid float literal\n";
    else
        std::cout << test << " is NOT valid\n";
}
// Expected output:
// 42.0 is a valid float literal
```

The state machine runs entirely at compile time - no runtime cost, no runtime failure, and invalid inputs become build errors rather than runtime surprises.

---

## Notes

- `std::string_view` is `constexpr` since C++17; more operations added in C++20 (`starts_with`, `ends_with`).
- `constexpr std::string` is available since C++20, but `string_view` is preferred when no allocation is needed.
- `string_view` doesn't own data - ensure the underlying string outlives the view.
- Use compile-time parsing for config validation, protocol checking, DSL processing.
