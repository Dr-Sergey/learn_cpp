# Use views::elements to project a range of tuples to a specific index

**Category:** Ranges Cpp20  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/elements_view>  

---

## Topic Overview

`views::elements<N>` projects a range of **tuple-like** elements to their Nth component. It works on any type that supports `std::get<N>()`: `std::tuple`, `std::pair`, `std::array`, and structured bindings-compatible types.

Think of it as a way to "zoom in" on one column of a table. If you have a range of student records as `(id, name, gpa)` tuples, `views::elements<1>` hands you just the names, without you having to write a lambda or manually index each element.

### Syntax

Here is what the pipe syntax looks like for extracting specific fields:

```cpp
auto names = pairs | std::views::elements<1>;  // extract second element
auto ids    = pairs | std::views::elements<0>;  // extract first element
```

### Convenience Aliases

| View | Equivalent to | Works on |
| --- | --- | --- |
| `views::keys` | `views::elements<0>` | `pair`, `tuple`, `map` iterators |
| `views::values` | `views::elements<1>` | `pair`, `tuple`, `map` iterators |
| `views::elements<N>` | (general form) | Any tuple-like with >= N+1 elements |

### How It Works Internally

The view yields references into the original tuples, not copies. That means writing through the view changes the original data.

```cpp
Source: range of tuple<A, B, C>

views::elements<0>  ->  range of A&     (references into original tuples)
views::elements<1>  ->  range of B&
views::elements<2>  ->  range of C&
```

### Supported Tuple-Like Types

- `std::pair<K, V>`
- `std::tuple<T...>`
- `std::array<T, N>`
- Any type with `std::tuple_size` and `std::get` specializations
- Range elements from `std::map` / `std::unordered_map` (which are `pair<const Key, Value>`)

---

## Self-Assessment

### Q1: Project the second element of a `vector<pair<int,string>>` using `views::elements<1>`

Here is the most common scenario: you have key-value pairs and you want to work with only one side. Notice that the view gives you references - writing through `names` actually modifies the original pairs.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <utility>
#include <vector>

int main() {
    std::vector<std::pair<int, std::string>> students = {
        {101, "Alice"},
        {102, "Bob"},
        {103, "Charlie"},
        {104, "Diana"},
    };

    // Extract just the names (second element)
    auto names = students | std::views::elements<1>;

    std::cout << "Names: ";
    for (const auto& name : names)
        std::cout << name << ' ';
    std::cout << '\n';

    // Extract just the IDs (first element) - equivalent to views::keys
    auto ids = students | std::views::elements<0>;

    std::cout << "IDs: ";
    for (int id : ids)
        std::cout << id << ' ';
    std::cout << '\n';

    // Modify through the view
    for (auto& name : names)
        name = "[" + name + "]";

    std::cout << "Modified: ";
    for (const auto& [id, name] : students)
        std::cout << id << '=' << name << ' ';
    std::cout << '\n';
}
// Expected output:
// Names: Alice Bob Charlie Diana
// IDs: 101 102 103 104
// Modified: 101=[Alice] 102=[Bob] 103=[Charlie] 104=[Diana]
```

The modification loop proves the view is non-owning - those bracket-wrapped names are sitting in the original `students` vector, not in some separate copy.

- `views::elements<1>` extracts the second element (`string`) from each `pair<int, string>`.
- The result is a range of `string&` references - writing through the view modifies the original pairs.
- `views::elements<0>` extracts the first element - identical to `views::keys`.

### Q2: Show that `views::elements<0>` is equivalent to `views::keys`

`views::keys` and `views::values` are not magic aliases - they are literally defined in the standard as `views::elements<0>` and `views::elements<1>`. The `static_assert` below confirms they produce the exact same type.

```cpp
#include <iostream>
#include <map>
#include <ranges>
#include <string>

int main() {
    std::map<std::string, int> scores = {
        {"Alice", 95}, {"Bob", 87}, {"Charlie", 92}
    };

    // Using views::keys
    std::cout << "keys:       ";
    for (const auto& name : scores | std::views::keys)
        std::cout << name << ' ';
    std::cout << '\n';

    // Using views::elements<0> - identical result
    std::cout << "elements<0>: ";
    for (const auto& name : scores | std::views::elements<0>)
        std::cout << name << ' ';
    std::cout << '\n';

    // Similarly, views::values == views::elements<1>
    std::cout << "values:      ";
    for (int s : scores | std::views::values)
        std::cout << s << ' ';
    std::cout << '\n';

    std::cout << "elements<1>: ";
    for (int s : scores | std::views::elements<1>)
        std::cout << s << ' ';
    std::cout << '\n';

    // Type check: they produce the same view type
    auto k1 = scores | std::views::keys;
    auto k2 = scores | std::views::elements<0>;
    static_assert(std::same_as<decltype(k1), decltype(k2)>);
    std::cout << "Same type: yes (static_assert passed)\n";
}
// Expected output:
// keys:       Alice Bob Charlie
// elements<0>: Alice Bob Charlie
// values:      95 87 92
// elements<1>: 95 87 92
// Same type: yes (static_assert passed)
```

One subtlety worth knowing: on `std::map`, the iterator yields `pair<const Key, Value>`, so `elements<0>` extracts `const Key&` (you can't modify a map key through the view), while `elements<1>` extracts a mutable `Value&`.

- `views::keys` is literally defined as `views::elements<0>` in the standard.
- `views::values` is literally defined as `views::elements<1>`.
- The `static_assert` confirms they produce the **exact same type**.
- On `std::map`, the iterator yields `pair<const Key, Value>`, so `elements<0>` extracts `const Key&` and `elements<1>` extracts `Value&`.

### Q3: Use `views::elements` on a range of 3-tuples to extract the third field

With three-field tuples the general `views::elements<N>` form really earns its keep - `views::keys` and `views::values` only reach the first two fields. Here the filter+elements pipeline shows how you can compose projection with filtering cleanly.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <tuple>
#include <vector>

int main() {
    using Record = std::tuple<int, std::string, double>;  // id, name, gpa

    std::vector<Record> students = {
        {1, "Alice",   3.9},
        {2, "Bob",     3.5},
        {3, "Charlie", 3.7},
        {4, "Diana",   4.0},
    };

    // Extract GPAs (third field, index 2)
    auto gpas = students | std::views::elements<2>;
    std::cout << "GPAs: ";
    for (double gpa : gpas)
        std::cout << gpa << ' ';
    std::cout << '\n';

    // Combine with other views: filter by GPA > 3.6, then get names
    auto honors = students
        | std::views::filter([](const Record& r) {
              return std::get<2>(r) > 3.6;
          })
        | std::views::elements<1>;  // extract name from filtered tuples

    std::cout << "Honors students: ";
    for (const auto& name : honors)
        std::cout << name << ' ';
    std::cout << '\n';

    // Extract all three fields separately
    auto ids   = students | std::views::elements<0>;
    auto names = students | std::views::elements<1>;

    std::cout << "All fields:\n";
    auto id_it   = std::ranges::begin(ids);
    auto name_it = std::ranges::begin(names);
    auto gpa_it  = std::ranges::begin(gpas);
    for (; id_it != std::ranges::end(ids); ++id_it, ++name_it, ++gpa_it) {
        std::cout << "  #" << *id_it << " " << *name_it
                  << " (GPA: " << *gpa_it << ")\n";
    }
}
// Expected output:
// GPAs: 3.9 3.5 3.7 4
// Honors students: Alice Charlie Diana
// All fields:
//   #1 Alice (GPA: 3.9)
//   #2 Bob (GPA: 3.5)
//   #3 Charlie (GPA: 3.7)
//   #4 Diana (GPA: 4)
```

The `filter | elements` pipeline is particularly elegant: you describe what you want (records with GPA > 3.6, then their names) and the pipeline handles the details. Each `elements<N>` view yields references into the original tuples, so no copying happens at any stage.

- `views::elements<2>` extracts the third field (`double` GPA) from each `tuple<int, string, double>`.
- The filter+elements pipeline first filters tuples by GPA > 3.6, then extracts names from the matching tuples.
- Each `elements<N>` view yields references into the original tuples, not copies.
- This approach is cleaner than writing `std::get<2>(record)` in every range-for loop.

---

## Notes

- `views::elements<N>` works with `std::array<T, M>` where `N < M`.
- On a `std::map<K,V>`, `views::keys` gives `const K&` (because map pairs are `pair<const K, V>`).
- The view preserves the range category of the source: if the source is random-access, so is `elements<N>`.
- Combined with `views::zip` (C++23), you can create tuple ranges from separate arrays and then project back with `elements`.
