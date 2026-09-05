# Use deducing-this (explicit object parameter, C++23) for CRTP without templates

**Category:** Modern OOP Patterns  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/member_functions#Explicit_object_member_functions>  

---

## Topic Overview

**Deducing this** (P0847, C++23) lets member functions accept an explicit object parameter (`this auto&& self`). The compiler deduces the actual type - including derived types - without CRTP templates. This eliminates the CRTP boilerplate entirely.

The reason CRTP exists in the first place is that a base class method needs to know the concrete derived type so it can call derived-class methods without going through virtual dispatch. The classic solution forces you to make the base a class template parameterized on the derived type. Deducing-this solves exactly the same problem: the compiler deduces the derived type for you when the member function is called, so you get zero-overhead static dispatch without writing a single template parameter.

### CRTP vs Deducing-This

Here is the same `Printable` mixin written both ways, so the contrast is obvious:

```cpp
CRTP (C++11-C++20):                    Deducing-this (C++23):
template <typename Derived>              struct Printable {
struct Printable {                           void print(this auto const& self) {
    void print() const {                         std::cout << self.to_string();
        auto& self = static_cast               }
            <const Derived&>(*this);         };
        std::cout << self.to_string();
    }                                    struct Name : Printable {
};                                           std::string to_string() const;
                                         };
struct Name : Printable<Name> {
    std::string to_string() const;       // No template parameter!
};                                       // No static_cast!
                                         // Derived type deduced automatically!
```

The CRTP version requires you to pass `Name` as a template argument, then cast `*this` back down to the derived type by hand. Both of those steps are error-prone and purely mechanical. Deducing-this removes them.

---

## Self-Assessment

### Q1: Rewrite a CRTP base class using deducing-this with `this auto&& self`

The key thing to look for here is how the deducing-this version drops both the template parameter and the `static_cast`. The compiler figures out the derived type at the call site instead:

```cpp
#include <iostream>
#include <string>

// BEFORE: CRTP approach (C++11-20)
template <typename Derived>
struct PrintableCRTP {
    void print() const {
        const auto& self = static_cast<const Derived&>(*this);
        std::cout << "[CRTP] " << self.to_string() << "\n";
    }
};

struct NameCRTP : PrintableCRTP<NameCRTP> {
    std::string value;
    std::string to_string() const { return value; }
};

// AFTER: Deducing-this approach (C++23)
struct Printable {
    // 'this auto const& self' deduces the ACTUAL derived type
    void print(this auto const& self) {
        std::cout << "[Deducing-this] " << self.to_string() << "\n";
    }
};

struct Name : Printable {
    std::string value;
    std::string to_string() const { return value; }
};

struct Point : Printable {
    int x, y;
    std::string to_string() const {
        return "(" + std::to_string(x) + ", " + std::to_string(y) + ")";
    }
};

int main() {
    // CRTP style
    NameCRTP n1{"Alice"};
    n1.print();

    // Deducing-this style -- cleaner, no template parameter!
    Name n2{"Bob"};
    n2.print();

    Point p{3, 4};
    p.print();
}
// Expected output:
//   [CRTP] Alice
//   [Deducing-this] Bob
//   [Deducing-this] (3, 4)
```

Notice that `Point` and `Name` both inherit from the same non-template `Printable` - no separate `Printable<Name>` and `Printable<Point>` instantiations.

**Key differences:**

| Aspect | CRTP | Deducing-this |
| --- | --- | --- |
| Base class template? | `template<typename D> struct Base` | `struct Base` (non-template!) |
| Derived inherits | `Derived : Base<Derived>` | `Derived : Base` |
| Access self | `static_cast<const D&>(*this)` | `this auto const& self` |
| Error-prone? | Yes - wrong cast, wrong template arg | No - compiler deduces correctly |
| Works with multiple inheritance? | Possible but verbose | Natural |

---

### Q2: Show how deducing-this enables recursive lambdas without `std::function` overhead

This use case surprises people at first. The same mechanism that lets a member function deduce its derived type also lets a lambda refer to itself without capturing it through `std::function`. The reason `std::function` is expensive is that it type-erases the callable and may heap-allocate it. Deducing-this avoids all of that:

```cpp
#include <iostream>

int main() {
    // BEFORE C++23: recursive lambda needs std::function (heap allocation!)
    // std::function<int(int)> fib = [&fib](int n) -> int {
    //     return n <= 1 ? n : fib(n-1) + fib(n-2);
    // };

    // C++23: recursive lambda with deducing-this -- ZERO overhead!
    auto fib = [](this auto const& self, int n) -> int {
        return n <= 1 ? n : self(n - 1) + self(n - 2);
    };

    std::cout << "fib(10) = " << fib(10) << "\n";

    // Tree traversal example
    struct Node {
        int value;
        Node* left = nullptr;
        Node* right = nullptr;
    };

    //       1
    //      / \
    //     2   3
    //    / \
    //   4   5
    Node n4{4}, n5{5}, n3{3};
    Node n2{2, &n4, &n5};
    Node root{1, &n2, &n3};

    // Recursive lambda for in-order traversal
    auto traverse = [](this auto const& self, const Node* node) -> void {
        if (!node) return;
        self(node->left);
        std::cout << node->value << " ";
        self(node->right);
    };

    std::cout << "In-order: ";
    traverse(&root);
    std::cout << "\n";
}
// Expected output:
//   fib(10) = 55
//   In-order: 4 2 5 1 3
```

The lambda's own type is deduced through `this auto const& self`, so it can call `self(...)` just like a named function. No capture, no heap, fully inlinable.

**Why this is better than `std::function`:**

| Approach | Heap allocation | Virtual call | Inlinable |
| --- | --- | --- | --- |
| `std::function<int(int)>` | Yes (~32 bytes + possible heap) | Yes (type-erased) | No |
| Deducing-this lambda | **No** | **No** | **Yes** |

---

### Q3: Implement a builder pattern where method chaining returns the correct derived type

This is the other classic CRTP headache: when a base class method returns `*this`, it returns a `BuilderBase&`, so you lose access to the derived class methods mid-chain. Deducing-this fixes this naturally because `self` is already the derived type:

```cpp
#include <iostream>
#include <string>
#include <optional>
#include <utility>

struct BuilderBase {
    std::string name_;
    int priority_ = 0;

    // Deducing-this: returns the ACTUAL derived type for chaining!
    auto& set_name(this auto& self, std::string name) {
        self.name_ = std::move(name);
        return self;  // returns HttpRequestBuilder&, not BuilderBase&!
    }

    auto& set_priority(this auto& self, int p) {
        self.priority_ = p;
        return self;
    }
};

struct HttpRequestBuilder : BuilderBase {
    std::string url_;
    std::string method_ = "GET";
    std::optional<std::string> body_;

    auto& set_url(this auto& self, std::string url) {
        self.url_ = std::move(url);
        return self;
    }

    auto& set_method(this auto& self, std::string method) {
        self.method_ = std::move(method);
        return self;
    }

    auto& set_body(this auto& self, std::string body) {
        self.body_ = std::move(body);
        return self;
    }

    void execute() const {
        std::cout << method_ << " " << url_ << "\n";
        std::cout << "  Name: " << name_ << ", Priority: " << priority_ << "\n";
        if (body_)
            std::cout << "  Body: " << *body_ << "\n";
    }
};

int main() {
    // Method chaining across base AND derived -- all return HttpRequestBuilder&!
    HttpRequestBuilder{}
        .set_name("API Call")           // base method -- returns HttpRequestBuilder&
        .set_priority(5)                // base method -- returns HttpRequestBuilder&
        .set_url("https://api.example.com/users")  // derived method
        .set_method("POST")             // derived method
        .set_body(R"({"name": "Alice"})")  // derived method
        .execute();

    // Without deducing-this, set_name() would return BuilderBase&
    // and then .set_url() would fail (BuilderBase has no set_url)!
}
// Expected output:
//   POST https://api.example.com/users
//     Name: API Call, Priority: 5
//     Body: {"name": "Alice"}
```

Compare that with what CRTP requires to achieve the same result - you need both the template parameter and a manual `static_cast`:

```cpp
template <typename Derived>
struct BuilderBase {
    Derived& set_name(std::string s) {
        name_ = s;
        return static_cast<Derived&>(*this);  // manual cast
    }
};
struct HttpBuilder : BuilderBase<HttpBuilder> { ... };
```

Deducing-this is simply the cleaner solution for this problem.

---

## Notes

- **Compiler support (2024):** GCC 14+, Clang 18+, MSVC 19.38+. Check with `__cpp_explicit_this_parameter >= 202110L`.
- **Deducing-this also simplifies:** const/non-const overload pairs, operator[] with ref-qualifiers, mixin compositions.
- **The explicit object parameter** is always the first parameter and uses the `this` keyword: `void f(this Self& self, int x)`.
- **Cannot combine** with `static`, `virtual`, or cv/ref-qualifiers on the function. It replaces the implicit `this` pointer.
- **P2481 (forwarding reference for implicit object):** Related proposal for even more flexibility.
- **Migration tip:** You can mix CRTP and deducing-this in the same codebase - they're not mutually exclusive.
