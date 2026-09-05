# Apply the Builder pattern using method chaining and C++ return *this idiom

**Category:** Modern OOP Patterns  
**Standard:** C++11, C++20, C++23  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-builder>  

---

## Topic Overview

The Builder pattern constructs complex objects step by step using **method chaining** - each setter returns a reference to `*this`, enabling a fluent API. In C++ this avoids constructors with many parameters (the "telescoping constructor" anti-pattern) while preserving compile-time safety.

### The Problem: Telescoping Constructors

You've probably seen a constructor call that looks like this. The problem is obvious the moment you read it:

```cpp
// BAD: which bool is what? What does 8080 mean?
HttpRequest req("GET", "/api/users", "application/json", true, 8080, false, true, "");
```

That call is completely unreadable. You can't tell which `true` controls redirects, which controls TLS, or what that `""` at the end is for. The Builder pattern solves this by naming each setter.

### Method Chaining with return `*this`

The trick is simple: every setter returns a reference to the builder itself, so you can chain the next call directly onto the result.

```cpp
class Builder {
public:
    Builder& set_x(int x) { x_ = x; return *this; }
    Builder& set_y(int y) { y_ = y; return *this; }
    Product build() { return Product{x_, y_}; }
private:
    int x_ = 0, y_ = 0;
};

auto product = Builder{}.set_x(10).set_y(20).build();
//                      ^^^^^^^^^^^^^^^^^^^^^^^^^^
//                      method chaining via return *this
```

Each setter modifies the builder and hands `*this` back, so the next call in the chain still operates on the same builder object.

### The Inheritance Problem with Builders

The moment you try to inherit from a builder, you run into a type-loss problem. A base-class setter can only return `Base&`, not `Derived&`, which breaks the chain as soon as you call a base method:

```cpp
class Base {
    Base& set_name(string n) { ... return *this; }  // returns Base&
};
class Derived : public Base {
    Derived& set_color(string c) { ... return *this; }
};

Derived{}.set_name("x")   // returns Base& - LOST Derived type!
         .set_color("red") // ERROR: Base has no set_color
```

The reason this trips people up is that `set_name` is physically called on a `Derived` object, but the return type is `Base&`, so the compiler forgets the derived type and you can no longer call derived methods.

**Solutions:**

1. **CRTP** (C++11) - `Base<Derived>` returns `Derived&`
2. **Deducing this** (C++23) - `this auto&& self` deduces the actual type

---

## Self-Assessment

### Q1: Implement a `RequestBuilder` with chainable setters returning `RequestBuilder&` and a final `build()`

Here's a realistic HTTP request builder. Pay attention to how `build()` does validation before handing back the finished object - that's one of the pattern's best features.

**Solution - HTTP Request Builder:**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <sstream>
#include <stdexcept>
#include <optional>
#include <utility>

struct HttpRequest {
    std::string method;
    std::string url;
    std::vector<std::pair<std::string, std::string>> headers;
    std::optional<std::string> body;
    int timeout_ms = 30000;
    bool follow_redirects = true;
};

class RequestBuilder {
    HttpRequest req_;

public:
    // Each setter returns *this by reference for chaining
    RequestBuilder& method(std::string m) {
        req_.method = std::move(m);
        return *this;
    }

    RequestBuilder& url(std::string u) {
        req_.url = std::move(u);
        return *this;
    }

    RequestBuilder& header(std::string key, std::string value) {
        req_.headers.emplace_back(std::move(key), std::move(value));
        return *this;
    }

    RequestBuilder& body(std::string b) {
        req_.body = std::move(b);
        return *this;
    }

    RequestBuilder& timeout(int ms) {
        req_.timeout_ms = ms;
        return *this;
    }

    RequestBuilder& follow_redirects(bool f) {
        req_.follow_redirects = f;
        return *this;
    }

    // Terminal operation: validate and build
    HttpRequest build() {
        if (req_.method.empty())
            throw std::logic_error("HTTP method is required");
        if (req_.url.empty())
            throw std::logic_error("URL is required");
        return std::move(req_);
    }
};

std::ostream& operator<<(std::ostream& os, const HttpRequest& r) {
    os << r.method << " " << r.url << "\n";
    for (const auto& [k, v] : r.headers)
        os << "  " << k << ": " << v << "\n";
    if (r.body) os << "  Body: " << *r.body << "\n";
    os << "  Timeout: " << r.timeout_ms << "ms\n";
    os << "  Redirects: " << (r.follow_redirects ? "yes" : "no");
    return os;
}

int main() {
    auto request = RequestBuilder{}
        .method("POST")
        .url("https://api.example.com/users")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer abc123")
        .body(R"({"name": "Alice", "age": 30})")
        .timeout(5000)
        .follow_redirects(false)
        .build();

    std::cout << request << "\n";

    // Minimal request - defaults apply
    auto get_req = RequestBuilder{}
        .method("GET")
        .url("https://api.example.com/health")
        .build();

    std::cout << "\n" << get_req << "\n";
}
// Expected output:
//   POST https://api.example.com/users
//     Content-Type: application/json
//     Authorization: Bearer abc123
//     Body: {"name": "Alice", "age": 30}
//     Timeout: 5000ms
//     Redirects: no
//
//   GET https://api.example.com/health
//     Timeout: 30000ms
//     Redirects: yes
```

Notice how readable the call site is. You can see exactly what each argument means, and `build()` catches missing required fields before anything goes wrong.

---

### Q2: Show how deducing-this (C++23) makes the builder pattern work with inheritance correctly

The deducing-this feature lets the compiler figure out the actual runtime type of `*this` and use that as the return type. This solves the type-loss problem from the overview without any of the CRTP boilerplate.

**Solution - Deducing This for Builder Inheritance:**

```cpp
#include <iostream>
#include <string>
#include <utility>

// C++23: deducing this solves the builder inheritance problem

struct Pizza {
    std::string size;
    std::string crust;
    bool extra_cheese = false;
    std::string topping;
};

// Base builder with deducing-this (C++23)
class PizzaBuilder {
protected:
    Pizza pizza_;

public:
    // this auto&& self: deduces the actual derived type!
    auto&& size(this auto&& self, std::string s) {
        self.pizza_.size = std::move(s);
        return std::forward<decltype(self)>(self);
    }

    auto&& crust(this auto&& self, std::string c) {
        self.pizza_.crust = std::move(c);
        return std::forward<decltype(self)>(self);
    }

    auto&& extra_cheese(this auto&& self, bool ec = true) {
        self.pizza_.extra_cheese = ec;
        return std::forward<decltype(self)>(self);
    }

    Pizza build() { return std::move(pizza_); }
};

// Derived builder adds more setters
class GourmetPizzaBuilder : public PizzaBuilder {
public:
    auto&& topping(this auto&& self, std::string t) {
        self.pizza_.topping = std::move(t);
        return std::forward<decltype(self)>(self);
    }
};

int main() {
    // With deducing-this, base methods return the ACTUAL derived type
    auto pizza = GourmetPizzaBuilder{}
        .size("Large")          // returns GourmetPizzaBuilder& (not PizzaBuilder&!)
        .crust("Thin")          // still GourmetPizzaBuilder&
        .extra_cheese()         // still GourmetPizzaBuilder&
        .topping("Truffle Oil") // works! type was preserved through base methods
        .build();

    std::cout << "Pizza: " << pizza.size << ", " << pizza.crust
              << ", cheese=" << pizza.extra_cheese
              << ", topping=" << pizza.topping << "\n";
}
// Expected output:
//   Pizza: Large, Thin, cheese=1, topping=Truffle Oil

// ======================================================
// For comparison, the C++11 CRTP approach:
// ======================================================
// template <typename Derived>
// class PizzaBuilderCRTP {
// protected:
//     Pizza pizza_;
//     Derived& self() { return static_cast<Derived&>(*this); }
// public:
//     Derived& size(std::string s) { pizza_.size = std::move(s); return self(); }
//     Derived& crust(std::string c) { pizza_.crust = std::move(c); return self(); }
// };
// class GourmetBuilder : public PizzaBuilderCRTP<GourmetBuilder> { ... };
//
// Drawback: CRTP requires the derived type as template parameter - verbose
// C++23 deducing-this avoids this entirely.
```

The key insight is `this auto&& self`: the compiler deduces `self` as `GourmetPizzaBuilder&` when the method is called on a `GourmetPizzaBuilder`, so the return type carries the derived type all the way through the chain.

---

### Q3: Compare named parameters (designated initializers) vs builder pattern for complex constructors

Before reaching for a Builder class, check whether C++20 designated initializers are enough for your situation. They're much simpler when validation isn't needed.

**Comparison:**

| Aspect | Designated Initializers (C++20) | Builder Pattern |
| --- | --- | --- |
| **Syntax** | `Type{.field = val}` | `Builder{}.field(val).build()` |
| **Validation** | None (direct construction) | `build()` can validate |
| **Defaults** | Built-in (unmentioned fields zero-init) | Must track manually |
| **Inheritance** | Not applicable (aggregates only) | Works with CRTP/deducing-this |
| **Complexity** | Simple - no extra class | Requires a Builder class |
| **Conditional building** | Awkward | Natural - `if(x) b.set_y(y)` |
| **IDE autocomplete** | Field names | Method names |

```cpp
#include <iostream>
#include <string>
#include <optional>

// ===== Approach 1: Designated Initializers (C++20) =====
struct ServerConfig {
    std::string host = "localhost";
    int port = 8080;
    int max_connections = 100;
    bool use_tls = false;
    std::string cert_path;
    int timeout_ms = 30000;
};

void use_designated_initializers() {
    // Clean, readable, no extra class needed
    ServerConfig cfg{
        .host = "api.example.com",
        .port = 443,
        .use_tls = true,
        .cert_path = "/etc/ssl/cert.pem",
        .timeout_ms = 5000
        // .max_connections uses default 100
    };
    std::cout << cfg.host << ":" << cfg.port << "\n";
}

// ===== Approach 2: Builder (when validation needed) =====
class ServerConfigBuilder {
    ServerConfig cfg_;
public:
    ServerConfigBuilder& host(std::string h) { cfg_.host = std::move(h); return *this; }
    ServerConfigBuilder& port(int p) { cfg_.port = p; return *this; }
    ServerConfigBuilder& tls(bool t, std::string cert = "") {
        cfg_.use_tls = t;
        cfg_.cert_path = std::move(cert);
        return *this;
    }

    ServerConfig build() {
        // Validation logic belongs here
        if (cfg_.use_tls && cfg_.cert_path.empty())
            throw std::logic_error("TLS enabled but no cert path");
        if (cfg_.port < 1 || cfg_.port > 65535)
            throw std::logic_error("Invalid port");
        return std::move(cfg_);
    }
};

int main() {
    use_designated_initializers();

    auto cfg = ServerConfigBuilder{}
        .host("api.example.com")
        .port(443)
        .tls(true, "/etc/ssl/cert.pem")
        .build();  // validates!

    std::cout << cfg.host << ":" << cfg.port << " tls=" << cfg.use_tls << "\n";
}
// Expected output:
//   api.example.com:443
//   api.example.com:443 tls=1
```

The bottom line is that designated initializers are the right choice for simple data structs with no cross-field rules. The moment you need validation, conditional logic, or a derived builder hierarchy, reach for the full Builder pattern.

**When to Use Which:**

```cpp
Use designated initializers when:
  // Simple aggregate with defaults
  // No validation needed
  // No inheritance hierarchy
  // All fields are independent

Use builder pattern when:
  // Complex validation (cross-field constraints)
  // Conditional building (add headers based on auth type)
  // Builder hierarchy (base builder + derived specializations)
  // Immutable product (fields private, set only via build)
  // Step-by-step construction with intermediate state
```

---

## Notes

- **Return by reference** (`Builder&`), not by value - chaining copies would be expensive and incorrect.
- **Move the product in `build()`** - `return std::move(product_)` transfers ownership efficiently.
- **Consider `[[nodiscard]]` on `build()`** - prevents accidentally ignoring the built object.
- **Thread safety:** Builders are typically not thread-safe - use one builder per thread.
- **C++23 deducing-this** eliminates the need for CRTP in builder hierarchies - much cleaner code.
- **Designated initializers require aggregates** - can't use with classes that have private members, constructors, or virtual functions.
