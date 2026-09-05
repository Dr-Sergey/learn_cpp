# Understand Template Instantiation and Its Compile-Time Cost

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/class_template>  

---

## Topic Overview

### How Templates Generate Code

Here's the mental model that explains most of the compile-time cost story: templates are **not code** - they are **recipes**. The compiler generates actual code (instantiation) for each unique set of template arguments you actually use. You write one template, but the compiler may generate many distinct classes from it.

```cpp
Template:    vector<T>
Usage:       vector<int>, vector<string>, vector<double>
Generated:   3 separate classes with all member functions
```

Each of those three is a completely independent class - they share no machine code at runtime.

### Why It's Expensive

Once you understand that each unique argument set generates a new class, the costs in the table below follow naturally. Nested templates multiply the problem because each layer adds a new dimension of unique combinations:

| Factor | Impact |
| --- | --- |
| Each unique `T` -> separate instantiation | Multiplies code generation |
| All member functions instantiated (if used) | More work per instantiation |
| Nested templates (e.g., `vector<map<string,int>>`) | Exponential nesting |
| Header-only libraries | Every TU pays the cost |
| Linker deduplication | Linker discards duplicates but has to compare |

### Mitigation Strategies

The strategies below range from a compiler directive (`extern template`) to architectural choices like type erasure. Pick the right tool based on how much control you have over which types get instantiated:

| Strategy | How | When |
| --- | --- | --- |
| `extern template` | Suppress implicit instantiation | Common specializations across many TUs |
| Explicit instantiation | Force instantiation in one TU | Paired with `extern template` |
| Type erasure | Single instantiation via `void*` or `std::function` | When performance allows |
| Forward declarations | Avoid including heavy headers | Implementation files only |
| Limit unique instantiations | Use `int` instead of `short`, `long` | When types are interchangeable |

---

## Self-Assessment

### Q1: Demonstrate how explicit instantiation in a `.cpp` file reduces compile time and binary size

The key idea here is the two-step pattern: `extern template` in the header says "don't generate this here", and `template class` in one `.cpp` says "generate it exactly here". Every other translation unit gets the compiled result for free:

```cpp
// === widget.h ===
#ifndef WIDGET_H
#define WIDGET_H

#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <numeric>

// Heavy template with many member functions
template <typename T>
class DataStore {
    std::vector<T> data_;
    std::string name_;

public:
    DataStore(std::string name) : name_(std::move(name)) {}

    void add(const T& item) { data_.push_back(item); }
    void add(T&& item) { data_.push_back(std::move(item)); }

    size_t size() const { return data_.size(); }
    bool empty() const { return data_.empty(); }
    const T& at(size_t i) const { return data_.at(i); }

    void sort() { std::sort(data_.begin(), data_.end()); }

    T sum() const {
        return std::accumulate(data_.begin(), data_.end(), T{});
    }

    void print() const {
        std::cout << name_ << ": [";
        for (size_t i = 0; i < data_.size(); ++i) {
            if (i) std::cout << ", ";
            std::cout << data_[i];
        }
        std::cout << "]\n";
    }

    void clear() { data_.clear(); }

    auto begin() const { return data_.begin(); }
    auto end() const { return data_.end(); }
};

// SUPPRESS implicit instantiation for common types
extern template class DataStore<int>;
extern template class DataStore<double>;
extern template class DataStore<std::string>;

#endif

// === widget.cpp (explicit instantiation TU) ===
// #include "widget.h"
//
// // Force ONE instantiation of each — all member functions generated HERE
// template class DataStore<int>;
// template class DataStore<double>;
// template class DataStore<std::string>;

// === main.cpp ===
// #include "widget.h"
// DataStore<int> ds("numbers");  // Uses pre-instantiated code — NO compile cost
// DataStore<double> ds2("reals"); // Same — already instantiated in widget.cpp

// === Single-file demo ===

// Explicit instantion (normally in widget.cpp):
template class DataStore<int>;
template class DataStore<double>;
template class DataStore<std::string>;

int main() {
    DataStore<int> ints("ints");
    ints.add(30);
    ints.add(10);
    ints.add(20);
    ints.sort();
    ints.print();  // ints: [10, 20, 30]
    std::cout << "Sum: " << ints.sum() << "\n";  // 60

    DataStore<std::string> words("words");
    words.add("cherry");
    words.add("apple");
    words.add("banana");
    words.sort();
    words.print();  // words: [apple, banana, cherry]

    std::cout << "\n=== Cost analysis ===\n";
    std::cout << "Without extern template:\n";
    std::cout << "  Each TU using DataStore<int> generates ALL member functions\n";
    std::cout << "  10 TUs x ~10 functions = ~100 function compilations\n";
    std::cout << "  Linker deduplicates -> wasted work\n\n";
    std::cout << "With extern template:\n";
    std::cout << "  1 TU generates all member functions (widget.cpp)\n";
    std::cout << "  9 other TUs: zero template compilation for DataStore<int>\n";
    std::cout << "  10x compile speedup for this template\n";

    return 0;
}
```

### Q2: Explain why each unique set of template arguments produces a separate instantiation

The cleanest proof that `Box<int>` and `Box<double>` are truly separate classes is that each gets its own static member variable at a different address. They don't share any state:

```cpp
#include <iostream>
#include <typeinfo>

// Each distinct set of template arguments creates a SEPARATE class/function
template <typename T>
class Box {
public:
    static int instance_count;

    Box() { ++instance_count; }
    ~Box() { --instance_count; }

    static void info() {
        std::cout << "Box<" << typeid(T).name() << ">: "
                  << "instance_count addr = " << &instance_count << "\n";
    }
};

// Each instantiation gets its OWN static member
template <typename T> int Box<T>::instance_count = 0;

// Even same-sized types get separate instantiations:
// Box<int> and Box<unsigned int> are DIFFERENT classes
// even though int and unsigned int are both 4 bytes

template <typename T, typename U>
class Pair {
public:
    static void info() {
        std::cout << "Pair<" << typeid(T).name() << "," << typeid(U).name() << ">\n";
    }
};
// Pair<int,int>, Pair<int,double>, Pair<double,int> = THREE classes

int main() {
    std::cout << "=== Each unique type = separate instantiation ===\n\n";

    Box<int>::info();
    Box<double>::info();
    Box<char>::info();

    // Prove they're different classes: different static member addresses
    std::cout << "\nStatic members are separate:\n";
    Box<int> a, b;
    Box<double> c;
    std::cout << "Box<int>::count = " << Box<int>::instance_count << "\n";     // 2
    std::cout << "Box<double>::count = " << Box<double>::instance_count << "\n"; // 1

    std::cout << "\n=== Why this happens ===\n";
    std::cout << "Templates are compile-time code generators.\n";
    std::cout << "vector<int> and vector<string> share NO code at runtime.\n";
    std::cout << "The compiler generates completely independent classes.\n\n";

    std::cout << "This means:\n";
    std::cout << "  vector<int>    -> ~20 functions generated\n";
    std::cout << "  vector<string> -> ~20 functions generated (different code!)\n";
    std::cout << "  vector<double> -> ~20 functions generated\n";
    std::cout << "  Total: ~60 functions for 3 instantiations\n\n";

    std::cout << "Compile-time cost grows linearly with unique instantiations.\n";
    std::cout << "Binary size also grows (though linker can merge identical code).\n";

    return 0;
}
```

### Q3: Use `extern template` to suppress implicit instantiation across translation units

Notice the two-step protocol in the comments: the header tells every including TU to stay hands-off, and exactly one `.cpp` file does the actual work. That's the entire pattern:

```cpp
#include <iostream>
#include <vector>
#include <string>

// In practice, this goes in a header:
//
// // my_types.h
// #include <vector>
// #include <string>
//
// // Tell all TUs: don't instantiate these — we'll provide them
// extern template class std::vector<int>;
// extern template class std::vector<double>;
// extern template class std::vector<std::string>;
//
// // my_types.cpp — the ONE TU that instantiates them:
// #include "my_types.h"
// template class std::vector<int>;
// template class std::vector<double>;
// template class std::vector<std::string>;

// For this demo, showing the pattern:

template <typename T>
class Logger {
    std::vector<std::string> entries_;
    T context_;
public:
    Logger(T ctx) : context_(std::move(ctx)) {}

    void log(std::string msg) {
        entries_.push_back("[" + std::to_string(entries_.size()) + "] " + std::move(msg));
    }

    void dump() const {
        std::cout << "Logger<" << typeid(T).name() << "> (" << entries_.size() << " entries):\n";
        for (const auto& e : entries_) std::cout << "  " << e << "\n";
    }
};

// Force instantiation here (normally in a .cpp file)
template class Logger<int>;
template class Logger<std::string>;

int main() {
    Logger<int> log1(42);
    log1.log("started");
    log1.log("processing");
    log1.dump();

    Logger<std::string> log2("system");
    log2.log("initialized");
    log2.dump();

    std::cout << "\n=== extern template summary ===\n";
    std::cout << "Step 1: In header:  extern template class Logger<int>;\n";
    std::cout << "  -> Suppresses instantiation in every TU that includes header\n\n";
    std::cout << "Step 2: In one .cpp: template class Logger<int>;\n";
    std::cout << "  -> Forces instantiation in this single TU\n\n";
    std::cout << "Result: Logger<int> compiled ONCE, linked everywhere.\n";
    std::cout << "Compile-time savings: proportional to (#TUs - 1)\n";

    return 0;
}
```

---

## Notes

- Templates produce **separate code** for each unique argument set - `vector<int>` and `vector<double>` are unrelated classes.
- Compile-time cost: proportional to (number of unique instantiations) x (number of member functions) x (number of TUs).
- `extern template` suppresses implicit instantiation; explicit instantiation in one `.cpp` provides the code.
- Only works for known specializations - `extern template` cannot cover user-provided types.
- Binary size concern: many instantiations -> code bloat. Linker's ICF (Identical Code Folding) helps but isn't guaranteed.
- Advanced: type erasure (e.g., `std::function`, `std::any`) avoids template bloat at the cost of runtime overhead.
