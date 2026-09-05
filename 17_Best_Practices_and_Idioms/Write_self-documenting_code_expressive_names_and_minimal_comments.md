# Write self-documenting code: expressive names and minimal comments

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#nl-naming-and-layout-rules>  

---

## Topic Overview

**Self-documenting code** uses expressive names and clear structure so that comments become mostly unnecessary. The mental model is simple: the code tells you *what* and *how*; comments explain *why*. If you need a comment to explain what the code does, that's a signal the code itself could be clearer.

```cpp
// BAD: needs comment because the code is cryptic
int f(int n) { return n <= 1 ? 1 : n * f(n-1); }  // factorial

// GOOD: name says it all
int factorial(int n) { return n <= 1 ? 1 : n * factorial(n - 1); }
```

---

## Self-Assessment

### Q1: Rewrite cryptic code with descriptive names

Here the same algorithm appears twice - once with single-letter variables and once with names that explain what each value represents. You'll notice that the "good" version doesn't even need a comment anywhere: the variable names tell the whole story.

```cpp
#include <iostream>
#include <vector>
#include <cmath>
#include <numeric>

// ========= BAD: cryptic single-letter variables =========
double f(const std::vector<double>& v) {
    double s = 0;  // what is s?
    for (auto x : v) s += x;  // what is x?
    double m = s / v.size();  // what is m?
    double ss = 0;
    for (auto x : v) ss += (x - m) * (x - m);
    return std::sqrt(ss / v.size());  // what does this return?
}

// ========= GOOD: self-documenting =========
double standard_deviation(const std::vector<double>& measurements) {
    const double sum = std::accumulate(
        measurements.begin(), measurements.end(), 0.0);
    const double mean = sum / measurements.size();

    double squared_diff_sum = 0.0;
    for (const double measurement : measurements) {
        const double deviation = measurement - mean;
        squared_diff_sum += deviation * deviation;
    }

    const double variance = squared_diff_sum / measurements.size();
    return std::sqrt(variance);
}

int main() {
    std::vector<double> temperatures = {20.1, 21.3, 19.8, 22.0, 20.5};

    // No comment needed - the function name explains everything
    double temp_std_dev = standard_deviation(temperatures);
    std::cout << "Temperature std dev: " << temp_std_dev << '\n';
}
// Expected output:
// Temperature std dev: 0.787274
```

`standard_deviation`, `measurements`, `mean`, `deviation`, `variance` - you can read the good version like a textbook definition of standard deviation. The bad version makes you reverse-engineer the algorithm from its cryptic variable names.

### Q2: Comment *what* (bad) vs comment *why* (good)

This is the subtler skill. Bad comments restate what the code already says; good comments explain the reasoning behind a decision that the code can't express on its own - a business rule, a workaround for a known bug, or a non-obvious constraint.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

struct Order {
    int id;
    double amount;
    bool is_priority;
};

void process_orders(std::vector<Order>& orders) {
    // BAD COMMENTS: describe WHAT (redundant with code)
    // Sort orders by amount in descending order
    std::sort(orders.begin(), orders.end(),
              [](const Order& a, const Order& b) {
                  return a.amount > b.amount;
              });

    // GOOD COMMENTS: explain WHY (not obvious from code)
    // Process highest-value orders first to maximize revenue
    // if we hit the daily processing limit.
    std::sort(orders.begin(), orders.end(),
              [](const Order& a, const Order& b) {
                  return a.amount > b.amount;
              });

    // BAD: "increment i" - obviously!
    // i++;  // increment i

    // GOOD: explain the non-obvious business rule
    // Skip the first order - it's the sentinel from the legacy system.
    auto it = orders.begin() + 1;

    // GOOD: explain a workaround
    // Using stable_sort because the payment gateway requires
    // equal-priority orders to arrive in submission order (bug #4521).
    std::stable_sort(orders.begin(), orders.end(),
                     [](const Order& a, const Order& b) {
                         return a.is_priority > b.is_priority;
                     });

    for (const auto& o : orders)
        std::cout << "Order #" << o.id << ": $" << o.amount << '\n';
}

int main() {
    std::vector<Order> orders = {{1, 50.0, false}, {2, 200.0, true}, {3, 75.0, false}};
    process_orders(orders);
}
// Expected output:
// Order #2: $200
// Order #1: $50
// Order #3: $75
```

The `stable_sort` comment referencing bug #4521 is a perfect example: you cannot possibly infer from the code alone that stable ordering was chosen because of a payment gateway constraint. That context only exists in someone's head and a bug tracker, and a comment is the right place to record it.

### Q3: Split a large function into well-named helpers (SRP)

When a function grows large, one of the clearest improvements you can make is to extract well-named helpers. The result is that the top-level function reads like a high-level description of the algorithm, and each helper has one narrow job. You almost don't need comments at all because the function names do the explaining.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <sstream>
#include <numeric>

struct SalesRecord {
    std::string product;
    int quantity;
    double price;
};

struct SalesReport {
    double total_revenue;
    std::string best_seller;
    int total_items;
};

// Each helper has ONE clear responsibility:

double calculate_total_revenue(const std::vector<SalesRecord>& records) {
    double total = 0;
    for (const auto& r : records)
        total += r.quantity * r.price;
    return total;
}

std::string find_best_seller(const std::vector<SalesRecord>& records) {
    const auto& best = *std::max_element(
        records.begin(), records.end(),
        [](const SalesRecord& a, const SalesRecord& b) {
            return (a.quantity * a.price) < (b.quantity * b.price);
        });
    return best.product;
}

int count_total_items(const std::vector<SalesRecord>& records) {
    int total = 0;
    for (const auto& r : records) total += r.quantity;
    return total;
}

std::string format_report(const SalesReport& report) {
    std::ostringstream oss;
    oss << "Revenue: $" << report.total_revenue << '\n'
        << "Best seller: " << report.best_seller << '\n'
        << "Items sold: " << report.total_items;
    return oss.str();
}

// Main function reads like a high-level description:
SalesReport generate_sales_report(const std::vector<SalesRecord>& records) {
    return {
        .total_revenue = calculate_total_revenue(records),
        .best_seller = find_best_seller(records),
        .total_items = count_total_items(records)
    };
}

int main() {
    std::vector<SalesRecord> records = {
        {"Widget A", 100, 9.99},
        {"Gadget B", 50, 24.99},
        {"Tool C", 200, 4.99}
    };

    auto report = generate_sales_report(records);
    std::cout << format_report(report) << '\n';
}
// Expected output:
// Revenue: $3247
// Best seller: Widget A
// Items sold: 350
```

`generate_sales_report` is three lines that read almost like English. You don't need to understand how `calculate_total_revenue` works to understand what `generate_sales_report` does - the names carry the meaning.

---

## Notes

- Name functions as verbs: `calculate_total()`, `find_best()`, `validate_input()`.
- Name variables as nouns: `total_revenue`, `max_temperature`, `user_count`.
- Name booleans as questions: `is_valid`, `has_data`, `should_retry`.
- Avoid abbreviations: `temperature` not `temp`, `index` not `idx`.
- If you need a comment to explain *what*, the code needs better names.
