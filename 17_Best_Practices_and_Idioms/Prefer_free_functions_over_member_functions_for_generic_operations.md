# Prefer free functions over member functions for generic operations

**Category:** Best Practices and Idioms  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-standalone>  

---

## Topic Overview

Free functions enable extension without modification, work with ADL, and can operate generically on types that share an interface but don't share a base class. The key insight is that a free function can be added to any type - including ones you don't own - while a member function requires modifying the class itself.

### Member vs Free Function Discovery

When you call a free function unqualified, the compiler casts a much wider net than it does for a member function call:

```cpp
obj.swap(other)    -> compiler looks in obj's class only
swap(obj, other)   -> compiler looks in:

                     1. Current namespace
                     2. Namespace of obj's type (ADL)
                     3. using declarations
```

That wider search is exactly what makes free functions so useful for generic code.

---

## Self-Assessment

### Q1: Explain how a non-member `swap` extends the interface without changing the class

Sometimes you are working with a third-party class you cannot modify. Free functions let you attach new behavior to it without touching the original source. Here `swap` is provided for a library type purely through a free function in the same translation unit.

```cpp
#include <algorithm>
#include <iostream>
#include <utility>

// Library class (we can't modify it)
class ThirdPartyWidget {
    int id_;
    double data_[100]; // expensive to copy
public:
    ThirdPartyWidget(int id) : id_(id), data_{} {}
    int id() const { return id_; }
    double* data() { return data_; }
    const double* data() const { return data_; }
};

// Extend with efficient swap WITHOUT changing ThirdPartyWidget
void swap(ThirdPartyWidget& a, ThirdPartyWidget& b) noexcept {
    // Use the class's public interface to swap
    ThirdPartyWidget tmp(0);
    std::memcpy(tmp.data(), a.data(), sizeof(double) * 100);
    std::memcpy(a.data(), b.data(), sizeof(double) * 100);
    std::memcpy(b.data(), tmp.data(), sizeof(double) * 100);
    std::cout << "Custom swap called for widgets " << a.id() << " and " << b.id() << '\n';
}

// ADL finds our swap:
template<typename T>
void sort_two(T& a, T& b) {
    using std::swap;  // fallback
    if (b.id() < a.id())
        swap(a, b);   // ADL finds our swap for ThirdPartyWidget
}

int main() {
    ThirdPartyWidget w1(2), w2(1);
    sort_two(w1, w2);
    std::cout << "After sort: " << w1.id() << ", " << w2.id() << '\n';
}
// Expected output:
// Custom swap called for widgets 2 and 1
// After sort: 1, 2
```

The `using std::swap;` line in the template is the standard pattern - it brings in the standard library fallback, and then the unqualified `swap(a, b)` call lets ADL pick a type-specific overload if one exists.

### Q2: Show how ADL finds the right free function for a type

This example has two types in two different namespaces, each with its own free functions. No `using` declarations are needed anywhere - ADL finds the right function purely from the argument type.

```cpp
#include <iostream>
#include <string>

namespace physics {
    struct Vector3 {
        double x, y, z;
    };

    // Free function in the same namespace as Vector3
    double magnitude(const Vector3& v) {
        return std::sqrt(v.x * v.x + v.y * v.y + v.z * v.z);
    }

    std::ostream& operator<<(std::ostream& os, const Vector3& v) {
        return os << "(" << v.x << ", " << v.y << ", " << v.z << ")";
    }
}

namespace graphics {
    struct Color {
        float r, g, b;
    };

    float brightness(const Color& c) {
        return 0.299f * c.r + 0.587f * c.g + 0.114f * c.b;
    }

    std::ostream& operator<<(std::ostream& os, const Color& c) {
        return os << "rgb(" << c.r << ", " << c.g << ", " << c.b << ")";
    }
}

int main() {
    physics::Vector3 v{3, 4, 0};
    graphics::Color c{1.0f, 0.5f, 0.0f};

    // ADL finds the right function based on argument type's namespace
    std::cout << v << " magnitude: " << magnitude(v) << '\n';
    std::cout << c << " brightness: " << brightness(c) << '\n';
    // No 'using' needed! ADL automatically looks in physics:: and graphics::
}
// Expected output:
// (3, 4, 0) magnitude: 5
// rgb(1, 0.5, 0) brightness: 0.5925
```

The `magnitude` and `brightness` calls have no namespace prefix - they just work. If you had tried to write these as member functions and then call them generically, you would need a common base class or a template. Free functions with ADL give you the same flexibility for free.

### Q3: Compare `std::size()`, `std::begin()`, `std::end()` as free functions with `.size()`, `.begin()` as members

This is one of the clearest arguments for free functions in the standard library. Member functions are tied to the type - C arrays have no members at all. Free functions can be overloaded for anything, including built-in types.

```cpp
#include <array>
#include <iostream>
#include <iterator>
#include <vector>

// Free functions work with BOTH containers AND C-arrays:
template<typename Range>
void print_info(const Range& r) {
    std::cout << "Size: " << std::size(r) << ", First: ";
    if (std::size(r) > 0)
        std::cout << *std::begin(r);
    std::cout << '\n';
}

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    int arr[] = {10, 20, 30};
    std::array<int, 4> sarr = {100, 200, 300, 400};

    // Free functions: work with all three
    print_info(vec);   // std::size(vec)  -> 5
    print_info(arr);   // std::size(arr)  -> 3 (deduced from C-array)
    print_info(sarr);  // std::size(sarr) -> 4

    // Member functions: DON'T work with C-arrays
    // vec.size();  // OK
    // arr.size();  // ERROR: C-array has no .size() method!
}
// Expected output:
// Size: 5, First: 1
// Size: 3, First: 10
// Size: 4, First: 100
```

The `print_info` template works uniformly for all three types because it uses the free function versions. If it used `.size()` and `.begin()`, it would fail to compile for raw arrays - even though arrays are perfectly valid sequences.

**Comparison:**

| Function | Member `.size()` | Free `std::size()` |
| --- | --- | --- |
| `vector<T>` | Yes | Yes |
| `array<T,N>` | Yes | Yes |
| `T arr[N]` (C-array) | No | Yes |
| `string_view` | Yes | Yes |
| `span<T>` | Yes | Yes |
| Generic code | Requires `.size()` method | Works with anything |

---

## Notes

- Use `using std::swap;` before unqualified `swap(a, b)` to enable ADL + fallback.
- `std::size()`, `std::begin()`, `std::end()`, `std::data()`, `std::empty()` were added in C++17.
- Ranges (C++20) use `std::ranges::begin()` etc. which handle even more edge cases.
- Free functions + ADL is the C++ way to achieve something like Rust traits or Haskell type classes.
