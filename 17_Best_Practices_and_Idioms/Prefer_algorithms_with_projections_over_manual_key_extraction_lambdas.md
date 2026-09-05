# Prefer algorithms with projections over manual key extraction lambdas

**Category:** Best Practices and Idioms  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges/sort>  

---

## Topic Overview

C++20 ranges algorithms accept a **projection** parameter - a callable that extracts the key to compare or process. This eliminates the need for verbose comparator lambdas where you used to spell out both arguments just to reach a single field. The result is code that reads almost like a sentence.

### Before vs After

Look at how much noise disappears when you switch from a hand-written comparator to a projection:

```cpp
// C++17: manual comparator lambda
std::sort(v.begin(), v.end(), [](const auto& a, const auto& b) {
    return a.name < b.name;
});

// C++20: projection
std::ranges::sort(v, {}, &Employee::name);
//                     ^^  ^^^^^^^^^^^^^^^
//                     default comparator  projection
```

The `{}` means "use the default comparator" (`std::less`), and `&Employee::name` is the projection - the algorithm will extract `name` from each element before comparing.

---

## Self-Assessment

### Q1: Replace `std::sort` with lambda with `std::ranges::sort` with projection

This example shows the same sort expressed both ways. Pay attention to how the C++17 version has to name both parameters and manually extract the field from each one, while the C++20 version just points at the field.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Employee {
    std::string name;
    int salary;
    int age;
};

int main() {
    std::vector<Employee> team = {
        {"Alice",   80000, 30},
        {"Charlie", 60000, 25},
        {"Bob",     90000, 35},
        {"Diana",   75000, 28},
    };

    // OLD WAY (C++17): verbose lambda comparator
    auto team_old = team;
    std::sort(team_old.begin(), team_old.end(),
        [](const Employee& a, const Employee& b) {
            return a.name < b.name;  // must name types, extract field twice
        });

    // NEW WAY (C++20): projection
    auto team_new = team;
    std::ranges::sort(team_new, {}, &Employee::name);  // one expression!

    std::cout << "Sorted by name:\n";
    for (const auto& e : team_new)
        std::cout << "  " << e.name << '\n';

    // Sort by salary (descending)
    std::ranges::sort(team, std::greater{}, &Employee::salary);
    std::cout << "Sorted by salary (desc):\n";
    for (const auto& e : team)
        std::cout << "  " << e.name << " $" << e.salary << '\n';
}
// Expected output:
// Sorted by name:
//   Alice
//   Bob
//   Charlie
//   Diana
// Sorted by salary (desc):
//   Bob $90000
//   Alice $80000
//   Diana $75000
//   Charlie $60000
```

Combining `std::greater{}` with a projection is particularly clean - it reads as "sort by salary, greatest first" without any comparator boilerplate.

### Q2: Show that projections reduce boilerplate and are more composable

Projections are not limited to `sort`. Every `std::ranges::` algorithm supports them - `min`, `max`, `count_if`, `stable_partition`, and so on. Once you learn the pattern, it applies everywhere.

```cpp
#include <algorithm>
#include <iostream>
#include <numeric>
#include <ranges>
#include <string>
#include <vector>

struct Product {
    std::string name;
    double price;
    int stock;
};

int main() {
    std::vector<Product> products = {
        {"Widget",  9.99,  100},
        {"Gadget", 24.99,   50},
        {"Gizmo",  14.99,  200},
        {"Doohickey", 4.99, 75},
    };

    // Find min/max by price - projection
    auto cheapest = std::ranges::min(products, {}, &Product::price);
    auto priciest = std::ranges::max(products, {}, &Product::price);
    std::cout << "Cheapest: " << cheapest.name << " $" << cheapest.price << '\n';
    std::cout << "Priciest: " << priciest.name << " $" << priciest.price << '\n';

    // Count products with stock > 60
    auto count = std::ranges::count_if(products,
        [](int s) { return s > 60; },  // predicate on the PROJECTED value
        &Product::stock);               // projection: extract stock
    std::cout << "In stock > 60: " << count << '\n';

    // Composable: projection works with any algorithm
    auto [lo, hi] = std::ranges::minmax(products, {}, &Product::stock);
    std::cout << "Stock range: " << lo.stock << " to " << hi.stock << '\n';

    // Projection with transform: extract names
    std::cout << "Products: ";
    for (const auto& name : products | std::views::transform(&Product::name))
        std::cout << name << " ";
    std::cout << '\n';
}
// Expected output:
// Cheapest: Doohickey $4.99
// Priciest: Gadget $24.99
// In stock > 60: 3
// Stock range: 50 to 200
// Products: Widget Gadget Gizmo Doohickey
```

Notice the `count_if` call: the predicate receives the already-projected value (an `int`), not the full `Product`. This is an important subtlety - the projection runs first, then the predicate sees the result.

### Q3: Combine a custom comparator with a projection

You can mix a custom comparator and a projection freely. The comparator receives the projected values, so you only have to think about the extracted key, not the whole struct.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Student {
    std::string name;
    double gpa;
    int year;
};

int main() {
    std::vector<Student> students = {
        {"Alice",   3.8, 3},
        {"Bob",     3.5, 2},
        {"Charlie", 3.9, 4},
        {"Diana",   3.5, 3},
    };

    // Custom comparator + projection:
    // Sort by GPA descending (comparator: greater), project: &Student::gpa
    std::ranges::sort(students, std::greater{}, &Student::gpa);

    std::cout << "By GPA (desc):\n";
    for (const auto& s : students)
        std::cout << "  " << s.name << " GPA=" << s.gpa << '\n';

    // Case-insensitive sort by name:
    // Projection: transforms name to lowercase for comparison
    std::ranges::sort(students, {},
        [](const Student& s) {
            std::string lower = s.name;
            std::ranges::transform(lower, lower.begin(), ::tolower);
            return lower;
        });

    std::cout << "By name (case-insensitive):\n";
    for (const auto& s : students)
        std::cout << "  " << s.name << '\n';

    // Stable partition: students with GPA >= 3.7, projected
    std::ranges::stable_partition(students,
        [](double gpa) { return gpa >= 3.7; },  // predicate on projected value
        &Student::gpa);                           // projection

    std::cout << "High GPA first:\n";
    for (const auto& s : students)
        std::cout << "  " << s.name << " " << s.gpa << '\n';
}
// Expected output:
// By GPA (desc):
//   Charlie GPA=3.9
//   Alice GPA=3.8
//   Bob GPA=3.5
//   Diana GPA=3.5
// By name (case-insensitive):
//   Alice
//   Bob
//   Charlie
//   Diana
// High GPA first:
//   Alice 3.8
//   Charlie 3.9
//   Bob 3.5
//   Diana 3.5
```

The lambda projection in the case-insensitive sort is a good example of when you go beyond a simple member pointer - you can project to anything the comparator should see, including a computed value.

---

## Notes

- The projection parameter is always the **last** parameter in ranges algorithms.
- `{}` as comparator means "use the default" (`std::less{}` or `std::ranges::less{}`).
- Member pointers (`&T::member`) are valid projections - `std::invoke` calls them.
- Projections compose with `views::transform` for pipelines.
- All `std::ranges::` algorithms support projections; classic `std::` algorithms don't.
