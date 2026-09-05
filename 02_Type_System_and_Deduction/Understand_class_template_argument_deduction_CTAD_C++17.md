# Understand class template argument deduction (CTAD, C++17)

**Category:** Type System and Deduction  
**Standard:** C++17  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#class-template-argument-deduction>  

---

## Topic Overview

C++17 allows class templates to deduce their template arguments from constructor arguments - no need to specify them explicitly. This is the same convenience you already had with function templates, now extended to class templates.

### Basic CTAD

Before C++17, you always had to spell out the template arguments. With C++17, the compiler figures them out from what you pass to the constructor. Here's what that looks like in practice:

```cpp
#include <vector>
#include <mutex>
#include <tuple>

// Before C++17: must specify template arguments
std::vector<int> v1 = {1, 2, 3};
std::pair<int, double> p1 = {42, 3.14};
std::lock_guard<std::mutex> lg1(mtx);

// C++17: CTAD deduces them automatically
std::vector v2 = {1, 2, 3};       // deduces vector<int>
std::pair p2 = {42, 3.14};         // deduces pair<int, double>
std::lock_guard lg2(mtx);          // deduces lock_guard<std::mutex>
std::tuple t = {1, 3.14, "hello"}; // deduces tuple<int, double, const char*>
```

### How CTAD Works

The compiler doesn't magically guess - it follows a specific procedure:

1. Compiler looks at all constructors of the class template.
2. Synthesizes a function template for each constructor.
3. Performs template argument deduction on those function templates.
4. User-defined **deduction guides** can override or supplement this.

This is worth understanding because deduction guides are your escape hatch when the automatic behavior isn't quite right.

### Deduction Guides

Sometimes the compiler can't figure out the template argument from the constructor parameters alone - for example, when you pass iterators and the element type `T` is hidden inside `iterator_traits`. That's where explicit deduction guides come in. You write one to tell the compiler the mapping from constructor argument types to the final template instantiation:

```cpp
template<typename T>
struct MyContainer {
    MyContainer(T value) {}                    // from value
    MyContainer(T* begin, T* end) {}           // from iterators
};

// Implicit deduction guides (from constructors):
// MyContainer(T) -> MyContainer<T>
// MyContainer(T*, T*) -> MyContainer<T>

// Explicit deduction guide:
template<typename Iter>
MyContainer(Iter, Iter) -> MyContainer<typename std::iterator_traits<Iter>::value_type>;
```

The explicit guide says: "when you call the constructor with two iterators, deduce `T` from the iterator's value type, not from the iterator type itself."

### Standard Library CTAD

The standard library types come with the right deduction guides already. You'll reach for these regularly:

```cpp
std::vector v{1, 2, 3};           // vector<int>
std::array a{1, 2, 3};            // array<int, 3>
std::optional o{42};              // optional<int>
std::unique_lock ul(mtx);         // unique_lock<std::mutex>
```

Notice `std::array` deduces both the element type and the size - that's particularly convenient.

---

## Self-Assessment

### Q1: Explain how `std::vector v{1,2,3};` deduces the element type

The interesting part here is the mechanism, not just the result. Watch the comment trail carefully - it traces how the compiler gets from `{1, 2, 3}` to `vector<int>`:

```cpp
#include <iostream>
#include <vector>
#include <array>
#include <tuple>
#include <optional>
#include <type_traits>

int main() {
    // CTAD in action — compiler deduces template arguments
    std::vector v1{1, 2, 3};              // vector<int>
    std::vector v2{1.0, 2.0, 3.0};        // vector<double>
    std::vector v3{"hello", "world"};     // vector<const char*>

    // How it works for vector:
    // std::vector has constructor: vector(initializer_list<T>)
    // Compiler synthesizes: template<typename T> vector(initializer_list<T>) -> vector<T>
    // {1, 2, 3} deduces initializer_list<int> -> T = int -> vector<int>

    static_assert(std::is_same_v<decltype(v1), std::vector<int>>);
    static_assert(std::is_same_v<decltype(v2), std::vector<double>>);

    // More CTAD examples:
    std::array a{1, 2, 3, 4};            // array<int, 4> — size AND type deduced!
    std::pair p{42, 3.14};               // pair<int, double>
    std::tuple t{1, "hi", 3.14};         // tuple<int, const char*, double>
    std::optional o{std::string("test")}; // optional<string>

    std::cout << "v1 size: " << v1.size() << "\n";     // 3
    std::cout << "array size: " << a.size() << "\n";    // 4
    std::cout << "pair: " << p.first << ", " << p.second << "\n"; // 42, 3.14

    // Gotcha: string literals
    std::vector sv{"hello", "world"};   // vector<const char*>, NOT vector<string>!
    // Fix: use string literals
    using namespace std::string_literals;
    std::vector sv2{"hello"s, "world"s}; // vector<string>
}
```

The string literal gotcha at the end is the most common CTAD surprise you'll hit. A string literal is `const char*`, not `std::string`, and CTAD dutifully deduces what you give it.

### Q2: Write a deduction guide for a custom class template taking iterator pairs

Without the deduction guide, `MyRange(iter, iter)` cannot deduce `T` because `T` doesn't appear directly in the iterator's type - it's buried inside `iterator_traits`. The guide bridges that gap:

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <iterator>
#include <type_traits>

template<typename T>
class MyRange {
    std::vector<T> data_;
public:
    // Constructor from value
    MyRange(T value) : data_{value} {}

    // Constructor from iterator pair
    template<typename Iter>
    MyRange(Iter first, Iter last) : data_(first, last) {}

    void print() const {
        for (const auto& x : data_) std::cout << x << " ";
        std::cout << "\n";
    }

    size_t size() const { return data_.size(); }
};

// Without a deduction guide, MyRange(iter, iter) can't deduce T:
// The constructor is template<typename Iter> MyRange(Iter, Iter)
// CTAD would try to deduce T from Iter, but T isn't directly related to Iter

// Deduction guide: tell the compiler how to get T from iterators
template<typename Iter>
MyRange(Iter, Iter) -> MyRange<typename std::iterator_traits<Iter>::value_type>;

// Another guide: from initializer_list
template<typename T>
MyRange(std::initializer_list<T>) -> MyRange<T>;

int main() {
    // Using the value constructor — CTAD works automatically
    MyRange r1(42);           // MyRange<int>
    r1.print();               // 42

    // Using the iterator constructor — needs our deduction guide!
    std::vector<double> source{1.1, 2.2, 3.3};
    MyRange r2(source.begin(), source.end());  // MyRange<double>
    r2.print();               // 1.1 2.2 3.3

    // Works with different container iterators
    std::list<std::string> names{"Alice", "Bob"};
    MyRange r3(names.begin(), names.end());  // MyRange<string>
    r3.print();               // Alice Bob

    // Works with raw pointers
    int arr[] = {10, 20, 30};
    MyRange r4(arr, arr + 3);  // MyRange<int>
    r4.print();                // 10 20 30
}
```

The key lesson: raw pointers also satisfy `iterator_traits`, so the same guide covers both container iterators and old-school pointer pairs.

### Q3: Show where CTAD produces unexpected deduction and how a deduction guide fixes it

This is where CTAD's "do what I mean" promise breaks down. The problem is that CTAD does what the constructor signature says, not what you intend:

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <utility>

// Problem: StringWrapper should hold std::string, but CTAD gets confused
template<typename T>
class StringWrapper {
    T data_;
public:
    StringWrapper(T data) : data_(std::move(data)) {}

    const T& get() const { return data_; }
};

// WITHOUT deduction guide:
// StringWrapper sw("hello"); -> StringWrapper<const char*>
// We wanted StringWrapper<std::string>!

// FIX: deduction guide that converts const char* -> std::string
StringWrapper(const char*) -> StringWrapper<std::string>;

// Another example: preventing decay
template<typename T>
class Holder {
    T value_;
public:
    Holder(const T& v) : value_(v) {}
};

// Without guide: Holder h("text") -> Holder<const char*> (array decays to pointer)
// With guide:
Holder(const char*) -> Holder<std::string>;

// Another common issue: aggregate CTAD (C++20)
template<typename T>
struct Pair {
    T first;
    T second;
};
// C++20 allows CTAD for aggregates:
// Pair p{1, 2}; -> Pair<int>
// But Pair p{1, 2.0}; -> ERROR: T can't be both int and double
// Fix with guide:
template<typename T, typename U>
    requires std::is_convertible_v<U, T>
Pair(T, U) -> Pair<std::common_type_t<T, U>>;

int main() {
    // Fixed: const char* -> std::string
    StringWrapper sw("hello");
    static_assert(std::is_same_v<decltype(sw), StringWrapper<std::string>>);
    std::cout << sw.get() << "\n";  // hello

    // Fixed: array decay
    Holder h("world");
    static_assert(std::is_same_v<decltype(h), Holder<std::string>>);
    std::cout << h.get() << "\n";   // world

    // Still works with explicit types:
    StringWrapper<int> sw2(42);  // T = int, explicit
    std::cout << sw2.get() << "\n"; // 42
}
```

The deduction guide acts as an interception layer: it catches the `const char*` call and redirects the deduction toward `std::string` before the class template is ever instantiated.

---

## Notes

- CTAD only works for constructors - you can't use it for member function templates.
- In C++17, `std::array` required a deduction guide for `std::array a{1,2,3}` to work - it deduces both the type AND size.
- C++20 added CTAD for aggregate types and alias templates.
- Deduction guides marked `explicit` only apply to direct initialization, not copy initialization.
- Avoid CTAD with `std::vector<bool>` - `std::vector v{true, false}` deduces `vector<bool>`, which is a special (non-standard) container.
- When CTAD is ambiguous, provide explicit deduction guides or specify template arguments manually.
