# Implement SFINAE-friendly and concept-constrained class interfaces

**Category:** OOP Design  
**Standard:** C++20  

---

## Topic Overview

One of the most common needs in generic C++ code is making certain methods available only when the template parameter satisfies some condition. You might want a `sum()` method that exists only when the element type supports addition, or a `print()` method that only compiles when the element type is printable. The way you express that constraint has evolved dramatically over the years, and it's worth understanding each era because you'll run into all of them in real codebases.

Here's the quick comparison so you can see where things stand:

| Technique | Standard | Error Messages | Readability |
| --- | --- | --- | --- |
| SFINAE (`enable_if`) | C++11 | Terrible | Poor |
| `if constexpr` | C++17 | Better | Good |
| **Concepts** | C++20 | **Excellent** | **Great** |

### Evolution

The same constraint - "this parameter must be an integer type" - looks very different depending on which era of C++ you're writing. Here's the before-and-after so the contrast is concrete:

```cpp
// C++11 SFINAE
template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
void process(T val);

// C++20 Concepts
template<std::integral T>
void process(T val);  // Same constraint, 10x more readable
```

The logic hasn't changed - only an integral type is accepted. But the C++20 version communicates that intent instantly, and if you pass the wrong type the compiler will say exactly which concept wasn't satisfied instead of dumping pages of template substitution noise.

---

## Self-Assessment

### Q1: Show SFINAE vs concepts for constraining class methods

The goal here is to understand both styles so you can read legacy code and write modern code. Notice how the SFINAE version buries the constraint in the return type, while the concepts version puts it right where you'd expect it - next to the function signature. This side-by-side comparison makes the readability difference impossible to ignore.

**Answer:**

```cpp
#include <type_traits>
#include <concepts>
#include <string>
#include <iostream>

// ===== C++11/14 SFINAE approach =====
template<typename T>
class ContainerOld {
    T data_;
public:
    explicit ContainerOld(T val) : data_(std::move(val)) {}

    // Only available for arithmetic types
    template<typename U = T>
    std::enable_if_t<std::is_arithmetic_v<U>, U>
    doubled() const { return data_ * 2; }

    // Only available for string-like types
    template<typename U = T>
    std::enable_if_t<std::is_convertible_v<U, std::string>, size_t>
    length() const { return std::string(data_).size(); }
};

// ===== C++20 Concepts approach =====
template<typename T>
class Container {
    T data_;
public:
    explicit Container(T val) : data_(std::move(val)) {}

    // Clean, readable constraints
    double doubled() const requires std::integral<T> || std::floating_point<T> {
        return data_ * 2;
    }

    size_t length() const requires std::convertible_to<T, std::string> {
        return std::string(data_).size();
    }

    // Unconstrained: available for all T
    const T& get() const { return data_; }
};

int main() {
    Container<int> ci(42);
    std::cout << ci.doubled() << "\n";  // 84
    // ci.length();  // COMPILE ERROR: int not convertible to string

    Container<std::string> cs("hello");
    std::cout << cs.length() << "\n";  // 5
    // cs.doubled();  // COMPILE ERROR: string not arithmetic
    return 0;
}
```

The `requires` clause at the end of a method declaration is the cleanest way to express a per-method constraint in C++20. If you try to call a method whose constraint isn't satisfied, the compiler's error message will say exactly that - no pages of template substitution failure, just "constraint not satisfied."

### Q2: Design a concept-constrained class with multiple constraint levels

Here's a more realistic design: a `DataSeries` class whose methods become available progressively depending on what the element type can do. Types that support addition get `sum()`. Types that support ordering get `max()`. Types that can be printed get `print()` and `operator<<`. This is sometimes called "graduated constraints," and it's a genuinely powerful way to design generic containers - each type gets exactly the interface it can support, nothing more, nothing less.

**Answer:**

```cpp
#include <concepts>
#include <iostream>
#include <vector>
#include <string>

// Custom concepts
template<typename T>
concept Printable = requires(std::ostream& os, const T& t) {
    { os << t } -> std::same_as<std::ostream&>;
};

template<typename T>
concept Hashable = requires(T t) {
    { std::hash<T>{}(t) } -> std::convertible_to<size_t>;
};

template<typename T>
concept Summable = requires(T a, T b) {
    { a + b } -> std::same_as<T>;
};

// Class with graduated constraints
template<typename T>
class DataSeries {
    std::vector<T> data_;
public:
    void add(T val) { data_.push_back(std::move(val)); }
    size_t size() const { return data_.size(); }

    // Only for summable types
    T sum() const requires Summable<T> {
        T result{};
        for (const auto& v : data_) result = result + v;
        return result;
    }

    // Only for ordered types
    T max() const requires std::totally_ordered<T> {
        T result = data_.front();
        for (const auto& v : data_)
            if (v > result) result = v;
        return result;
    }

    // Only for printable types
    void print() const requires Printable<T> {
        for (const auto& v : data_) std::cout << v << " ";
        std::cout << "\n";
    }

    // Constrained friend
    friend std::ostream& operator<<(std::ostream& os, const DataSeries& ds)
        requires Printable<T> {
        os << "[";
        for (size_t i = 0; i < ds.data_.size(); ++i) {
            if (i > 0) os << ", ";
            os << ds.data_[i];
        }
        return os << "]";
    }
};

int main() {
    DataSeries<int> ints;
    ints.add(3); ints.add(1); ints.add(4);
    std::cout << "Sum: " << ints.sum() << "\n";  // 8
    std::cout << "Max: " << ints.max() << "\n";  // 4
    ints.print();  // 3 1 4

    DataSeries<std::string> strs;
    strs.add("hello"); strs.add("world");
    std::cout << "Sum: " << strs.sum() << "\n";  // helloworld
    strs.print();
    return 0;
}
```

Notice that a `DataSeries<std::string>` has `sum()` (because string concatenation via `+` satisfies `Summable`) but also has `max()` (because strings have a total order). Each constraint is evaluated independently, so you get exactly the methods your type can support - nothing more, nothing less.

### Q3: Show SFINAE-friendly trait for detecting capabilities

Before concepts existed, detecting whether a type has a given member function required the "detection idiom" - a small `std::void_t`-based specialization trick. It's worth knowing because you'll encounter it in existing codebases. The C++20 concept equivalent is shown side-by-side so you can see how much simpler it became.

**Answer:**

```cpp
#include <type_traits>
#include <iostream>

// C++17 detection idiom
namespace detail {
    template<typename, typename = void>
    struct has_serialize : std::false_type {};

    template<typename T>
    struct has_serialize<T, std::void_t<
        decltype(std::declval<T>().serialize())
    >> : std::true_type {};

    template<typename, typename = void>
    struct has_size : std::false_type {};

    template<typename T>
    struct has_size<T, std::void_t<
        decltype(std::declval<T>().size())
    >> : std::true_type {};
}

template<typename T>
inline constexpr bool has_serialize_v = detail::has_serialize<T>::value;

template<typename T>
inline constexpr bool has_size_v = detail::has_size<T>::value;

// SFINAE-friendly process function
template<typename T>
void process(const T& obj) {
    if constexpr (has_serialize_v<T>) {
        std::cout << "Serialized: " << obj.serialize() << "\n";
    }
    if constexpr (has_size_v<T>) {
        std::cout << "Size: " << obj.size() << "\n";
    }
}

// C++20: same with concepts (much cleaner)
template<typename T>
concept HasSerialize = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

struct Widget {
    std::string serialize() const { return "widget-data"; }
};

struct Collection {
    size_t size() const { return 42; }
};

int main() {
    process(Widget{});      // Serialized: widget-data
    process(Collection{});  // Size: 42
    return 0;
}
```

The reason the `void_t` trick works is subtle, and this is the part that genuinely trips people up. Template specialization substitution normally fails silently - that's what "SFINAE" means: Substitution Failure Is Not An Error. When `decltype(std::declval<T>().serialize())` would produce an error (because `T` has no `serialize()`), the whole specialization is quietly discarded and the primary template - inheriting from `std::false_type` - wins instead. That's the detection idiom in a nutshell: arrange things so the compiler silently picks the "false" path when the operation doesn't exist. In C++20, `requires` expressions do exactly the same thing with far less ceremony.

---

## Notes

- C++20 concepts are the default choice for constraining templates - reserve SFINAE for legacy code or pre-C++20 targets.
- A `requires` clause on a method conditionally enables it based on the class template parameter, with no extra template boilerplate needed.
- Concepts give clear, actionable compiler errors - "constraint not satisfied" instead of pages of SFINAE substitution noise.
- The detection idiom (`void_t` plus specialization) is the C++17 pattern for detecting members when concepts aren't available.
- Use `if constexpr` for branching on type traits inside a function body when you want a single template to handle multiple cases without enabling or disabling whole methods.
- Subsumption is a useful property of concepts: a more constrained overload is automatically preferred over a less constrained one, so you can write specializations without explicit priority tricks.
