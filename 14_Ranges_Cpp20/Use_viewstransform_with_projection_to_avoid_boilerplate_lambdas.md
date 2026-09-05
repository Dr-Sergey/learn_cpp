# Use views::transform with projection to avoid boilerplate lambdas

**Category:** Ranges Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges>  

---

## Topic Overview

C++20 ranges algorithms accept an optional **projection** parameter - a callable applied to each element before the algorithm's operation. This eliminates many trivial lambdas that previously existed only to extract a field from a struct before a comparison.

### Without vs With Projections

Here's the same sort written both ways. The projection form is not just shorter - it communicates intent more directly, because the "sort by name" logic is expressed as a data member pointer rather than buried inside a comparison lambda.

```cpp
// WITHOUT projection (C++17)
std::sort(vec.begin(), vec.end(),
    [](const Person& a, const Person& b) { return a.name < b.name; });

// WITH projection (C++20)
std::ranges::sort(vec, {}, &Person::name);  // {} = default comparator
```

### What Can Be a Projection

Projections are flexible - they accept anything that `std::invoke` can call on an element:

| Projection type | Example | What it does |
| --- | --- | --- |
| Pointer to member | `&Person::name` | Extracts `name` field |
| Pointer to member function | `&std::string::size` | Calls `.size()` |
| Lambda | `[](const Person& p) { return p.age * 2; }` | Custom transformation |
| `std::identity{}` | (default) | No-op, passes element unchanged |
| Function pointer | `std::abs` | Applies `abs` before comparison |

### How Projections Work Internally

The mechanics are simple: the algorithm calls `std::invoke(proj, element)` before passing the result to the comparator or predicate. You never see this call yourself, but understanding it helps explain why member function pointers and data member pointers both work - `std::invoke` handles both cases uniformly.

```cpp
For ranges::sort(vec, comp, proj):
  comp(std::invoke(proj, a), std::invoke(proj, b))
  -> comp(a.name, b.name)  // when proj = &Person::name
```

### Projection vs views::transform

These two features serve different purposes. Projections are local to a single algorithm call; `views::transform` creates a reusable view in a pipeline.

| Feature | Projection | views::transform |
| --- | --- | --- |
| Scope | Inside one algorithm call | Creates a new view |
| Composable | No - per-algorithm only | Yes - pipes into other views |
| Zero overhead | Yes - inlined by compiler | Yes - lazy, no copies |
| Mutates original | Depends on algorithm | No - transforms on the fly |

---

## Self-Assessment

### Q1: Use a member pointer as a projection in `std::ranges::sort(vec, {}, &Person::name)`

This example shows sorting a vector of structs by different fields using projections. Notice that `{}` as the comparator argument means "use the default", which is `std::ranges::less{}` - i.e., ascending order.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Person {
    std::string name;
    int age;
    double salary;
};

int main() {
    std::vector<Person> people = {
        {"Charlie", 30, 75000},
        {"Alice",   25, 82000},
        {"Bob",     35, 68000},
        {"Diana",   28, 91000},
    };

    // Sort by name (alphabetical)
    std::ranges::sort(people, {}, &Person::name);
    std::cout << "By name: ";
    for (const auto& p : people) std::cout << p.name << ' ';
    std::cout << '\n';

    // Sort by age (ascending)
    std::ranges::sort(people, {}, &Person::age);
    std::cout << "By age:  ";
    for (const auto& p : people) std::cout << p.name << '(' << p.age << ") ";
    std::cout << '\n';

    // Sort by salary (descending)
    std::ranges::sort(people, std::greater{}, &Person::salary);
    std::cout << "By salary (desc): ";
    for (const auto& p : people)
        std::cout << p.name << '(' << p.salary << ") ";
    std::cout << '\n';

    // Find person with max salary
    auto richest = std::ranges::max(people, {}, &Person::salary);
    std::cout << "Highest paid: " << richest.name << '\n';
}
// Expected output:
// By name: Alice Bob Charlie Diana
// By age:  Alice Diana Charlie Bob
// By salary (desc): Diana Alice Charlie Bob
// Highest paid: Diana
```

`ranges::max` with a projection returns the full `Person` object that has the maximum salary - not just the salary value itself. This is a common pattern: you project a key for comparison, but the algorithm hands back the original element, not the projected key.

### Q2: Explain how projections are applied inside range algorithms and their zero-overhead nature

The pseudo-code below shows the internal structure of a projection-aware sort. The projection is called via `std::invoke` on each element before the comparator sees it, but the algorithm still moves/swaps the original elements.

```cpp
// Simplified pseudo-code for ranges::sort with projection:
template<typename R, typename Comp, typename Proj>
void sort(R&& range, Comp comp, Proj proj) {
    // ...
    // When comparing elements a and b:
    if (std::invoke(comp, std::invoke(proj, a), std::invoke(proj, b))) {
        swap(a, b);
    }
    // ...
}
```

The reason projections have zero overhead is that `std::invoke(&Employee::id, emp)` compiles to exactly `emp.id` - a plain member access with no indirection. The compiler inlines the whole thing.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Employee {
    std::string name;
    int id;
};

int main() {
    std::vector<Employee> team = {
        {"Alice", 3}, {"Bob", 1}, {"Charlie", 2}
    };

    // WITH projection (recommended)
    std::ranges::sort(team, {}, &Employee::id);

    // Equivalent WITHOUT projection (more verbose)
    // std::ranges::sort(team, [](const Employee& a, const Employee& b) {
    //     return a.id < b.id;
    // });

    // Both generate IDENTICAL assembly (zero overhead)
    // The projection &Employee::id is inlined by the compiler to a direct
    // member access, with no function call overhead.

    for (const auto& e : team)
        std::cout << e.name << '(' << e.id << ") ";
    std::cout << '\n';

    // Projection with member function pointer
    std::vector<std::string> words = {"banana", "fig", "apple", "elderberry"};
    std::ranges::sort(words, {}, &std::string::size);  // sort by length
    std::cout << "By length: ";
    for (const auto& w : words) std::cout << w << ' ';
    std::cout << '\n';
}
// Expected output:
// Alice(1) Bob(2) Charlie(3)
// By length: fig apple banana elderberry
```

Projections are also constrained by concepts at compile time. If you pass something that can't be invoked on an element, you get a clear error at the point of the call rather than a cryptic message buried inside the algorithm's implementation.

### Q3: Show that projections can be composed with `views::transform`

Projections live inside algorithm calls; `views::transform` lives in pipelines. They play complementary roles, and you can use both together when the situation calls for it.

```cpp
#include <algorithm>
#include <cmath>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Product {
    std::string name;
    double price;
    int quantity;
};

int main() {
    std::vector<Product> inventory = {
        {"Widget",  9.99,  50},
        {"Gadget", 24.99, 100},
        {"Gizmo",  14.99,  30},
        {"Doohickey", 4.99, 200},
    };

    // views::transform extracts a field (like a projection for views)
    auto names = inventory | std::views::transform(&Product::name);
    std::cout << "Products: ";
    for (const auto& n : names) std::cout << n << ' ';
    std::cout << '\n';

    // Compute total value: transform to price*quantity, then sum
    auto values = inventory | std::views::transform(
        [](const Product& p) { return p.price * p.quantity; });

    double total = std::ranges::fold_left(values, 0.0, std::plus{});
    std::cout << "Total inventory value: $" << std::round(total * 100) / 100 << '\n';

    // Use projection in algorithm ON a transformed view
    // Sort products by total value descending
    std::ranges::sort(inventory, std::greater{},
        [](const Product& p) { return p.price * p.quantity; });

    std::cout << "By total value (desc):\n";
    for (const auto& p : inventory)
        std::cout << "  " << p.name << ": $" << (p.price * p.quantity) << '\n';

    // Pipeline: filter expensive items, project names
    auto expensive_names = inventory
        | std::views::filter([](const Product& p) { return p.price > 10.0; })
        | std::views::transform(&Product::name);

    std::cout << "Expensive: ";
    for (const auto& n : expensive_names) std::cout << n << ' ';
    std::cout << '\n';
}
// Expected output:
// Products: Widget Gadget Gizmo Doohickey
// Total inventory value: $3946
// By total value (desc):
//   Gadget: $2499
//   Doohickey: $998
//   Widget: $499.5
//   Gizmo: $449.7
// Expensive: Gadget Gizmo
```

The general rule is: use a projection when you need the algorithm to compare or inspect by a key, and use `views::transform` when you want to build a pipeline that produces projected values as a new range. For complex computed keys like `price * quantity`, a lambda works in both positions. `views::transform(&Product::name)` is particularly clean - it takes a member pointer directly, exactly like a projection would, but produces a composable view of just the names.

---

## Notes

- Projections are available on **all** ranges algorithms: `sort`, `find`, `count`, `min`, `max`, `copy_if`, etc.
- The default projection is `std::identity{}` - doing nothing.
- `&T::member` works as projection because `std::invoke(ptr_to_member, object)` is defined to extract the member.
- For chaining multiple projections, use `views::transform` instead - projections don't compose directly.
