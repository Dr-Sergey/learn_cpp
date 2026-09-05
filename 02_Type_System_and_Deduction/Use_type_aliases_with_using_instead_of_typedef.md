# Use Type Aliases with `using` Instead of `typedef`

**Category:** Type System and Deduction  
**Standard:** Not version-specific  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP11.md#type-aliases>  

---

## Topic Overview

### Why `using` Over `typedef`

C++11 introduced `using` alias declarations as a modern replacement for `typedef`. Both create type aliases, but `using` is strictly more powerful. The key upgrade is **alias templates** - something `typedef` simply cannot do.

| Feature | `typedef` | `using` |
| --- | --- | --- |
| Simple type alias | Yes | Yes |
| Function pointer alias | Yes (ugly syntax) | Yes (cleaner) |
| **Alias templates** (template aliases) | **No** | **Yes** |
| Readability | Poor for complex types | Left-to-right reading |

### Syntax Comparison

The readability difference is most visible with function pointers - `typedef` buries the name in the middle of the declaration, while `using` puts it cleanly on the left:

```cpp
// Simple alias
typedef unsigned long ulong;       // old
using ulong = unsigned long;       // new (clearer: name = type)

// Function pointer
typedef void (*Callback)(int, double);          // old — where's the name?
using Callback = void(*)(int, double);          // new — name on the left

// Array
typedef int IntArray[10];           // old
using IntArray = int[10];           // new

// Complex nested type
typedef std::map<std::string, std::vector<int>> MapType;   // old
using MapType = std::map<std::string, std::vector<int>>;   // new
```

### Alias Templates (Only With `using`)

This is the real reason to prefer `using`. You can parameterize a type alias over a template parameter, which `typedef` fundamentally can't do:

```cpp
// This is IMPOSSIBLE with typedef:
template<typename T>
using Vec = std::vector<T>;

Vec<int> v;     // std::vector<int>
Vec<double> d;  // std::vector<double>

// With typedef, you'd need a whole struct wrapper:
template<typename T>
struct VecTypedef {
    typedef std::vector<T> type;
};
VecTypedef<int>::type v;  // ugly!
```

The `::type` suffix required by the struct workaround is a real pain - you'd need `typename VecTypedef<T>::type` inside templates, which is both noisy and requires an extra `typename` keyword. Alias templates eliminate all of that.

---

## Self-Assessment

### Q1: Rewrite a complex nested typedef chain using `using` alias declarations

Pay attention to how much easier it is to read the `using` versions, especially for the function pointer and member function pointer cases:

```cpp
#include <iostream>
#include <map>
#include <vector>
#include <string>
#include <functional>
#include <memory>

// === OLD: typedef chain (hard to read) ===
typedef std::map<std::string, std::vector<int>> OldScoreMap;
typedef std::pair<const std::string, std::vector<int>> OldScoreEntry;
typedef void (*OldHandler)(const OldScoreMap&);
typedef std::map<std::string, OldHandler> OldDispatch;

// === NEW: using aliases (clear left-to-right) ===
using ScoreMap    = std::map<std::string, std::vector<int>>;
using ScoreEntry  = std::pair<const std::string, std::vector<int>>;
using Handler     = void(*)(const ScoreMap&);
using Dispatch    = std::map<std::string, Handler>;

// Even more complex: nested function pointer typedefs
typedef int (*OldComparator)(const void*, const void*);
typedef void (*OldSorter)(void*, size_t, size_t, OldComparator);

using Comparator = int(*)(const void*, const void*);
using Sorter     = void(*)(void*, size_t, size_t, Comparator);

// Member function pointer
struct Widget {
    int process(double d) { return static_cast<int>(d * 2); }
};

typedef int (Widget::*OldMethod)(double);
using Method = int (Widget::*)(double);

void print_scores(const ScoreMap& scores) {
    for (const auto& [name, vals] : scores) {
        std::cout << name << ": ";
        for (int v : vals) std::cout << v << " ";
        std::cout << "\n";
    }
}

int main() {
    // Using the clean aliases
    ScoreMap scores;
    scores["Alice"] = {95, 87, 92};
    scores["Bob"]   = {88, 91, 76};

    Handler handler = print_scores;
    handler(scores);

    // Member function pointer
    Method m = &Widget::process;
    Widget w;
    int result = (w.*m)(3.14);
    std::cout << "Widget::process(3.14) = " << result << "\n";

    return 0;
}
```

**Output:**

```text
Alice: 95 87 92
Bob: 88 91 76
Widget::process(3.14) = 6
```

### Q2: Show how template aliases (alias templates) cannot be expressed with `typedef`

This example demonstrates all the common alias template patterns, then shows the `typedef` struct workaround you'd be stuck with without `using`:

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <string>
#include <memory>
#include <functional>

// === ALIAS TEMPLATES with using (C++11) ===

// 1. Generic container alias
template<typename T>
using Vec = std::vector<T>;

// 2. Map with string keys
template<typename V>
using StringMap = std::map<std::string, V>;

// 3. Smart pointer alias
template<typename T>
using Ptr = std::unique_ptr<T>;

// 4. Callback type
template<typename R, typename... Args>
using Func = std::function<R(Args...)>;

// 5. Allocator-aware alias
template<typename T, typename Alloc = std::allocator<T>>
using List = std::vector<T, Alloc>;

// === TYPEDEF WORKAROUND (ugly, verbose) ===
// You CANNOT write: template<typename T> typedef std::vector<T> VecOld;
// You must wrap in a struct:

template<typename T>
struct VecOld {
    typedef std::vector<T> type;  // not Vec<T>, must use ::type
};

template<typename V>
struct StringMapOld {
    typedef std::map<std::string, V> type;
};

int main() {
    // Clean alias template usage
    Vec<int> nums = {1, 2, 3};
    StringMap<double> grades;
    grades["Alice"] = 95.5;
    grades["Bob"] = 87.3;

    Ptr<int> p = std::make_unique<int>(42);

    Func<int, int, int> add = [](int a, int b) { return a + b; };

    std::cout << "Vec<int>: ";
    for (int n : nums) std::cout << n << " ";
    std::cout << "\n";

    std::cout << "StringMap: Alice=" << grades["Alice"] << "\n";
    std::cout << "Ptr: " << *p << "\n";
    std::cout << "Func: add(3,4)=" << add(3, 4) << "\n";

    // Old typedef way — verbose and has ::type overhead
    VecOld<int>::type old_nums = {4, 5, 6};
    StringMapOld<int>::type old_map;
    old_map["key"] = 42;

    std::cout << "\n(typedef way requires ::type everywhere)\n";

    // Alias templates work naturally in template contexts:
    auto print_vec = [](const auto& v) {
        for (const auto& x : v) std::cout << x << " ";
        std::cout << "\n";
    };

    Vec<double> doubles = {1.1, 2.2, 3.3};
    Vec<std::string> words = {"hello", "world"};
    print_vec(doubles);
    print_vec(words);

    return 0;
}
```

The difference in call site ergonomics is stark: `Vec<int>` vs `VecOld<int>::type`. In a template function body the `typedef` form also requires a `typename` prefix, adding even more noise.

**Output:**

```text
Vec<int>: 1 2 3
StringMap: Alice=95.5
Ptr: 42
Func: add(3,4)=7

(typedef way requires ::type everywhere)
1.1 2.2 3.3
hello world
```

### Q3: Write a type alias that abstracts an allocator-aware container's iterator type

Alias templates are useful for hiding verbose iterator types so your function signatures stay readable. If you later swap the underlying container or add a custom allocator, all code using the alias updates automatically:

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <deque>
#include <string>
#include <memory>

// Alias templates to abstract container details
template<typename T, typename Alloc = std::allocator<T>>
using ContainerIter = typename std::vector<T, Alloc>::iterator;

template<typename T, typename Alloc = std::allocator<T>>
using ContainerConstIter = typename std::vector<T, Alloc>::const_iterator;

// More general: parameterize the container itself
template<template<typename, typename> class Container,
         typename T, typename Alloc = std::allocator<T>>
using Iter = typename Container<T, Alloc>::iterator;

template<template<typename, typename> class Container,
         typename T, typename Alloc = std::allocator<T>>
using ConstIter = typename Container<T, Alloc>::const_iterator;

template<template<typename, typename> class Container,
         typename T, typename Alloc = std::allocator<T>>
using ValueType = typename Container<T, Alloc>::value_type;

// Algorithm using the aliases
template<template<typename, typename> class Container,
         typename T, typename Alloc>
void print_range(Iter<Container, T, Alloc> begin,
                 Iter<Container, T, Alloc> end) {
    std::cout << "[";
    for (auto it = begin; it != end; ++it) {
        if (it != begin) std::cout << ", ";
        std::cout << *it;
    }
    std::cout << "]\n";
}

// Practical: hide allocator in domain types
template<typename T>
using FastVec = std::vector<T>;  // could swap allocator transparently

template<typename T>
using FastVecIter = typename FastVec<T>::iterator;

template<typename T>
using FastVecConstIter = typename FastVec<T>::const_iterator;

// Find in a FastVec — iterator type is clean
template<typename T>
FastVecIter<T> find_in(FastVec<T>& vec, const T& value) {
    for (auto it = vec.begin(); it != vec.end(); ++it) {
        if (*it == value) return it;
    }
    return vec.end();
}

int main() {
    // Using aliased iterator types
    FastVec<int> nums = {10, 20, 30, 40, 50};

    FastVecIter<int> it = find_in(nums, 30);
    if (it != nums.end()) {
        std::cout << "Found: " << *it << " at index "
                  << std::distance(nums.begin(), it) << "\n";
    }

    // Different container, same alias pattern
    FastVec<std::string> words = {"hello", "world", "foo"};
    auto sit = find_in(words, std::string("world"));
    if (sit != words.end()) {
        std::cout << "Found: " << *sit << "\n";
    }

    // Benefit: if we change FastVec to use a custom allocator,
    // all iterator types automatically update:
    // template<typename T>
    // using FastVec = std::vector<T, MyPoolAllocator<T>>;
    // ^ All code using FastVecIter<T> still compiles correctly

    std::cout << "\nAlias templates make allocator-aware code clean.\n";

    return 0;
}
```

The comment at the end points to the real payoff: if you ever need to switch `FastVec` to use a pool allocator, you change one line in the alias definition and everything else recompiles correctly. Without the alias layer, you'd have to track down every `std::vector<T>::iterator` mention by hand.

**Output:**

```text
Found: 30 at index 2
Found: world

Alias templates make allocator-aware code clean.
```

---

## Notes

- **Always prefer `using` over `typedef`** in modern C++. There is no case where `typedef` is better.
- `using` aliases are identical to `typedef` for simple cases - they produce the same type, not a new type.
- Alias templates participate in template argument deduction and work naturally with concepts.
- For class members, `using` also handles inheriting constructors (`using Base::Base;`) and bringing base class names into scope (`using Base::method;`) - different uses of the same keyword.
