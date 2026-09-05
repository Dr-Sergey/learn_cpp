# Understand Explicit Instantiation and Prevent Implicit Instantiation

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/class_template>  

---

## Topic Overview

### What Is Template Instantiation

When you use a template with specific types, the compiler generates code for that specialization - this is **instantiation**. There are three flavors:

| Type | Syntax | When | Where |
| --- | --- | --- | --- |
| **Implicit** | `std::vector<int> v;` | On first use | Every TU that uses it |
| **Explicit definition** | `template class std::vector<int>;` | Forced | Exactly one TU |
| **Explicit declaration** (`extern template`) | `extern template class std::vector<int>;` | Suppressed | TUs that don't need to instantiate |

### The Problem: Redundant Instantiation

Without `extern template`, every `.cpp` file that uses `std::vector<int>` independently instantiates it. The linker deduplicates later, but the **compile cost** is paid in every translation unit. In a large project with dozens of TUs all using the same heavy template, this adds up fast:

```cpp
TU1: uses vector<int> -> instantiates vector<int> (slow)
TU2: uses vector<int> -> instantiates vector<int> (slow, redundant)
TU3: uses vector<int> -> instantiates vector<int> (slow, redundant)
Linker: keeps one, discards duplicates
```

### The Solution: `extern template`

The fix is to declare the instantiation `extern` in the header (telling every TU "don't instantiate this, someone else will"), and then provide the actual instantiation in exactly one `.cpp` file:

```cpp
header.h:          extern template class std::vector<int>;     // suppress
instantiation.cpp: template class std::vector<int>;            // define once
TU1: includes header -> no instantiation (fast)
TU2: includes header -> no instantiation (fast)
Linker: uses the one from instantiation.cpp
```

---

## Self-Assessment

### Q1: Use `extern template` to prevent implicit instantiation of `std::vector<HeavyType>` in multiple TUs

Here is the full pattern spread across multiple files. The key is that `extern template` in the header acts as a promise: "the definition exists elsewhere, don't generate it here." Only `heavy_type_instantiation.cpp` actually pays the instantiation cost:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// === heavy_type.h ===
#ifndef HEAVY_TYPE_H
#define HEAVY_TYPE_H

#include <vector>
#include <string>
#include <map>

struct HeavyType {
    std::string name;
    std::map<std::string, std::vector<int>> data;
    double values[100];

    HeavyType() = default;
    HeavyType(std::string n) : name(std::move(n)) {}
};

// EXTERN TEMPLATE DECLARATION:
// Tells the compiler: "Don't instantiate vector<HeavyType> here.
// It will be provided by another TU."
extern template class std::vector<HeavyType>;

#endif

// === heavy_type_instantiation.cpp ===
// This is the ONE TU that provides the explicit instantiation
#include "heavy_type.h"

// EXPLICIT INSTANTIATION DEFINITION:
// Forces full instantiation of all vector<HeavyType> member functions
template class std::vector<HeavyType>;

// === main.cpp ===
#include "heavy_type.h"  // includes extern template -> NO implicit instantiation
#include <iostream>

int main() {
    // Uses vector<HeavyType> but does NOT instantiate it (extern template)
    std::vector<HeavyType> items;
    items.emplace_back("Item1");
    items.emplace_back("Item2");
    items.push_back(HeavyType("Item3"));

    std::cout << "Items: " << items.size() << "\n";  // 3

    for (const auto& item : items) {
        std::cout << "  " << item.name << "\n";
    }

    return 0;
}

// === another_module.cpp ===
#include "heavy_type.h"  // Also gets extern template -> NO instantiation here either
#include <algorithm>

void sort_heavy(std::vector<HeavyType>& v) {
    std::sort(v.begin(), v.end(), [](const HeavyType& a, const HeavyType& b) {
        return a.name < b.name;
    });
}

// Without extern template:
//   main.cpp:           instantiates vector<HeavyType> (slow)
//   another_module.cpp: instantiates vector<HeavyType> (slow, redundant)
//   -> 2x compile cost, linker discards one
//
// With extern template:
//   heavy_type_instantiation.cpp: instantiates ONCE
//   main.cpp:                     no instantiation (fast)
//   another_module.cpp:           no instantiation (fast)
//   -> compile cost paid only once
```

The savings scale with the number of TUs. If ten files all include that header, only one of them pays to compile `vector<HeavyType>`.

### Q2: Provide an explicit instantiation definition in exactly one TU

This example shows the same pattern applied to a custom template. Notice that `DataProcessor<double>` is deliberately left out of the explicit instantiation list - any TU that uses it will instantiate it implicitly, which is sometimes what you want for less-common specializations:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// === my_template.h ===
#ifndef MY_TEMPLATE_H
#define MY_TEMPLATE_H

#include <iostream>
#include <string>

// Template class definition in header
template <typename T>
class DataProcessor {
    T data_;
public:
    DataProcessor(T d) : data_(std::move(d)) {}

    void process() {
        std::cout << "Processing: " << data_ << "\n";
    }

    T get() const { return data_; }

    void set(T d) { data_ = std::move(d); }
};

// Template function definition in header
template <typename T>
T transform(const T& input) {
    return input + input;  // requires operator+
}

// SUPPRESS implicit instantiation for common types:
extern template class DataProcessor<int>;
extern template class DataProcessor<std::string>;
extern template int transform<int>(const int&);
extern template std::string transform<std::string>(const std::string&);

#endif

// === my_template.cpp ===
// EXACTLY ONE TU provides the explicit instantiation definitions
#include "my_template.h"

// Explicit instantiation definitions - generates ALL member functions
template class DataProcessor<int>;
template class DataProcessor<std::string>;

// Explicit instantiation of function templates
template int transform<int>(const int&);
template std::string transform<std::string>(const std::string&);

// Note: DataProcessor<double> is NOT explicitly instantiated
// -> any TU using DataProcessor<double> will implicitly instantiate it

// === main.cpp ===
#include "my_template.h"
#include <iostream>

int main() {
    // Uses pre-instantiated specializations (no compile cost here)
    DataProcessor<int> dp1(42);
    dp1.process();              // Processing: 42

    DataProcessor<std::string> dp2("hello");
    dp2.process();              // Processing: hello

    std::cout << transform(10) << "\n";            // 20
    std::cout << transform(std::string("ab")) << "\n"; // abab

    // DataProcessor<double> is NOT extern template'd
    // -> implicitly instantiated HERE (compile cost paid)
    DataProcessor<double> dp3(3.14);
    dp3.process();              // Processing: 3.14

    return 0;
}
```

### Q3: Measure compile-time improvement from using explicit instantiation on a large template

This example makes the trade-off concrete. The comment table at the bottom shows what happens as the number of TUs grows - without `extern template`, instantiation cost scales linearly with TU count:

```cpp
// === Practical example: measuring the effect ===
// This demonstrates the concept; actual measurements require multi-TU builds.

#include <iostream>
#include <chrono>
#include <vector>
#include <string>
#include <map>
#include <set>

// Heavy template: many member functions to instantiate
template <typename K, typename V>
class HeavyCache {
    std::map<K, V> store_;
    std::vector<K> access_order_;
    std::set<K> dirty_keys_;

public:
    void put(const K& key, const V& val) {
        store_[key] = val;
        access_order_.push_back(key);
        dirty_keys_.insert(key);
    }

    V get(const K& key) const {
        return store_.at(key);
    }

    size_t size() const { return store_.size(); }
    size_t dirty_count() const { return dirty_keys_.size(); }

    void flush() {
        dirty_keys_.clear();
    }

    std::vector<K> get_keys() const {
        std::vector<K> keys;
        for (const auto& [k, v] : store_) keys.push_back(k);
        return keys;
    }
};

// In a real project:
// header.h:
//   extern template class HeavyCache<std::string, int>;
//   extern template class HeavyCache<int, std::string>;
//
// instantiation.cpp:
//   template class HeavyCache<std::string, int>;
//   template class HeavyCache<int, std::string>;
//
// Typical compile-time savings:
//
// ┌─────────────────────┬──────────────┬──────────────────┐
// │ Scenario            │ Without ext  │ With ext tmplt   │
// ├─────────────────────┼──────────────┼──────────────────┤
// │ 1 TU                │ 1.0x         │ 1.0x (no gain)   │
// │ 5 TUs using it      │ 5.0x         │ 1.0x + link      │
// │ 20 TUs using it     │ 20.0x        │ 1.0x + link      │
// │ 100 TUs using it    │ 100.0x       │ 1.0x + link      │
// └─────────────────────┴──────────────┴──────────────────┘
//
// Real-world: 10-50% compile time reduction on large projects

int main() {
    HeavyCache<std::string, int> cache;
    cache.put("alpha", 1);
    cache.put("beta", 2);
    cache.put("gamma", 3);

    std::cout << "Cache size:  " << cache.size() << "\n";         // 3
    std::cout << "Dirty keys:  " << cache.dirty_count() << "\n";  // 3
    std::cout << "beta value:  " << cache.get("beta") << "\n";    // 2

    cache.flush();
    std::cout << "After flush, dirty: " << cache.dirty_count() << "\n";  // 0

    auto keys = cache.get_keys();
    std::cout << "Keys: ";
    for (const auto& k : keys) std::cout << k << " ";
    std::cout << "\n";  // alpha beta gamma

    std::cout << "\n=== When to use extern template ===\n";
    std::cout << "1. Template used in many TUs (>5)\n";
    std::cout << "2. Template is 'heavy' (many members, complex types)\n";
    std::cout << "3. Common specializations are known (string, int, etc.)\n";
    std::cout << "4. Build times are a bottleneck\n";
    std::cout << "\nDo NOT use when:\n";
    std::cout << "- Template is header-only library used in 1-2 TUs\n";
    std::cout << "- Types are unknown until user provides them\n";
    std::cout << "- You need all specializations to be visible for inlining\n";

    return 0;
}
```

One thing worth noting: explicit instantiation does not change runtime behavior at all. It is purely a compile-time optimization. If you remove `extern template` from an otherwise correct program, it still works - it just compiles slower.

---

## Notes

- `extern template class X<T>;` **suppresses** implicit instantiation in a TU.
- `template class X<T>;` **forces** explicit instantiation in exactly one TU.
- The `extern` declaration goes in the **header**; the definition goes in **one .cpp file**.
- Savings scale with number of TUs: N TUs -> ~Nx compile time without -> ~1x with extern template.
- Does not affect runtime behavior - only compile/link performance.
- Works for both class templates and function templates.
- For function templates: `extern template int foo<int>(int);` / `template int foo<int>(int);`
