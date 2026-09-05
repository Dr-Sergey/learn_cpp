# Use the Hidden-Friend Idiom to Improve Overload Resolution

**Category:** Templates and Generic Programming  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/friend>  

---

## Topic Overview

### What Is a Hidden Friend

A **hidden friend** is a `friend` function defined **inside the class body**. It is:

- **Not** a member function
- **Not** visible via normal (unqualified or qualified) lookup
- **Only** found through **Argument-Dependent Lookup (ADL)**

```cpp
class Widget {
    int val_;
public:
    explicit Widget(int v) : val_(v) {}

    // Hidden friend — defined INSIDE the class body
    friend bool operator==(const Widget& a, const Widget& b) {
        return a.val_ == b.val_;
    }
};
// operator== is NOT a member. NOT visible via ::operator==.
// Found ONLY when at least one argument is a Widget (ADL).
```

### Why Hidden Friends Matter

The reason this idiom exists is overload set pollution. When you define `operator==` as an ordinary function at namespace scope, the compiler must consider it as a candidate for every `operator==` call in that namespace - even calls that have nothing to do with your type. Hidden friends are invisible to those unrelated calls, which keeps the overload set small and makes both compilation and error messages faster and cleaner.

| Lookup Type | Finds Hidden Friend? |
| --- | :---: |
| Unqualified lookup without ADL-triggering args | No |
| Qualified lookup (e.g., `ns::f(x)`) | No |
| ADL (argument's class is in scope) | Yes |

### Benefits

| Benefit | Explanation |
| --- | --- |
| **Reduced overload set** | The function only participates when an argument's type matches, avoiding spurious conversions |
| **Faster compilation** | Fewer candidates for the compiler to consider |
| **Implicit conversions on both sides** | Unlike member operators, both LHS and RHS can be implicitly converted |
| **Cleaner namespace** | Doesn't pollute the enclosing namespace |
| **Symmetric operators** | `a == b` and `b == a` both work even if one side needs conversion |

### ADL Recap

**Argument-Dependent Lookup** examines the namespaces and classes associated with a function's arguments. For `operator==(x, y)`, if `x` is `Widget`, the compiler looks inside `Widget`'s definition and finds the hidden friend. No namespace prefix needed - the argument type is the key.

---

## Self-Assessment

### Q1: Implement `operator==` as a hidden friend (defined inside the class body) and explain ADL

When the compiler sees `red == also_red`, it rewrites that as `operator==(red, also_red)`. Because `red` is a `Color`, ADL searches inside the `Color` class definition for a matching `friend` function. It finds `operator==` there and uses it. The call is completely natural even though the function has no namespace-scope declaration.

```cpp
#include <iostream>
#include <string>

class Color {
    int r_, g_, b_;
public:
    Color(int r, int g, int b) : r_(r), g_(g), b_(b) {}

    // Hidden friend operator==
    // Defined inside the class body -> only findable via ADL
    friend bool operator==(const Color& a, const Color& b) {
        return a.r_ == b.r_ && a.g_ == b.g_ && a.b_ == b.b_;
    }

    // Hidden friend operator!=  (pre-C++20 you need this; C++20 auto-generates it)
    friend bool operator!=(const Color& a, const Color& b) {
        return !(a == b);  // calls the hidden friend above via ADL
    }

    // Hidden friend for streaming
    friend std::ostream& operator<<(std::ostream& os, const Color& c) {
        return os << "Color(" << c.r_ << "," << c.g_ << "," << c.b_ << ")";
    }
};

int main() {
    Color red(255, 0, 0);
    Color blue(0, 0, 255);
    Color also_red(255, 0, 0);

    // ADL kicks in: the compiler sees Color arguments, looks inside Color
    // for friend functions, and finds operator==
    std::cout << std::boolalpha;
    std::cout << "red == also_red: " << (red == also_red) << "\n";   // true
    std::cout << "red == blue:     " << (red == blue) << "\n";       // false
    std::cout << "red != blue:     " << (red != blue) << "\n";       // true
    std::cout << "Output via ADL:  " << red << "\n";

    // This demonstrates ADL: the call `red == also_red` is actually
    // `operator==(red, also_red)`. Since `red` is of type `Color`,
    // ADL searches the class `Color` for matching friend functions.
    return 0;
}
```

**Expected output:**

```text
red == also_red: true
red == blue:     false
red != blue:     true
Output via ADL:  Color(255,0,0)
```

### Q2: Show that hidden friends are not found by qualified or unqualified lookup - only by ADL

The point of this example is to make the invisibility concrete. You can call `distance(a, b)` naturally, but you cannot take its address as `&geom::distance` or call it as `geom::distance(a, b)` - it has no namespace-scope name.

```cpp
#include <iostream>

namespace geom {

class Point {
    double x_, y_;
public:
    Point(double x, double y) : x_(x), y_(y) {}

    // Hidden friend — defined inside class body
    friend double distance(const Point& a, const Point& b) {
        double dx = a.x_ - b.x_;
        double dy = a.y_ - b.y_;
        return std::sqrt(dx * dx + dy * dy);
    }

    // Hidden friend operator+
    friend Point operator+(const Point& a, const Point& b) {
        return Point(a.x_ + b.x_, a.y_ + b.y_);
    }

    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        return os << "(" << p.x_ << "," << p.y_ << ")";
    }
};

}  // namespace geom

int main() {
    geom::Point a(0, 0), b(3, 4);

    // ADL: `a` and `b` are geom::Point -> compiler looks inside geom::Point
    double d = distance(a, b);
    std::cout << "distance(a, b) = " << d << "\n";  // 5

    // ADL for operator+
    auto c = a + b;
    std::cout << "a + b = " << c << "\n";

    // Qualified lookup — would NOT compile:
    // double d2 = geom::distance(a, b);  // ERROR: 'distance' is not a member of 'geom'

    // Unqualified lookup without ADL-triggering arguments — would NOT compile:
    // (There's no way to call distance(int, int) since int doesn't trigger ADL into Point)

    // Proof that it's truly hidden:
    // The function `distance` doesn't exist as geom::distance.
    // It ONLY exists as a hidden friend of geom::Point.
    // You can't take its address: auto fp = &geom::distance; // ERROR

    std::cout << "\nHidden friends are invisible to qualified lookup.\n";
    std::cout << "They are only found when an argument's type triggers ADL.\n";

    return 0;
}
```

**Expected output:**

```text
distance(a, b) = 5
a + b = (3,4)

Hidden friends are invisible to qualified lookup.
They are only found when an argument's type triggers ADL.
```

### Q3: Compare hidden friends with non-member non-friends for the same operator

Both styles allow the same symmetric comparisons. The difference shows up only when unrelated code is in scope - the non-member `operator==` for `Label` is always visible, while the hidden-friend `operator==` for `Token` is invisible until a `Token` argument appears.

| Aspect | Hidden Friend | Non-Member Non-Friend |
| --- | --- | --- |
| **Where defined** | Inside the class body | Outside the class, at namespace scope |
| **Found by qualified lookup** | No | Yes |
| **Found by unqualified lookup** | No (only ADL) | Yes |
| **Overload set participation** | Only when an argument is the class type | Always (if in scope) |
| **Access to private members** | Yes (it's a `friend`) | No (unless declared `friend` separately) |
| **Implicit conversions** | Both sides (symmetric) | Both sides (symmetric) |
| **Compilation speed impact** | Reduced overload sets -> faster | May slow overload resolution |

```cpp
#include <iostream>
#include <string>

// === Approach 1: Hidden Friend ===
class Token {
    std::string text_;
public:
    Token(const std::string& t) : text_(t) {}  // implicit conversion from string

    // Hidden friend — only found via ADL
    friend bool operator==(const Token& a, const Token& b) {
        return a.text_ == b.text_;
    }

    friend std::ostream& operator<<(std::ostream& os, const Token& t) {
        return os << "Token(\"" << t.text_ << "\")";
    }
};

// === Approach 2: Non-Member Non-Friend ===
class Label {
    std::string text_;
public:
    Label(const std::string& t) : text_(t) {}  // implicit conversion from string
    const std::string& text() const { return text_; }
};

// Ordinary free function at namespace scope — always visible
bool operator==(const Label& a, const Label& b) {
    return a.text() == b.text();  // must use public interface
}

std::ostream& operator<<(std::ostream& os, const Label& l) {
    return os << "Label(\"" << l.text() << "\")";
}

int main() {
    std::cout << std::boolalpha;

    // Both support symmetric implicit conversions:
    Token t("hello");
    std::cout << "Token == Token:    " << (t == Token("hello")) << "\n";  // true
    std::cout << "string == Token:   " << (std::string("hello") == t) << "\n";  // true (ADL)

    Label l("hello");
    std::cout << "Label == Label:    " << (l == Label("hello")) << "\n";  // true
    std::cout << "string == Label:   " << (std::string("hello") == l) << "\n";  // true

    // Key difference: The Label operator== is visible even without Label arguments
    // in the same namespace. The Token operator== is completely invisible unless
    // at least one argument is a Token.

    // In large codebases, hidden friends dramatically reduce the overload set
    // the compiler must consider, improving both compile times and error messages.

    std::cout << "\n=== Summary ===\n";
    std::cout << "Hidden friend: only found via ADL -> smaller overload sets\n";
    std::cout << "Non-member: always in scope -> can cause unwanted conversions\n";
    std::cout << "Both allow symmetric implicit conversions on LHS and RHS\n";
    std::cout << "Hidden friends can access private members directly\n";

    return 0;
}
```

**Expected output:**

```text
Token == Token:    true
string == Token:   true
Label == Label:    true
string == Label:   true

=== Summary ===
Hidden friend: only found via ADL -> smaller overload sets
Non-member: always in scope -> can cause unwanted conversions
Both allow symmetric implicit conversions on LHS and RHS
Hidden friends can access private members directly
```

---

## Notes

- **Hidden friends** are the idiomatic C++ way to define operators and swap for your types.
- They solve the "too many candidates" problem by only appearing when ADL brings them in.
- In C++20, `operator<=>` and `operator==` as hidden friends enable all six comparison operators automatically.
- The standard library uses hidden friends extensively (e.g., `std::ranges` customization points).
- **Rule of thumb:** If a non-member function conceptually belongs to a class, make it a hidden friend.
- `swap` as a hidden friend is the modern pattern - enables `using std::swap; swap(a, b);` idiom.
