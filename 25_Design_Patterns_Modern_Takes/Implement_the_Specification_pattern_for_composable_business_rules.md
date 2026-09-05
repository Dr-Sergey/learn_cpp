# Implement the Specification pattern for composable business rules

**Category:** Design Patterns Modern Takes  
**Standard:** C++20  
**Reference:** <https://en.wikipedia.org/wiki/Specification_pattern>  

---

## Topic Overview

The **Specification pattern** encapsulates a business rule as a reusable, composable object. Using C++20 concepts and `std::ranges::filter`, specifications become first-class predicates that combine with `And`, `Or`, `Not` - replacing ad-hoc lambda chains with named, testable, composable business rules.

The key insight is that scattered `if (price < 50 && in_stock && category == "electronics")` checks buried deep in your code are hard to test, hard to name, and impossible to reuse. The Specification pattern gives each rule a name and makes rules snap together like puzzle pieces.

### Specification Composition

Here is a quick mental map of how specifications combine. Think of each node as a named rule object, and the combinators as the glue between them.

```cpp
AgeSpec(18)        &&  PremiumSpec()       -> AndSpec
  "is adult"           "is premium user"     "adult premium user"

AgeSpec(65)        ||  DisabilitySpec()    -> OrSpec
  "is senior"          "has disability"      "eligible for discount"

!BannedSpec()                              -> NotSpec
  "not banned"                                "account in good standing"
```

---

## Self-Assessment

### Q1: Define a Specification<T> concept with an is_satisfied_by(const T&) -> bool method

The first step is to use a C++20 concept to express what a Specification must look like. Any type that has an `is_satisfied_by` method returning `bool` qualifies. The concrete spec structs below each model one rule, and `static_assert` confirms they satisfy the concept at compile time - so you catch mistakes before they reach runtime.

**Answer:**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <concepts>

// Domain entity
struct Product {
    std::string name;
    double price;
    std::string category;
    bool in_stock;
};

// Specification concept (C++20)
template<typename Spec, typename T>
concept Specification = requires(const Spec& spec, const T& item) {
    { spec.is_satisfied_by(item) } -> std::same_as<bool>;
};

// Concrete specifications
struct InStockSpec {
    bool is_satisfied_by(const Product& p) const { return p.in_stock; }
};

struct PriceRangeSpec {
    double min_price, max_price;
    bool is_satisfied_by(const Product& p) const {
        return p.price >= min_price && p.price <= max_price;
    }
};

struct CategorySpec {
    std::string category;
    bool is_satisfied_by(const Product& p) const {
        return p.category == category;
    }
};

// Verify they satisfy the concept
static_assert(Specification<InStockSpec, Product>);
static_assert(Specification<PriceRangeSpec, Product>);
static_assert(Specification<CategorySpec, Product>);

int main() {
    std::vector<Product> products = {
        {"Laptop", 999.99, "electronics", true},
        {"Headphones", 49.99, "electronics", false},
        {"Book", 15.99, "books", true},
        {"Monitor", 399.00, "electronics", true},
    };

    PriceRangeSpec affordable{0, 100};
    InStockSpec available;

    for (const auto& p : products) {
        if (affordable.is_satisfied_by(p) && available.is_satisfied_by(p))
            std::cout << p.name << " ($" << p.price << ")\n";
    }
    // Book ($15.99)
}
```

Notice that the concept check at the top (`static_assert`) is your safety net. If you ever forget to implement `is_satisfied_by`, the error appears immediately at the `static_assert` line rather than deep inside some template.

### Q2: Compose specifications with And, Or, Not combinators returning new Specification objects

This is where the pattern really pays off. Instead of nesting `if` conditions, you build composite spec objects out of smaller ones. The operator overloads (`&&`, `||`, `!`) are defined once, and then composing rules reads almost like English: `InStockSpec{} && CheapSpec{50.0} && CategorySpec{"electronics"}`.

The reason this trips people up is that the `&&` here is not the `&&` between booleans. It's an overloaded operator that builds an `AndSpec` struct wrapping both sub-specs. The actual `bool` evaluation only happens when you call `is_satisfied_by`.

**Answer:**

```cpp
#include <iostream>
#include <string>
#include <vector>

struct Product {
    std::string name;
    double price;
    std::string category;
    bool in_stock;
};

// Combinator: And
template<typename Spec1, typename Spec2>
struct AndSpec {
    Spec1 first;
    Spec2 second;
    bool is_satisfied_by(const Product& p) const {
        return first.is_satisfied_by(p) && second.is_satisfied_by(p);
    }
};

// Combinator: Or
template<typename Spec1, typename Spec2>
struct OrSpec {
    Spec1 first;
    Spec2 second;
    bool is_satisfied_by(const Product& p) const {
        return first.is_satisfied_by(p) || second.is_satisfied_by(p);
    }
};

// Combinator: Not
template<typename Spec>
struct NotSpec {
    Spec inner;
    bool is_satisfied_by(const Product& p) const {
        return !inner.is_satisfied_by(p);
    }
};

// Operator overloads for clean syntax
template<typename S1, typename S2>
auto operator&&(S1 a, S2 b) { return AndSpec<S1, S2>{a, b}; }

template<typename S1, typename S2>
auto operator||(S1 a, S2 b) { return OrSpec<S1, S2>{a, b}; }

template<typename S>
auto operator!(S s) { return NotSpec<S>{s}; }

// Specs
struct InStockSpec {
    bool is_satisfied_by(const Product& p) const { return p.in_stock; }
};
struct CheapSpec {
    double max;
    bool is_satisfied_by(const Product& p) const { return p.price <= max; }
};
struct CategorySpec {
    std::string cat;
    bool is_satisfied_by(const Product& p) const { return p.category == cat; }
};

int main() {
    std::vector<Product> products = {
        {"Laptop", 999.99, "electronics", true},
        {"Headphones", 49.99, "electronics", false},
        {"Cable", 9.99, "electronics", true},
        {"Novel", 12.99, "books", true},
    };

    // Compose: cheap electronics that are in stock
    auto spec = InStockSpec{} && CheapSpec{50.0} && CategorySpec{"electronics"};

    for (const auto& p : products) {
        if (spec.is_satisfied_by(p))
            std::cout << p.name << " ($" << p.price << ")\n";
    }
    // Cable ($9.99)

    // Or: cheap OR out of stock
    auto alt_spec = CheapSpec{20.0} || !InStockSpec{};
    std::cout << "\nCheap or out of stock:\n";
    for (const auto& p : products) {
        if (alt_spec.is_satisfied_by(p))
            std::cout << "  " << p.name << '\n';
    }
    // Headphones, Cable, Novel
}
```

All of this composition resolves at compile time - the compiler sees the full chain of `AndSpec<AndSpec<InStockSpec, CheapSpec>, CategorySpec>` as a nested type and inlines the evaluation completely.

### Q3: Use specifications to filter a container with std::ranges::filter without writing raw predicates

The wrinkle is that `std::views::filter` expects a callable, not an object with `is_satisfied_by`. The `as_filter()` adapter bridges that gap by wrapping any Specification into a lambda the ranges library can call directly. Once you have that adapter, you can pipe specifications into range chains just like any other predicate.

**Answer:**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <ranges>
#include <algorithm>

struct Product {
    std::string name;
    double price;
    std::string category;
    bool in_stock;
};

// Specifications
struct InStock {
    bool is_satisfied_by(const Product& p) const { return p.in_stock; }
};
struct MaxPrice {
    double limit;
    bool is_satisfied_by(const Product& p) const { return p.price <= limit; }
};
struct Category {
    std::string cat;
    bool is_satisfied_by(const Product& p) const { return p.category == cat; }
};

// Adapter: Specification -> ranges-compatible predicate
template<typename Spec>
auto as_filter(Spec spec) {
    return [spec = std::move(spec)](const auto& item) {
        return spec.is_satisfied_by(item);
    };
}

int main() {
    std::vector<Product> products = {
        {"Laptop",     999.99, "electronics", true},
        {"Mouse",       29.99, "electronics", true},
        {"Headphones",  49.99, "electronics", false},
        {"Novel",       12.99, "books",       true},
        {"Cable",        9.99, "electronics", true},
    };

    // Chain specs with ranges::filter
    auto results = products
        | std::views::filter(as_filter(InStock{}))
        | std::views::filter(as_filter(MaxPrice{50.0}))
        | std::views::filter(as_filter(Category{"electronics"}));

    std::cout << "In-stock, cheap electronics:\n";
    for (const auto& p : results) {
        std::cout << "  " << p.name << " ($" << p.price << ")\n";
    }
    // Mouse ($29.99)
    // Cable ($9.99)

    // Reuse specs across different queries
    auto cheap = as_filter(MaxPrice{20.0});
    auto in_stock = as_filter(InStock{});

    auto budget_items = products
        | std::views::filter(cheap)
        | std::views::filter(in_stock);

    std::cout << "\nBudget in-stock items:\n";
    for (const auto& p : budget_items)
        std::cout << "  " << p.name << '\n';
    // Novel, Cable
}
```

Notice that `cheap` and `in_stock` are reusable variables. You define the rule once and thread it through different pipelines. That is the payoff of naming your predicates instead of writing anonymous lambdas every time.

---

## Notes

- The Specification pattern replaces scattered `if` predicates with **named, reusable, composable** business rules.
- Template-based combinators (And/Or/Not) resolve at compile time, giving you zero overhead compared to raw lambdas.
- The `as_filter()` adapter bridges Specification objects to `std::ranges::filter` seamlessly.
- For runtime-composable specs (configured from user input), use `std::function<bool(const T&)>` instead of templates - you lose the compile-time inlining but gain the ability to build rule sets dynamically.
