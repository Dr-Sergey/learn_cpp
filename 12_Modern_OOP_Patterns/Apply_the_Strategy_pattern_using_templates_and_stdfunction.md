# Apply the Strategy pattern using templates and std::function

**Category:** Modern OOP Patterns  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/function>  

---

## Topic Overview

The Strategy pattern encapsulates interchangeable algorithms behind a common interface. In modern C++, there are two primary approaches: **compile-time strategies** (template parameters - zero overhead, fixed at compile time) and **runtime strategies** (`std::function` - flexible, switchable at runtime).

Understanding which to use comes down to one question: do you know the strategy when you compile, or does it depend on something that only exists at runtime (user input, config files, plugins)?

### Two Flavors

Here's the side-by-side mental model. The left is faster and the right is more flexible:

```cpp
Compile-time Strategy (Template):      Runtime Strategy (std::function):
┌─────────────────────────────┐       ┌─────────────────────────────┐
│ template <typename Compare>  │       │ class Sorter {              │
│ class Sorter {               │       │   std::function<bool(int,int)> │
│   void sort(vec) {           │       │     compare_;               │
│     std::sort(v, Compare{}); │       │   void sort(vec) {          │
│   }                          │       │     std::sort(v, compare_); │
│ };                           │       │   }                         │
│ // Sorter<std::less<>> s;    │       │   void set(cmp) { compare_ = cmp; } │
│ // Type is fixed forever     │       │   // Can change at runtime  │
└─────────────────────────────┘       └─────────────────────────────┘
```

---

## Self-Assessment

### Q1: Implement a `Sorter` class parameterized on a comparison strategy using a template policy

The strategy types here are simple empty structs with an `operator()`. The compiler sees exactly what the comparison does and can inline it completely - no function call overhead at all.

**Solution - Template-Based Strategy:**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

// Comparison strategies as policy types
struct Ascending {
    template <typename T>
    bool operator()(const T& a, const T& b) const { return a < b; }
};

struct Descending {
    template <typename T>
    bool operator()(const T& a, const T& b) const { return a > b; }
};

struct ByLength {
    bool operator()(const std::string& a, const std::string& b) const {
        return a.size() < b.size();
    }
};

// Template-parameterized Sorter
template <typename CompareStrategy = Ascending>
class Sorter {
    CompareStrategy comp_;  // may be empty (EBO-eligible)
public:
    template <typename Container>
    void sort(Container& c) const {
        std::sort(c.begin(), c.end(), comp_);
    }

    template <typename Container>
    void print(const Container& c, const std::string& label) const {
        std::cout << label << ": ";
        for (const auto& x : c) std::cout << x << " ";
        std::cout << "\n";
    }
};

int main() {
    std::vector<int> nums = {5, 2, 8, 1, 9, 3};

    Sorter<Ascending> asc_sorter;
    auto v1 = nums;
    asc_sorter.sort(v1);
    asc_sorter.print(v1, "Ascending");

    Sorter<Descending> desc_sorter;
    auto v2 = nums;
    desc_sorter.sort(v2);
    desc_sorter.print(v2, "Descending");

    // String sorting by length
    std::vector<std::string> words = {"banana", "fig", "apple", "kiwi"};
    Sorter<ByLength> len_sorter;
    len_sorter.sort(words);
    len_sorter.print(words, "By length");
}
// Expected output:
//   Ascending: 1 2 3 5 8 9
//   Descending: 9 8 5 3 2 1
//   By length: fig kiwi apple banana
```

Each `Sorter<X>` is a distinct type, which means the strategy is baked in at compile time. You get maximum performance, but you can't swap strategies without constructing a new object of a different type.

---

### Q2: Contrast compile-time strategy (template) vs runtime strategy (`std::function` member)

This example puts both approaches side by side so you can see the key difference: the runtime version lets you call `set_strategy` and change behavior on the fly, which is simply impossible with templates.

**Solution - Side-by-Side Comparison:**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

// ===== Compile-time strategy =====
template <typename Cmp>
class CompileTimeSorter {
public:
    void sort(std::vector<int>& v) {
        std::sort(v.begin(), v.end(), Cmp{});  // inlined, no overhead
    }
    // Type is baked in: CompileTimeSorter<std::less<>> != CompileTimeSorter<std::greater<>>
};

// ===== Runtime strategy =====
class RuntimeSorter {
    std::function<bool(int, int)> compare_;
public:
    explicit RuntimeSorter(std::function<bool(int, int)> cmp)
        : compare_(std::move(cmp)) {}

    void set_strategy(std::function<bool(int, int)> cmp) {
        compare_ = std::move(cmp);  // can change at runtime!
    }

    void sort(std::vector<int>& v) {
        std::sort(v.begin(), v.end(), compare_);  // type-erased call
    }
};

int main() {
    std::vector<int> data = {5, 2, 8, 1, 9};

    // Compile-time: type encodes the strategy
    CompileTimeSorter<std::less<int>> ct_sorter;
    auto v1 = data;
    ct_sorter.sort(v1);
    // Can't change to descending without a new object of different type

    // Runtime: strategy can change dynamically
    RuntimeSorter rt_sorter([](int a, int b) { return a < b; });
    auto v2 = data;
    rt_sorter.sort(v2);
    std::cout << "Runtime (asc): ";
    for (int x : v2) std::cout << x << " ";
    std::cout << "\n";

    // Switch to descending at runtime!
    rt_sorter.set_strategy([](int a, int b) { return a > b; });
    v2 = data;
    rt_sorter.sort(v2);
    std::cout << "Runtime (desc): ";
    for (int x : v2) std::cout << x << " ";
    std::cout << "\n";

    // Read user input to decide strategy (impossible with compile-time)
    int choice = 1;  // simulate user choosing "1 = ascending"
    rt_sorter.set_strategy(choice == 1
        ? std::function<bool(int,int)>([](int a, int b) { return a < b; })
        : std::function<bool(int,int)>([](int a, int b) { return a > b; }));
}
// Expected output:
//   Runtime (asc): 1 2 5 8 9
//   Runtime (desc): 9 8 5 2 1
```

| Feature | Compile-Time (Template) | Runtime (`std::function`) |
| --- | --- | --- |
| **Performance** | Zero overhead - inlined | ~20-40 bytes per function, indirect call |
| **Flexibility** | Fixed at compile time | Changeable at runtime |
| **Type identity** | Different types per strategy | Same type regardless of strategy |
| **Container storage** | Can't mix different strategies | Can store in `vector<RuntimeSorter>` |
| **Binary size** | Separate code per instantiation | Single code path |
| **Use case** | Known at compile time, hot path | User-configurable, plugin-like |

---

### Q3: Show how `std::function` enables storing strategies in a container for dynamic dispatch

This is where `std::function` really earns its place. Because it erases the concrete type of any callable, you can store lambdas, function pointers, and functors all in the same `unordered_map`. A template approach can't do this - different strategies are different types and can't live in the same container.

**Solution - Strategy Registry:**

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <string>
#include <unordered_map>

// Strategy type: transforms a string
using TextTransform = std::function<std::string(const std::string&)>;

// Strategies stored in a map - selected by name at runtime
class TextProcessor {
    std::unordered_map<std::string, TextTransform> strategies_;
    std::vector<std::string> pipeline_;  // ordered list of strategy names to apply

public:
    void register_strategy(const std::string& name, TextTransform strategy) {
        strategies_[name] = std::move(strategy);
    }

    void add_to_pipeline(const std::string& name) {
        pipeline_.push_back(name);
    }

    void clear_pipeline() { pipeline_.clear(); }

    std::string process(std::string text) const {
        for (const auto& name : pipeline_) {
            auto it = strategies_.find(name);
            if (it != strategies_.end()) {
                text = it->second(text);
            }
        }
        return text;
    }
};

int main() {
    TextProcessor proc;

    // Register various strategies (could come from plugins, config, etc.)
    proc.register_strategy("uppercase", [](const std::string& s) {
        std::string result = s;
        for (char& c : result) c = std::toupper(c);
        return result;
    });

    proc.register_strategy("trim", [](const std::string& s) {
        size_t start = s.find_first_not_of(" \t");
        size_t end = s.find_last_not_of(" \t");
        return (start == std::string::npos) ? "" : s.substr(start, end - start + 1);
    });

    proc.register_strategy("exclaim", [](const std::string& s) {
        return s + "!!!";
    });

    proc.register_strategy("bracket", [](const std::string& s) {
        return "[" + s + "]";
    });

    // Build a pipeline dynamically
    proc.add_to_pipeline("trim");
    proc.add_to_pipeline("uppercase");
    proc.add_to_pipeline("exclaim");
    proc.add_to_pipeline("bracket");

    std::string result = proc.process("  hello world  ");
    std::cout << "Result: " << result << "\n";

    // Change pipeline at runtime
    proc.clear_pipeline();
    proc.add_to_pipeline("trim");
    proc.add_to_pipeline("bracket");
    result = proc.process("  hello world  ");
    std::cout << "Result: " << result << "\n";
}
// Expected output:
//   Result: [HELLO WORLD!!!]
//   Result: [hello world]
```

`std::function` erases the concrete callable type, allowing storage in homogeneous containers. You can't do this with templates alone - `std::function<R(Args...)>` is the bridge between compile-time callables and runtime collections.

---

## Notes

- **`std::function` has overhead:** type erasure, possible heap allocation, non-inlinable indirect call. For hot paths, prefer template parameters.
- **`std::move_only_function` (C++23):** like `std::function` but supports move-only callables - useful for strategies holding `unique_ptr` or other non-copyable state.
- **Stateful strategies:** Both template and `std::function` strategies can carry state - the template approach stores state in the policy object, `std::function` captures it in a lambda.
- **Combine both approaches:** Use template strategies for core algorithms, `std::function` for user-facing extensibility points.
- **`std::sort` itself uses the Strategy pattern** - its comparator parameter is a compile-time strategy (template parameter).
