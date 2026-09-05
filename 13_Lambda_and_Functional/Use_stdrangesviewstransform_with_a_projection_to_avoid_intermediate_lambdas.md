# Use std::ranges::views::transform with a projection to avoid intermediate lambdas

**Category:** Lambda and Functional  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/transform_view>  

---

## Topic Overview

In C++20, **member pointers** - both data members and member functions - can be used directly as callables with `views::transform`, `std::invoke`, and algorithm projections. This eliminates the need for trivial one-liner lambdas like `[](const auto& p) { return p.name; }` that exist for no other purpose than to extract a single field.

The key insight is that `std::invoke(&Person::name, p)` returns `p.name`. Because ranges algorithms and views internally use `std::invoke`, they can accept a member pointer anywhere they accept a callable.

```cpp
struct Person { std::string name; int age; };
std::vector<Person> people = {{"Alice", 30}, {"Bob", 25}};

// Lambda (verbose):
for (auto& n : people | views::transform([](auto& p) { return p.name; }))
    std::cout << n;

// Member pointer (clean!):
for (auto& n : people | views::transform(&Person::name))
    std::cout << n;
```

Both lines do the same thing. The member pointer version is shorter and signals the intent more directly - you're projecting onto `name`, not writing an anonymous function that happens to access `name`.

---

## Self-Assessment

### Q1: Project a range of structs to a member using `views::transform(&T::member)`

Notice how the same `&Product::name` member pointer works with both a view pipeline and the projection parameter of `std::ranges::max` and `std::ranges::sort`. You write the projection once and it works everywhere.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <ranges>
#include <algorithm>
#include <numeric>

namespace rv = std::ranges::views;

struct Product {
    std::string name;
    double price;
    int quantity;
};

int main() {
    std::vector<Product> products = {
        {"Laptop", 999.99, 5},
        {"Mouse", 29.99, 50},
        {"Keyboard", 79.99, 30},
        {"Monitor", 449.99, 10}
    };

    // Project to names - NO lambda needed!
    std::cout << "Products:\n";
    for (const auto& name : products | rv::transform(&Product::name)) {
        std::cout << "  " << name << "\n";
    }

    // Project to prices:
    std::cout << "\nPrices:\n";
    for (double p : products | rv::transform(&Product::price)) {
        std::cout << "  $" << p << "\n";
    }

    // Use projection in algorithm:
    auto most_expensive = std::ranges::max(products, {}, &Product::price);
    std::cout << "\nMost expensive: " << most_expensive.name
              << " ($" << most_expensive.price << ")\n";

    // Sort by quantity (ascending):
    std::ranges::sort(products, {}, &Product::quantity);
    std::cout << "\nSorted by quantity:\n";
    for (const auto& p : products) {
        std::cout << "  " << p.name << ": " << p.quantity << "\n";
    }
}
// Expected output:
//   Products:
//     Laptop
//     Mouse
//     Keyboard
//     Monitor
//
//   Prices:
//     $999.99
//     $29.99
//     $79.99
//     $449.99
//
//   Most expensive: Laptop ($999.99)
//
//   Sorted by quantity:
//     Laptop: 5
//     Monitor: 10
//     Keyboard: 30
//     Mouse: 50
```

---

### Q2: Show that member function pointers work as projections too

This is the part that surprises people. You'd expect member data pointers to work as projections, since they just access a field. But member function pointers work the same way - `std::invoke(&Person::full_name, p)` calls `p.full_name()`, so you can pass `&Person::full_name` anywhere you'd pass a lambda that calls that method.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <ranges>
#include <algorithm>

namespace rv = std::ranges::views;

struct Person {
    std::string first;
    std::string last;

    std::string full_name() const { return first + " " + last; }
    size_t name_length() const { return first.size() + last.size(); }
};

int main() {
    std::vector<Person> people = {
        {"Alice", "Smith"},
        {"Bob", "Johnson"},
        {"Charlie", "Brown"}
    };

    // Member FUNCTION pointer as projection - works with std::invoke!
    std::cout << "Full names:\n";
    for (const auto& name : people | rv::transform(&Person::full_name)) {
        std::cout << "  " << name << "\n";
    }

    // Member data pointer:
    std::cout << "\nFirst names:\n";
    for (const auto& f : people | rv::transform(&Person::first)) {
        std::cout << "  " << f << "\n";
    }

    // In algorithms with projection parameter:
    auto longest = std::ranges::max(people, {}, &Person::name_length);
    std::cout << "\nLongest name: " << longest.full_name() << "\n";

    // std::string::size works too:
    std::vector<std::string> words = {"short", "a", "medium-length", "hi"};
    std::ranges::sort(words, {}, &std::string::size);
    for (const auto& w : words) std::cout << w << " ";
    std::cout << "\n";
}
// Expected output:
//   Full names:
//     Alice Smith
//     Bob Johnson
//     Charlie Brown
//
//   First names:
//     Alice
//     Bob
//     Charlie
//
//   Longest name: Bob Johnson
//   a hi short medium-length
```

---

### Q3: Compose a filter and a projection in a single pipeline without intermediate containers

One of the biggest practical wins from projections is that they compose cleanly in pipelines. You can filter by one member, then project to another member, all without allocating intermediate vectors. The range is lazy - elements flow through each stage on demand.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <ranges>
#include <algorithm>

namespace rv = std::ranges::views;

struct Employee {
    std::string name;
    std::string dept;
    double salary;
};

int main() {
    std::vector<Employee> employees = {
        {"Alice", "Engineering", 95000},
        {"Bob", "Marketing", 72000},
        {"Charlie", "Engineering", 110000},
        {"Diana", "Marketing", 68000},
        {"Eve", "Engineering", 88000},
        {"Frank", "Sales", 75000}
    };

    // Pipeline: filter by dept -> project to name -> NO intermediate vector!
    std::cout << "Engineering team:\n";
    auto eng_names = employees
        | rv::filter([](const Employee& e) { return e.dept == "Engineering"; })
        | rv::transform(&Employee::name);

    for (const auto& name : eng_names) {
        std::cout << "  " << name << "\n";
    }

    // Filter + project to salary + compute sum
    double eng_total = 0;
    for (double s : employees
            | rv::filter([](const Employee& e) { return e.dept == "Engineering"; })
            | rv::transform(&Employee::salary)) {
        eng_total += s;
    }
    std::cout << "\nEngineering total salary: $" << eng_total << "\n";

    // Alternative: Use ranges algorithm with projection
    // No pipeline needed - projection IS the "transform"
    auto highest_paid = std::ranges::max(employees, {}, &Employee::salary);
    std::cout << "Highest paid: " << highest_paid.name
              << " ($" << highest_paid.salary << ")\n";

    // Complex pipeline: filter + transform + enumerate
    std::cout << "\nHigh earners (>$80k):\n";
    int idx = 1;
    for (const auto& name : employees
            | rv::filter([](const Employee& e) { return e.salary > 80000; })
            | rv::transform(&Employee::name)) {
        std::cout << "  " << idx++ << ". " << name << "\n";
    }
}
// Expected output:
//   Engineering team:
//     Alice
//     Charlie
//     Eve
//
//   Engineering total salary: $293000
//   Highest paid: Charlie ($110000)
//
//   High earners (>$80k):
//     1. Alice
//     2. Charlie
//     3. Eve
```

---

## Notes

- **Member pointer as projection** works because `std::invoke(&T::member, obj)` returns `obj.member` - both data members and member functions follow this rule.
- **Algorithm projections** (the second-to-last parameter in most ranges algorithms) accept the same callables as `views::transform`, so you can use member pointers there too.
- **No copy overhead:** `views::transform(&T::member)` gives references to members, not copies.
- **Combine projections:** You can chain transforms: `transform(&Employee::name) | transform(&string::size)`.
- **`std::identity`:** C++20's identity projection `{}` is the default - pass no projection to use elements as-is.
