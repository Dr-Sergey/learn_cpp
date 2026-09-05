# Apply AI tools to migrate C++ codebases between standards (C++14 to C++20)

**Category:** AI Assisted Development  
**Standard:** C++14  

---

## Topic Overview

Migrating a C++ codebase from C++14/17 to C++20/23 involves considerably more than just enabling a new compiler flag. The standard adds concepts, ranges, coroutines, modules, `std::format`, designated initializers, `operator<=>`, and more - and many of these are meant to replace idioms you have been using for years. Doing that migration safely requires knowing what to change, in what order, and how risky each change is.

AI tools help with all three concerns. They can identify **deprecated features** in your existing code, suggest **modern replacements**, generate **migration checklists** prioritized by effort and risk, and rewrite individual functions to use new language features. The key to a successful migration is systematic and incremental: change one category of thing at a time, test after each step, and let AI handle the mechanical pattern-matching work so you can focus on the cases that require judgment.

### C++14 -> C++20 Key Migration Points

| C++14/17 Pattern | C++20 Replacement | Migration Complexity |
| --- | --- | --- |
| SFINAE / `enable_if` | Concepts + `requires` | Medium |
| Manual type traits | Concepts | Low-Medium |
| `<algorithm>` with iterators | Ranges views + actions | Medium |
| `std::bind` | Lambdas (already available) | Low |
| Header-based includes | Modules (optional) | High |
| Callback chains | Coroutines | High |
| `volatile` for threading | `std::atomic` (should already be) | Critical |
| Uniform initialization gotchas | Designated initializers | Low |
| `std::result_of` | `std::invoke_result` | Low |
| Custom three-way compare | `operator<=>` | Medium |

---

## Self-Assessment

### Q1: Migrate SFINAE to concepts

**Answer:**

SFINAE (Substitution Failure Is Not An Error) was the C++14 way to constrain templates. It works, but the syntax is a puzzle that even experienced C++ programmers have to look up every time. Concepts in C++20 express the same constraints in a form that reads like what it means. Beyond readability, there is a practical benefit: when a constraint is violated, concepts produce clear error messages, while SFINAE failures produce walls of template instantiation noise.

The prompt should ask the AI to preserve the same constraints and overload behavior, and to call out any subtle semantic differences:

```cpp
=== PROMPT ===

"Migrate this SFINAE-based template code to C++20 concepts.
Preserve the same constraints and overload behavior.
Explain any subtle differences."
```

Here is what the migration looks like for a pair of overloaded functions and a constrained container function:

```cpp
// === C++14 SFINAE code ===
template<typename T,
    typename = std::enable_if_t<std::is_integral_v<T>>>
auto process_number(T value) -> decltype(value * 2) {
    return value * 2;
}

template<typename T,
    typename = std::enable_if_t<std::is_floating_point_v<T>>,
    typename = void>  // Extra param to avoid redefinition
auto process_number(T value) -> decltype(value * 2.0) {
    return std::round(value * 2.0);
}

// Constrained container function
template<typename Container,
    typename = std::enable_if_t<
        std::is_same_v<
            typename Container::value_type,
            std::decay_t<typename Container::value_type>
        > && !std::is_same_v<Container, std::string>
    >>
void sort_and_unique(Container& c) {
    std::sort(c.begin(), c.end());
    c.erase(std::unique(c.begin(), c.end()), c.end());
}


// === AI-MIGRATED C++20 with concepts ===

// Clean, readable concept definitions
template<typename T>
concept Integral = std::integral<T>;

template<typename T>
concept FloatingPoint = std::floating_point<T>;

template<typename C>
concept SortableContainer =
    std::ranges::random_access_range<C> &&
    std::sortable<std::ranges::iterator_t<C>> &&
    !std::same_as<C, std::string>;

// Clear overloads - no SFINAE hacks
auto process_number(Integral auto value) {
    return value * 2;
}

auto process_number(FloatingPoint auto value) {
    return std::round(value * 2.0);
}

// Constrained with concept
void sort_and_unique(SortableContainer auto& c) {
    std::ranges::sort(c);
    auto [first, last] = std::ranges::unique(c);
    c.erase(first, last);
}

// AI note: "Key difference - SFINAE silently removes
// overloads, concepts produce clear error messages.
// Both are equivalent for valid inputs, but concepts
// give better diagnostics for invalid inputs."
```

The reason the SFINAE version needed the extra `typename = void` parameter was to avoid two overloads with identical signatures after substitution. Concepts do not have that problem - the constraint is part of the function template, so the two overloads are naturally distinct. The concept version also reads like documentation: `SortableContainer` tells you exactly what the function requires.

### Q2: Migrate algorithm code to ranges

**Answer:**

The iterator-based `<algorithm>` style forces you to pass `begin()` and `end()` pairs everywhere, and multi-step pipelines (filter, then sort, then take N) require intermediate containers and multiple passes. C++20 ranges let you express the same pipeline as a chain of view operations - and the filter/transform steps are lazy, meaning they do not do work until you actually iterate the result.

The important caveat the AI will note is that lazy views cannot be sorted in place, so you still need to materialize to a container before sorting:

```cpp
=== PROMPT ===

"Convert this iterator-based algorithm code to C++20 ranges.
Use views for lazy evaluation where beneficial.
Keep the same logic and output."
```

Here is the before and after for a realistic pipeline that filters a list of orders, sorts by priority, and takes the top ten:

```cpp
// === C++14 iterator-based ===
std::vector<Order> get_high_priority_orders(
    const std::vector<Order>& orders,
    double min_value) {

    std::vector<Order> result;

    // Filter by status
    std::copy_if(orders.begin(), orders.end(),
        std::back_inserter(result),
        [](const Order& o) { return o.status == Status::Active; });

    // Remove low-value orders
    result.erase(
        std::remove_if(result.begin(), result.end(),
            [min_value](const Order& o) {
                return o.total_value < min_value;
            }),
        result.end());

    // Sort by priority (highest first)
    std::sort(result.begin(), result.end(),
        [](const Order& a, const Order& b) {
            return a.priority > b.priority;
        });

    // Take top 10
    if (result.size() > 10)
        result.resize(10);

    return result;
}


// === AI-MIGRATED C++20 ranges ===
std::vector<Order> get_high_priority_orders(
    std::span<const Order> orders,
    double min_value) {

    // Single pipeline: filter, sort, take
    auto filtered = orders
        | std::views::filter([](const Order& o) {
              return o.status == Status::Active;
          })
        | std::views::filter([min_value](const Order& o) {
              return o.total_value >= min_value;
          });

    // Materialize to sort (views can't sort in-place)
    std::vector<Order> result(filtered.begin(), filtered.end());

    std::ranges::sort(result, std::greater{},
                      &Order::priority);

    // Take top 10
    result.resize(std::min(result.size(), size_t{10}));
    return result;
}

// AI note: "std::views::filter is lazy - elements are
// evaluated on demand. But we need to materialize for
// sorting. In C++23, you can use std::ranges::to<vector>
// for cleaner materialization."
```

Notice that the ranges version accepts `std::span<const Order>` instead of `const std::vector<Order>&` - this is the idiomatic change that goes along with using ranges, since `span` works with any contiguous sequence, not just vectors.

### Q3: Generate a migration checklist with AI

**Answer:**

For a large codebase you want to tackle migration in an order that maximizes benefit and minimizes risk - not alphabetically, not by file, but by the cost/benefit ratio of each type of change. Ask the AI to analyze your build configuration and generate a prioritized checklist:

```cpp
=== PROMPT ===

"Analyze this CMakeLists.txt and source list.
Generate a C++14 -> C++20 migration checklist.
For each item, rate: effort (1-5), risk (1-5),
and benefit (1-5). Sort by benefit/effort ratio."


=== AI-GENERATED MIGRATION CHECKLIST ===

 Priority  | Item                           | Effort | Risk | Benefit
 -----------|--------------------------------|--------|------|--------
 1 (FIRST) | Add [[nodiscard]] to getters   |   1    |  1   |   4
 2         | Replace enable_if -> concepts  |   2    |  2   |   5
 3         | Replace typedef -> using        |   1    |  1   |   2
 4         | Add operator<=> to value types  |   2    |  2   |   4
 5         | Use designated initializers     |   1    |  1   |   3
 6         | Replace std::bind -> lambdas   |   2    |  1   |   3
 7         | Migrate algorithms -> ranges   |   3    |  2   |   4
 8         | Add constexpr to pure functions |   2    |  2   |   3
 9         | Replace output params -> return |   3    |  3   |   4
 10        | Use std::format (if available)  |   2    |  2   |   3
 11        | Add concepts to template APIs   |   3    |  2   |   5
 12 (LAST) | Migrate to modules              |   5    |  5   |   3


Compiler compatibility notes:
- GCC 10+: full C++20 support (except modules)
- Clang 16+: most C++20 features
- MSVC 19.29+: good C++20, modules partial
- Modules: NOT recommended until build system
  support matures (CMake 3.28+)
```

The CMakeLists.txt changes to enable C++20 are straightforward; the incremental build flags handle compiler-specific features that were not yet stabilized in older versions:

```cpp
// CMakeLists.txt migration:
// BEFORE:
set(CMAKE_CXX_STANDARD 14)

// AFTER (incremental):
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)  // Disable GNU extensions

// Add compiler-specific flags for full C++20:
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    add_compile_options(-fconcepts -fcoroutines)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    add_compile_options(-fcoroutines-ts)  # Clang < 16
endif()
```

The checklist ordering is worth internalizing: low-effort, low-risk changes like `[[nodiscard]]` and `typedef -> using` come first because they give you quick wins without destabilizing anything. Modules come last because the tooling is still maturing - the risk column reflects that, not a problem with the feature itself.

---

## Notes

- **Start with `-std=c++20`** and fix compilation errors - the compiler itself flags deprecated features and removed names.
- AI can **batch-process files** with prompts like "find all SFINAE patterns in this file and convert to concepts."
- **Test after each migration step** - ranges and concepts can subtly change overload resolution in ways that are hard to predict.
- `std::result_of` was removed in C++20; replace it with `std::invoke_result`.
- Watch for **aggregate initialization changes** in C++20: user-declared constructors prevent aggregation, which can silently break existing initialization syntax.
- AI can generate **compatibility macros** for gradual migration (e.g., `#if __cpp_concepts >= 202002L`).
- Modules migration is the **riskiest** - do it last, if at all, and only when your build system (CMake 3.28+) and compiler have solid support.
