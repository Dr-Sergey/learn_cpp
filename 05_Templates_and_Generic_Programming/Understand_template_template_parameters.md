# Understand Template Template Parameters

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/template_parameters>  

---

## Topic Overview

### What Are Template Template Parameters

A **template template parameter** is a template parameter that is itself a template. The key insight is that it lets you parameterize a class or function on the **container template** rather than a concrete instantiated type. The difference is subtle but important: you're saying "I want something that acts like a container template" rather than "I want a specific container of a specific element type."

```cpp
// Regular:      takes a TYPE
template <typename T> class Box {};

// Template template: takes a TEMPLATE
template <template <typename...> class Container, typename T>
class Stack {
    Container<T> storage_;    // Instantiate the Container with T
};

Stack<std::vector, int> s;    // storage_ is vector<int>
Stack<std::deque, double> s2; // storage_ is deque<double>
```

Notice that when you write `Stack<std::vector, int>`, you're passing the `std::vector` template itself - not `std::vector<int>`. The `Stack` template then plugs in `T` to form the concrete type internally.

### Syntax

The syntax wraps the inner template's parameter list inside the outer template parameter:

```cpp
template <template <typename...> class Name>   // Using typename...
//        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//        template template parameter

template <template <typename> class Name>      // Exactly one type param
template <template <typename, typename> class Name>  // Exactly two
```

### `typename...` vs Fixed Params

The choice between `typename...` and a fixed-arity form matters because standard containers like `std::vector` have more than one template parameter (the second being the allocator). If you lock your parameter to exactly one type argument, `std::vector` won't match:

| Syntax | Matches |
| --- | --- |
| `template<typename...> class C` | Any template with type params (flexible) |
| `template<typename> class C` | Only templates with exactly 1 type param |
| `template<typename,typename> class C` | Only templates with exactly 2 type params |

In C++17+, `template<typename...>` can match templates with default arguments (like `std::vector<T, Alloc=...>`).

---

## Self-Assessment

### Q1: Write a class template that accepts a container template as a parameter: `template<template<typename...> class Container>`

The caller chooses which container to use - `std::vector`, `std::deque`, `std::list` - and the template instantiates it with `T`. Everything else about `DataBuffer` stays the same regardless of which container you pick:

```cpp
#include <iostream>
#include <vector>
#include <deque>
#include <list>
#include <string>

// Class parameterized on a container TEMPLATE (not a concrete type)
template <template <typename...> class Container, typename T>
class DataBuffer {
    Container<T> buffer_;
    std::string name_;

public:
    DataBuffer(std::string name) : name_(std::move(name)) {}

    void push(const T& val) { buffer_.push_back(val); }

    void print() const {
        std::cout << name_ << ": [";
        bool first = true;
        for (const auto& v : buffer_) {
            if (!first) std::cout << ", ";
            std::cout << v;
            first = false;
        }
        std::cout << "] (size=" << buffer_.size() << ")\n";
    }

    size_t size() const { return buffer_.size(); }
};

// The caller chooses WHICH container to use — without committing to a type
template <template <typename...> class Container>
void fill_container(DataBuffer<Container, int>& buf) {
    for (int i = 1; i <= 5; ++i)
        buf.push(i * 10);
}

int main() {
    // Same interface, different underlying containers
    DataBuffer<std::vector, int> vec_buf("vector");
    DataBuffer<std::deque, int>  deq_buf("deque");
    DataBuffer<std::list, int>   list_buf("list");

    fill_container(vec_buf);
    fill_container(deq_buf);
    fill_container(list_buf);

    vec_buf.print();   // vector: [10, 20, 30, 40, 50]
    deq_buf.print();   // deque:  [10, 20, 30, 40, 50]
    list_buf.print();  // list:   [10, 20, 30, 40, 50]

    // Different element type
    DataBuffer<std::vector, std::string> str_buf("strings");
    str_buf.push("hello");
    str_buf.push("world");
    str_buf.print();  // strings: [hello, world]

    return 0;
}
```

### Q2: Explain a case where template template parameters fail with standard library containers that have default allocators

The reason this trips people up is that `std::vector` is not `template<typename T>` - it's `template<typename T, typename Alloc>`. If your template template parameter expects exactly one type parameter, the match fails. The `typename...` variadic form solves this in C++11 and later, and C++17 relaxed the matching rules further:

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <string>

// === The Problem (pre-C++17) ===
// std::vector is really: template<typename T, typename Alloc = std::allocator<T>> class vector;
// It has TWO template parameters, not one!

// In C++11/14, this FAILS:
// template <template <typename> class Container, typename T>
//                     ^^^^^^^^ expects ONE param
// class Stack {
//     Container<T> data_;  // vector<T> needs vector<T, Alloc>!
// };
// Stack<std::vector, int> s;  // ERROR: vector has 2 params, not 1

// === Solution 1: Use typename... (C++17 friendly) ===
template <template <typename...> class Container, typename T>
class Stack17 {
    Container<T> data_;
public:
    void push(const T& v) { data_.push_back(v); }
    size_t size() const { return data_.size(); }
};

// In C++17, template template parameters with default args are matched relaxedly
// So template<typename> class Container can also match std::vector in C++17!

// === Solution 2: Use a type alias ===
template <typename T>
using Vec = std::vector<T>;  // Vec is a template with ONE param

template <template <typename> class Container, typename T>
class Stack_alias {
    Container<T> data_;
public:
    void push(const T& v) { data_.push_back(v); }
    size_t size() const { return data_.size(); }
};

// === Problem with non-type template params ===
// std::array<T, N> has a NON-TYPE parameter -> cannot match template<typename...>
// template<template<typename...> class C> -> does NOT match std::array

int main() {
    // Works with typename...
    Stack17<std::vector, int> s1;
    s1.push(42);

    // Works with alias
    Stack_alias<Vec, double> s2;
    s2.push(3.14);

    std::cout << "Stack17 size: " << s1.size() << "\n";    // 1
    std::cout << "Stack_alias size: " << s2.size() << "\n"; // 1

    std::cout << "\n=== Common pitfalls ===\n";
    std::cout << "1. Pre-C++17: template<typename> class doesn't match vector (has 2 params)\n";
    std::cout << "   Fix: use template<typename...> class or alias templates\n";
    std::cout << "2. std::array<T,N> has non-type param -> can't match typename...\n";
    std::cout << "3. std::map<K,V,Comp,Alloc> has 4 params but typename... matches in C++17\n";

    return 0;
}
```

### Q3: Rewrite a template template parameter example using type aliases to simplify the interface

Often the simplest fix for complex template template parameter syntax is to expose a cleaner alias to your users - or to skip template template parameters entirely and just take the fully-instantiated container type. The third approach shown here is worth knowing because it sidesteps the whole matching problem:

```cpp
#include <iostream>
#include <vector>
#include <deque>
#include <list>
#include <string>
#include <algorithm>

// === BEFORE: Template template parameter (verbose) ===
template <template <typename...> class Container, typename T>
class Collection_TTP {
    Container<T> data_;
public:
    void add(T val) { data_.push_back(std::move(val)); }
    size_t size() const { return data_.size(); }
    void print(const std::string& label) const {
        std::cout << label << ": ";
        for (const auto& v : data_) std::cout << v << " ";
        std::cout << "\n";
    }
};
// Usage: Collection_TTP<std::vector, int> c;  // verbose

// === AFTER: Type alias simplifies the interface ===
template <typename T>
using VecCollection = Collection_TTP<std::vector, T>;

template <typename T>
using DequeCollection = Collection_TTP<std::deque, T>;

template <typename T>
using ListCollection = Collection_TTP<std::list, T>;
// Usage: VecCollection<int> c;  // much cleaner!

// === Alternative: Pass the fully-instantiated container type ===
template <typename Container>
class Collection_Simple {
    Container data_;
    using T = typename Container::value_type;
public:
    void add(T val) { data_.push_back(std::move(val)); }
    size_t size() const { return data_.size(); }
    void print(const std::string& label) const {
        std::cout << label << ": ";
        for (const auto& v : data_) std::cout << v << " ";
        std::cout << "\n";
    }
};
// Usage: Collection_Simple<std::vector<int>> c;
// Simpler to write and understand — no template template params!

int main() {
    std::cout << "=== Template template parameter (verbose) ===\n";
    Collection_TTP<std::vector, int> ttp;
    ttp.add(3); ttp.add(1); ttp.add(2);
    ttp.print("TTP vector");

    std::cout << "\n=== Type alias (clean) ===\n";
    VecCollection<int> vc;
    vc.add(30); vc.add(10); vc.add(20);
    vc.print("VecCollection");

    DequeCollection<std::string> dc;
    dc.add("hello"); dc.add("world");
    dc.print("DequeCollection");

    std::cout << "\n=== Simple container type param (simplest) ===\n";
    Collection_Simple<std::vector<int>> simple;
    simple.add(100); simple.add(200);
    simple.print("Simple");

    std::cout << "\nRecommendation:\n";
    std::cout << "  1. If you need to rebind container to different types -> template template\n";
    std::cout << "  2. If container type is fixed -> pass full type (simplest)\n";
    std::cout << "  3. Use type aliases to hide template template complexity from users\n";

    return 0;
}
```

---

## Notes

- Template template parameters let you parameterize on a **template** rather than a concrete type.
- Syntax: `template <template <typename...> class Container>` - note the `class` keyword (or `typename` in C++17+).
- Use `typename...` to match containers with default template arguments (allocators, comparators).
- C++17 relaxed matching rules: `template<typename>` can now match `vector<T, Alloc>` if `Alloc` has a default.
- Type aliases simplify the user interface: `using VecStack = Stack<std::vector, int>;`
- Alternative: just take the full container type (`template<typename C>`) and extract `C::value_type`.
