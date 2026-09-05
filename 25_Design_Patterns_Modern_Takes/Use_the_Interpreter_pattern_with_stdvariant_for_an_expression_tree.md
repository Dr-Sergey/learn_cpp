# Use the Interpreter pattern with std::variant for an expression tree

**Category:** Design Patterns Modern Takes  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant/visit>  

---

## Topic Overview

The **Interpreter pattern** represents a language grammar as a tree of objects that can be evaluated. Using `std::variant` + `std::visit` instead of virtual classes, the expression tree becomes a **closed sum type** - adding a new visitor (like `eval` or `print`) requires no changes to the expression types, while adding a new expression type forces updating all visitors at compile time.

The reason this matters is the classic "expression problem" trade-off. With virtual dispatch, you can add new types freely but adding new operations is painful. With `std::variant`, it's the reverse: new operations (visitors) are trivially cheap to add, but adding a new expression type means touching every existing visitor. For expression trees where the grammar is stable but you keep adding operations, `std::variant` wins.

### Expression Tree Structure

To make this concrete, here's the tree for `(1 + (-2)) * 3`. Each node in the tree is one of the variant alternatives, and evaluating the tree means walking it with a visitor.

```cpp
    Mul
   /   \
  Add   Lit(3)
 / \
Lit(1)  Neg
         |
       Lit(2)

eval:  (1 + (-2)) * 3  =  -3
print: ((1 + (-(2))) * 3)
```

---

## Self-Assessment

### Q1: Define Expr = std::variant<Literal, Add, Mul, Neg> and implement eval(Expr) with std::visit

The tricky part here is that the variant is **recursive** - an `Add` contains two sub-expressions, which are themselves `Expr`. Because `std::variant` needs to know its size at compile time, you can't nest a variant inside itself directly. The solution is to use `unique_ptr<Expr>` as indirection: the `Add` struct holds pointers to child nodes rather than the nodes themselves.

The `overloaded` helper at the top is a standard trick for building a visitor on the spot from a set of lambdas. Each lambda handles one alternative in the variant.

**Answer:**

```cpp
#include <iostream>
#include <variant>
#include <memory>

// Forward declarations needed for recursive variant
struct Literal;
struct Add;
struct Mul;
struct Neg;

// Expr = recursive variant (using unique_ptr for indirection)
using Expr = std::variant<Literal, Add, Mul, Neg>;
using ExprPtr = std::unique_ptr<Expr>;

struct Literal { double value; };
struct Add { ExprPtr left, right; };
struct Mul { ExprPtr left, right; };
struct Neg { ExprPtr operand; };

// Helper to create expression nodes
ExprPtr lit(double v)            { return std::make_unique<Expr>(Literal{v}); }
ExprPtr add(ExprPtr a, ExprPtr b){ return std::make_unique<Expr>(Add{std::move(a), std::move(b)}); }
ExprPtr mul(ExprPtr a, ExprPtr b){ return std::make_unique<Expr>(Mul{std::move(a), std::move(b)}); }
ExprPtr neg(ExprPtr a)           { return std::make_unique<Expr>(Neg{std::move(a)}); }

// eval visitor
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

double eval(const Expr& expr) {
    return std::visit(overloaded{
        [](const Literal& l) -> double { return l.value; },
        [](const Add& a)     -> double { return eval(*a.left) + eval(*a.right); },
        [](const Mul& m)     -> double { return eval(*m.left) * eval(*m.right); },
        [](const Neg& n)     -> double { return -eval(*n.operand); },
    }, expr);
}

int main() {
    // Build: (1 + (-2)) * 3
    auto expr = mul(
        add(lit(1), neg(lit(2))),
        lit(3)
    );

    std::cout << "Result: " << eval(*expr) << '\n';  // -3

    // Build: 5 * (3 + 4)
    auto expr2 = mul(lit(5), add(lit(3), lit(4)));
    std::cout << "Result: " << eval(*expr2) << '\n';  // 35
}
```

Notice that `eval` is recursive - when it hits an `Add` node it calls itself on both children. This is exactly how you would walk any tree, just written as a visitor dispatch instead of virtual method calls.

### Q2: Add a pretty_print(Expr) visitor without modifying the Expr types

Here's the thing that makes the variant approach really attractive: adding a new operation means writing a new free function with a new `std::visit` call. You don't touch the `Literal`, `Add`, `Mul`, or `Neg` types at all. Compare this to the virtual-dispatch version, where you would have to add a new pure virtual method to a base class and then implement it in every derived class.

**Answer:**

```cpp
#include <iostream>
#include <variant>
#include <memory>
#include <string>

struct Literal;
struct Add;
struct Mul;
struct Neg;

using Expr = std::variant<Literal, Add, Mul, Neg>;
using ExprPtr = std::unique_ptr<Expr>;

struct Literal { double value; };
struct Add { ExprPtr left, right; };
struct Mul { ExprPtr left, right; };
struct Neg { ExprPtr operand; };

ExprPtr lit(double v)            { return std::make_unique<Expr>(Literal{v}); }
ExprPtr add(ExprPtr a, ExprPtr b){ return std::make_unique<Expr>(Add{std::move(a), std::move(b)}); }
ExprPtr mul(ExprPtr a, ExprPtr b){ return std::make_unique<Expr>(Mul{std::move(a), std::move(b)}); }
ExprPtr neg(ExprPtr a)           { return std::make_unique<Expr>(Neg{std::move(a)}); }

template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// eval - no changes to Expr types
double eval(const Expr& e) {
    return std::visit(overloaded{
        [](const Literal& l) -> double { return l.value; },
        [](const Add& a) -> double { return eval(*a.left) + eval(*a.right); },
        [](const Mul& m) -> double { return eval(*m.left) * eval(*m.right); },
        [](const Neg& n) -> double { return -eval(*n.operand); },
    }, e);
}

// pretty_print - NEW visitor, Expr types untouched
std::string pretty_print(const Expr& e) {
    return std::visit(overloaded{
        [](const Literal& l) -> std::string {
            auto s = std::to_string(l.value);
            // Remove trailing zeros
            s.erase(s.find_last_not_of('0') + 1, std::string::npos);
            s.erase(s.find_last_not_of('.') + 1, std::string::npos);
            return s;
        },
        [](const Add& a) -> std::string {
            return "(" + pretty_print(*a.left) + " + " + pretty_print(*a.right) + ")";
        },
        [](const Mul& m) -> std::string {
            return "(" + pretty_print(*m.left) + " * " + pretty_print(*m.right) + ")";
        },
        [](const Neg& n) -> std::string {
            return "(-" + pretty_print(*n.operand) + ")";
        },
    }, e);
}

// count_nodes - another visitor, still no changes
int count_nodes(const Expr& e) {
    return std::visit(overloaded{
        [](const Literal&) { return 1; },
        [](const Add& a) { return 1 + count_nodes(*a.left) + count_nodes(*a.right); },
        [](const Mul& m) { return 1 + count_nodes(*m.left) + count_nodes(*m.right); },
        [](const Neg& n) { return 1 + count_nodes(*n.operand); },
    }, e);
}

int main() {
    auto expr = mul(add(lit(1), neg(lit(2))), lit(3));

    std::cout << "Expression: " << pretty_print(*expr) << '\n';
    std::cout << "Result:     " << eval(*expr) << '\n';
    std::cout << "Nodes:      " << count_nodes(*expr) << '\n';
    // Expression: ((1 + (-2)) * 3)
    // Result:     -3
    // Nodes:      5
}
```

Three different operations (`eval`, `pretty_print`, `count_nodes`) and not a single modification to the node structs. That's the payoff of the closed sum type approach.

### Q3: Show how adding a new expression node forces updating all visitors (exhaustive checking)

This is the other side of the trade-off. When you add `Pow` to the variant, the compiler forces you to handle it in every existing visitor. If you forget, you get a **compile error** - not a silent runtime crash or a missed case. This exhaustiveness guarantee is one of the strongest arguments for `std::variant` over virtual dispatch, where a missing override silently calls the base class method.

**Answer:**

```cpp
#include <iostream>
#include <variant>
#include <memory>
#include <string>
#include <cmath>

struct Literal;
struct Add;
struct Mul;
struct Neg;
struct Pow;  // NEW expression node!

// Adding Pow to variant forces ALL visitors to handle it
using Expr = std::variant<Literal, Add, Mul, Neg, Pow>;
using ExprPtr = std::unique_ptr<Expr>;

struct Literal { double value; };
struct Add { ExprPtr left, right; };
struct Mul { ExprPtr left, right; };
struct Neg { ExprPtr operand; };
struct Pow { ExprPtr base, exponent; };  // NEW

ExprPtr lit(double v)            { return std::make_unique<Expr>(Literal{v}); }
ExprPtr add(ExprPtr a, ExprPtr b){ return std::make_unique<Expr>(Add{std::move(a), std::move(b)}); }
ExprPtr mul(ExprPtr a, ExprPtr b){ return std::make_unique<Expr>(Mul{std::move(a), std::move(b)}); }
ExprPtr neg(ExprPtr a)           { return std::make_unique<Expr>(Neg{std::move(a)}); }
ExprPtr pw(ExprPtr b, ExprPtr e) { return std::make_unique<Expr>(Pow{std::move(b), std::move(e)}); }

template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// eval MUST handle Pow - otherwise compile error
double eval(const Expr& e) {
    return std::visit(overloaded{
        [](const Literal& l) { return l.value; },
        [](const Add& a) { return eval(*a.left) + eval(*a.right); },
        [](const Mul& m) { return eval(*m.left) * eval(*m.right); },
        [](const Neg& n) { return -eval(*n.operand); },
        [](const Pow& p) { return std::pow(eval(*p.base), eval(*p.exponent)); },
        // Without this line: error: no matching function for call to
        // 'overloaded<...>::operator()(const Pow&)'
    }, e);
}

// pretty_print MUST also handle Pow
std::string pretty_print(const Expr& e) {
    return std::visit(overloaded{
        [](const Literal& l) { return std::to_string(static_cast<int>(l.value)); },
        [](const Add& a) { return "(" + pretty_print(*a.left) + " + " + pretty_print(*a.right) + ")"; },
        [](const Mul& m) { return "(" + pretty_print(*m.left) + " * " + pretty_print(*m.right) + ")"; },
        [](const Neg& n) { return "(-" + pretty_print(*n.operand) + ")"; },
        [](const Pow& p) { return "(" + pretty_print(*p.base) + " ^ " + pretty_print(*p.exponent) + ")"; },
        // Without this line: COMPILE ERROR - exhaustive check forces update
    }, e);
}

int main() {
    // 2^10 = 1024
    auto expr = pw(lit(2), lit(10));
    std::cout << pretty_print(*expr) << " = " << eval(*expr) << '\n';
    // (2 ^ 10) = 1024

    // (3 + 2)^2 = 25
    auto expr2 = pw(add(lit(3), lit(2)), lit(2));
    std::cout << pretty_print(*expr2) << " = " << eval(*expr2) << '\n';
    // ((3 + 2) ^ 2) = 25
}
```

The compile error when you forget a case is the key feature here. In a large codebase with many visitors spread across many files, this is what keeps you from accidentally shipping broken behavior. The variant literally will not compile until every handler is present.

---

## Notes

- `std::variant` enforces **exhaustive visiting** - no risk of forgetting a case (unlike virtual dispatch where a missing override silently calls the base).
- Recursive variants need indirection (`unique_ptr<Expr>`) because the variant's size must be known at compile time, and you can't have a type that contains itself directly.
- The trade-off: adding a new **visitor** (operation) is easy - just write a new free function. Adding a new **expression type** forces updating all visitors. This is the classic "expression problem."
- Use `auto&&` catch-all in a visitor only for explicitly optional cases; avoid it if you want the exhaustiveness guarantee.
