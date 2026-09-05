# Use reflection for automatic debug printing and logging

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2996>  

---

## Topic Overview

### The Problem

One of the most tedious parts of C++ is writing `operator<<` overloads for structs just so you can print them during debugging. Every time you add a field you have to update the printer. Every time you rename a field you have to find and fix all the string literals. With reflection, all of that boilerplate disappears.

Here is what the old way looks like:

```cpp
struct Config {
    std::string host;
    int port;
    bool tls;
};

// You have to write this by hand for every struct:
std::ostream& operator<<(std::ostream& os, const Config& c) {
    return os << "Config{host=" << c.host
              << ", port=" << c.port
              << ", tls=" << c.tls << "}";
}
```

With reflection, this becomes automatic.

### Automatic `operator<<` with Reflection

The `auto_print` function below works for any aggregate type. It uses `template for` to expand over the members at compile time, printing each one's name and value. You never write a single field name yourself.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>

template<typename T>
    requires std::is_aggregate_v<T>
std::ostream& auto_print(std::ostream& os, const T& obj) {
    os << std::meta::display_string_of(^^T) << "{";
    bool first = true;

    template for (constexpr auto member :
                  std::meta::nonstatic_data_members_of(^^T)) {
        if (!first) os << ", ";
        os << std::meta::identifier_of(member) << "="
           << obj.[:member:];
        first = false;
    }

    return os << "}";
}

struct Config {
    std::string host = "localhost";
    int port = 8080;
    bool tls = true;
};

int main() {
    Config c;
    auto_print(std::cout, c);
    // Output: Config{host=localhost, port=8080, tls=1}
}
```

The `requires std::is_aggregate_v<T>` constraint is worth noting - it limits the function to aggregates (plain structs and arrays with no user-provided constructors), which are the types where `nonstatic_data_members_of` gives you a clean, predictable list of fields.

### Generic Debug Logger

This version adds `std::format`-based value formatting and a `source_location`-aware macro so you can log any aggregate with its file and line number attached.

```cpp
#include <meta>
#include <format>
#include <source_location>
#include <iostream>

template<typename T>
std::string to_debug_string(const T& obj) {
    if constexpr (std::is_aggregate_v<T>) {
        std::string result = std::string(std::meta::display_string_of(^^T)) + "{";
        bool first = true;

        template for (constexpr auto m :
                      std::meta::nonstatic_data_members_of(^^T)) {
            if (!first) result += ", ";
            result += std::meta::identifier_of(m);
            result += "=";
            result += std::format("{}", obj.[:m:]);
            first = false;
        }

        result += "}";
        return result;
    } else {
        return std::format("{}", obj);
    }
}

#define LOG_DEBUG(obj) \
    std::cerr << "[DEBUG " << std::source_location::current().file_name() \
              << ":" << std::source_location::current().line() << "] " \
              << to_debug_string(obj) << "\n"
```

The `if constexpr (std::is_aggregate_v<T>)` branch handles structs; the `else` branch handles primitive types and anything that has a `std::format` specialization. This makes `to_debug_string` usable from other templates even when the type is not a struct.

### Recursive Printing for Nested Structs

When your structs contain other structs, a flat printer is not enough. This version recurses into nested aggregates and produces an indented, tree-shaped output.

```cpp
template<typename T>
std::string deep_debug(const T& obj, int indent = 0) {
    std::string pad(indent * 2, ' ');

    if constexpr (std::is_aggregate_v<T>) {
        std::string result = std::string(std::meta::display_string_of(^^T)) + " {\n";

        template for (constexpr auto m :
                      std::meta::nonstatic_data_members_of(^^T)) {
            result += pad + "  " + std::string(std::meta::identifier_of(m)) + " = ";
            result += deep_debug(obj.[:m:], indent + 1);
            result += "\n";
        }

        result += pad + "}";
        return result;
    } else {
        return std::format("{}", obj);
    }
}

struct Address {
    std::string city;
    int zip;
};

struct Person {
    std::string name;
    int age;
    Address address;
};

int main() {
    Person p{"Alice", 30, {"NYC", 10001}};
    std::cout << deep_debug(p) << "\n";
    // Output:
    // Person {
    //   name = Alice
    //   age = 30
    //   address = Address {
    //     city = NYC
    //     zip = 10001
    //   }
    // }
}
```

The recursion here is natural: when `deep_debug` hits the `address` member, `obj.[:m:]` gives it an `Address`, and it calls `deep_debug` again on that - which follows the aggregate branch and prints `city` and `zip` with one more level of indentation.

### Structured Logging with Key-Value Pairs

For production logging systems that want structured data rather than a formatted string, you can extract a list of name/value pairs instead:

```cpp
template<typename T>
auto to_log_map(const T& obj) {
    std::vector<std::pair<std::string, std::string>> entries;

    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^^T)) {
        entries.emplace_back(
            std::string(std::meta::identifier_of(m)),
            std::format("{}", obj.[:m:])
        );
    }

    return entries;
}

// Integration with structured logging systems:
// auto kv = to_log_map(config);
// logger.info("Server starting", kv);
```

This pattern integrates naturally with logging frameworks that accept key-value data (think OpenTelemetry structured logs, or any JSON-oriented logger). You get structured field names from reflection, which means log queries and alerting rules that filter on field names will keep working correctly when you rename a struct field - because the logged key name changes automatically too.

---

## Self-Assessment

### Q1: Write a generic `to_string` function using reflection

```cpp
template<typename T>
std::string auto_to_string(const T& obj) {
    if constexpr (std::is_aggregate_v<T>) {
        std::string result = "{";
        bool first = true;
        template for (constexpr auto m :
                      std::meta::nonstatic_data_members_of(^^T)) {
            if (!first) result += ", ";
            result += std::format("{}={}", std::meta::identifier_of(m), obj.[:m:]);
            first = false;
        }
        return result + "}";
    } else {
        return std::format("{}", obj);
    }
}
```

This is the compact form of the pattern: for aggregates, iterate over members and format each as `name=value`; for everything else, fall back to `std::format`. The whole thing is generic and works on any type you hand it.

### Q2: How does `template for` differ from a regular `for` loop

`template for` is a **compile-time expansion** loop (sometimes called "expansion statements"). Each iteration is instantiated separately by the compiler, which gives you two things a normal loop cannot provide:

- Different types per iteration - each member may have a completely different type, and `template for` handles that naturally because each iteration is its own template instantiation.
- Compile-time access to reflection values - the `constexpr auto m` loop variable is a compile-time constant in each iteration, which means you can splice it, use it in `if constexpr`, and pass it to `consteval` functions.

A regular `for` loop iterates at runtime over a homogeneous container where every element has the same type. It cannot be used here because the member types differ from field to field.

### Q3: Generate an `operator==` automatically using reflection

```cpp
template<typename T>
    requires std::is_aggregate_v<T>
bool auto_equals(const T& a, const T& b) {
    bool equal = true;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^^T)) {
        equal = equal && (a.[:m:] == b.[:m:]);
    }
    return equal;
}
```

This walks every field and short-circuits by accumulating into `equal`. The result is a correct, field-by-field equality comparison that stays automatically in sync with the struct - no manual `==` overload needed.

---

## Notes

- Reflection-based debug printing eliminates hundreds of lines of boilerplate in real codebases.
- The `template for` / expansion statement syntax may evolve - check the latest P2996 revision.
- For production logging, consider generating `std::format` specializations instead of `operator<<`.
- Reflection-based printing integrates naturally with structured logging frameworks.
