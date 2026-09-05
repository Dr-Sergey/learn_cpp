# Use std::views::keys and views::values to iterate map keys or values

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/keys_view>  

---

## Topic Overview

`std::views::keys` and `std::views::values` (C++20) are range adaptors that project only the first or second element of a range of pair-like elements. When applied to a `std::map`, they let you iterate over keys or values without writing explicit lambda transforms.

Before these adaptors existed, iterating just the keys of a map meant writing `for (const auto& [k, v] : m)` and ignoring `v`, or writing a `views::transform` lambda. Both work, but `views::keys` is shorter, self-documenting, and requires no lambda at all.

### What They Do

```cpp
std::map<string, int>:  {"alice": 95, "bob": 87, "charlie": 42}
                            |                              |
  views::keys --------->   "alice", "bob", "charlie"      |
  views::values ------------------------------------------> 95, 87, 42
```

### Under the Hood

It's worth understanding what these adaptors actually are, because it makes their behavior predictable.

```cpp
// views::keys is equivalent to:
views::elements<0>

// views::values is equivalent to:
views::elements<1>

// Both are syntactic sugar for:
views::transform([](auto& pair) -> auto& { return std::get<N>(pair); })
```

### Core Example

Here are keys, values, and a composed pipeline that filters before projecting.

```cpp
#include <map>
#include <ranges>
#include <iostream>
#include <string>

int main() {
    std::map<std::string, int> scores{
        {"Alice", 95}, {"Bob", 87}, {"Charlie", 42}
    };

    // Iterate only keys:
    std::cout << "Keys: ";
    for (const auto& key : scores | std::views::keys)
        std::cout << key << " ";
    std::cout << "\n";
    // Output: Keys: Alice Bob Charlie

    // Iterate only values:
    std::cout << "Values: ";
    for (int val : scores | std::views::values)
        std::cout << val << " ";
    std::cout << "\n";
    // Output: Values: 95 87 42

    // Compose with other views:
    auto high_scores = scores
        | std::views::filter([](const auto& p) { return p.second >= 80; })
        | std::views::keys;

    std::cout << "High scorers: ";
    for (const auto& name : high_scores)
        std::cout << name << " ";
    std::cout << "\n";
    // Output: High scorers: Alice Bob
}
```

Notice that `views::filter` still receives the full pair (you filter on `p.second`), and only after filtering do you project to just the keys. The ordering in the pipeline matters.

---

## Self-Assessment

### Q1: Iterate only the keys of a std::map<string,int> using std::views::keys

**Answer:**

`views::keys` gives you references to the map's actual key objects - no copies - and since map keys are inherently `const`, they come back as `const std::string&`.

```cpp
#include <map>
#include <string>
#include <ranges>
#include <iostream>
#include <algorithm>
#include <vector>

int main() {
    std::map<std::string, int> inventory{
        {"sword", 1}, {"potion", 5}, {"shield", 2}, {"arrow", 50}
    };

    // === Basic key iteration ===
    std::cout << "Items: ";
    for (const auto& item : inventory | std::views::keys)
        std::cout << item << " ";
    std::cout << "\n";
    // Output: Items: arrow potion shield sword
    // (map is sorted by key)

    // === Count keys matching a condition ===
    auto long_names = inventory | std::views::keys
        | std::views::filter([](const std::string& s) { return s.size() > 5; });

    std::cout << "Long names: ";
    for (const auto& name : long_names)
        std::cout << name << " ";
    std::cout << "\n";
    // Output: Long names: potion shield

    // === Check if a key exists using ranges ===
    auto keys = inventory | std::views::keys;
    bool has_sword = std::ranges::find(keys, "sword") != keys.end();
    std::cout << "Has sword: " << std::boolalpha << has_sword << "\n";
    // Output: Has sword: true

    // === views::keys gives REFERENCES to the actual keys ===
    // (const references, since map keys are const)
    for (const auto& key : inventory | std::views::keys) {
        // key is "const std::string&" - no copy made
        static_assert(std::is_same_v<decltype(key), const std::string&>);
    }
}
```

`views::keys` extracts the `.first` member of each pair in the map, producing a range of key references. No copies are made - the view operates lazily. Since map keys are inherently `const`, the references are `const std::string&`.

### Q2: Collect all values from an unordered_map using views::values | ranges::to\<vector\>

**Answer:**

Lazy views are great for iteration, but sometimes you need a concrete container. `ranges::to<std::vector>()` (C++23) materializes the lazy view in one step.

```cpp
#include <unordered_map>
#include <string>
#include <ranges>
#include <vector>
#include <iostream>
#include <algorithm>
#include <numeric>

int main() {
    std::unordered_map<std::string, double> prices{
        {"apple", 1.50}, {"banana", 0.75}, {"cherry", 3.00},
        {"date", 2.25}, {"elderberry", 5.00}
    };

    // === Collect values into a vector (C++23 ranges::to) ===
    auto price_list = prices | std::views::values
                             | std::ranges::to<std::vector>();
    // price_list is std::vector<double>

    // Sort the collected values (can't sort a view of an unordered_map)
    std::ranges::sort(price_list);

    std::cout << "Sorted prices: ";
    for (double p : price_list)
        std::cout << p << " ";
    std::cout << "\n";
    // Output: Sorted prices: 0.75 1.5 2.25 3 5

    // === Compute statistics from values ===
    double total = std::accumulate(price_list.begin(), price_list.end(), 0.0);
    std::cout << "Total: " << total << "\n";
    // Output: Total: 12.5

    // === Pre-C++23: collect manually ===
    std::vector<double> manual;
    for (double v : prices | std::views::values)
        manual.push_back(v);
    // or:
    // std::vector<double> manual(std::views::values(prices).begin(),
    //                             std::views::values(prices).end());

    // === Collect filtered values ===
    auto expensive = prices
        | std::views::filter([](const auto& p) { return p.second > 2.0; })
        | std::views::values
        | std::ranges::to<std::vector>();

    std::cout << "Expensive items count: " << expensive.size() << "\n";
    // Output: Expensive items count: 3  (cherry=3.00, date=2.25, elderberry=5.00)

    // === Modify values through views::values ===
    std::unordered_map<std::string, int> stock{
        {"apple", 10}, {"banana", 5}, {"cherry", 3}
    };

    // values gives mutable references for non-const maps:
    for (int& qty : stock | std::views::values)
        qty *= 2; // double all stock

    for (const auto& [item, qty] : stock)
        std::cout << item << ": " << qty << " ";
    std::cout << "\n";
    // Output: cherry: 6 banana: 10 apple: 20  (order varies)
}
```

`views::values` produces references to the `.second` member of each pair. With `ranges::to<std::vector>()` (C++23), you can materialize the lazy view into a concrete container in one expression. For mutable maps, `views::values` gives mutable references, allowing in-place modification.

### Q3: Explain that views::keys is equivalent to views::transform([](auto& p){ return p.first; })

**Answer:**

Understanding the equivalence helps you predict how `views::keys` behaves in edge cases, and it reveals why `views::elements<N>` generalizes the idea to any tuple index.

```cpp
#include <map>
#include <ranges>
#include <iostream>
#include <string>
#include <type_traits>
#include <tuple>
#include <utility>
#include <vector>

int main() {
    std::map<std::string, int> m{
        {"x", 1}, {"y", 2}, {"z", 3}
    };

    // === views::keys ===
    auto keys_view = m | std::views::keys;

    // === Equivalent transform ===
    auto transform_view = m | std::views::transform(
        [](auto& pair) -> const std::string& { return pair.first; }
    );

    // Both produce the same output:
    std::cout << "keys: ";
    for (const auto& k : keys_view) std::cout << k << " ";
    std::cout << "\nmanual: ";
    for (const auto& k : transform_view) std::cout << k << " ";
    std::cout << "\n";
    // Output:
    // keys: x y z
    // manual: x y z

    // === Why views::keys is better ===
    // 1. Shorter, more readable
    // 2. Returns REFERENCES (not copies) by default
    // 3. Standard name - everyone knows what it means
    // 4. Works with any tuple-like type (not just std::pair)

    // === Under the hood ===
    // views::keys is defined as views::elements<0>
    // views::values is defined as views::elements<1>
    // views::elements<N> uses std::get<N> on each element

    // This means it works with tuples too:
    std::vector<std::tuple<int, double, std::string>> data{
        {1, 3.14, "a"}, {2, 2.72, "b"}, {3, 1.41, "c"}
    };

    // views::keys = views::elements<0> -> gets the ints
    std::cout << "tuple firsts: ";
    for (int i : data | std::views::keys) // or std::views::elements<0>
        std::cout << i << " ";
    std::cout << "\n";
    // Output: tuple firsts: 1 2 3

    // views::elements<2> -> gets the strings (no shorthand name)
    std::cout << "tuple thirds: ";
    for (const auto& s : data | std::views::elements<2>)
        std::cout << s << " ";
    std::cout << "\n";
    // Output: tuple thirds: a b c

    // So views::keys/values are convenient aliases for the most common cases.
}
```

`views::keys` is defined as `views::elements<0>`, which in turn uses `std::get<0>()` on each element. This is semantically equivalent to a `views::transform` that extracts `.first`, but `views::keys` is more concise, returns references by default, and works with any tuple-like element (not just `std::pair`). `views::elements<N>` generalizes this to any tuple index.

---

## Notes

- **Reference semantics:** `views::keys` and `views::values` produce references to the original elements - no copies. For `const` maps, these are `const` references.
- **Mutable values:** For non-const maps, `views::values` gives mutable references, enabling `for (auto& v : m | views::values) v = 0;`.
- **Keys are always const** in `std::map` and `std::unordered_map` because modifying keys would corrupt the data structure.
- **`views::elements<N>`** is the general form: `views::keys` = `elements<0>`, `views::values` = `elements<1>`. Use `elements<2>`, `elements<3>`, etc. for tuples with more than 2 elements.
- **Works with any pair/tuple range:** Not limited to maps - works with `vector<pair>`, `vector<tuple>`, etc.
- Compile with `-std=c++20 -Wall -Wextra` (or `-std=c++23` for `ranges::to`).
