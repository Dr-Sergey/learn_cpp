# Implement the Builder pattern for complex object construction in C++

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

The **Builder pattern** separates the construction of a complex object from its final representation. The idea is simple: instead of one enormous constructor call that passes twelve arguments in the right order and hopes for the best, you build the object step by step using a dedicated Builder object that collects the pieces, validates them, and hands you back a finished product. The result is code at the call site that reads almost like a configuration file.

Here's when you should reach for a Builder:

| Use Builder When | Example |
| --- | --- |
| Many constructor parameters | Network connection config |
| Some params optional, some required | HTTP request builder |
| Construction has validation rules | SQL query builder |
| Object should be immutable after construction | Config objects |

### Builder vs Aggregate Init vs Constructor

To put Builder in perspective, here are the three main options for constructing complex objects:

```cpp
Constructor:      Widget(a, b, c, d, e)       // Easy to swap args
Aggregate init:   Widget{.x=1, .y=2}          // C++20, limited validation
Builder:          Widget::Builder().x(1).y(2).build()  // Validation + readability
```

The constructor collapses all information into a single dense call - easy to get arguments in the wrong order, and impossible to add validation without making it messy. Aggregate initialization with named members (C++20 designated initializers) is very readable but doesn't let you run validation logic before the object is created. The Builder gives you both readability and the chance to reject bad combinations before the object is born.

---

## Self-Assessment

### Q1: Implement a fluent Builder for an HTTP request

A fluent builder returns `*this` from every setter, so you can chain calls in a natural left-to-right reading order. Validation happens at `build()` time, not scattered across constructors. Notice that `HttpRequest`'s constructor is `private` - the only way to get one is through the Builder, which means malformed `HttpRequest` objects literally cannot be constructed.

**Answer:**

```cpp
#include <string>
#include <map>
#include <optional>
#include <stdexcept>
#include <iostream>

class HttpRequest {
public:
    class Builder;  // Forward declare

    // Getters
    const std::string& method() const { return method_; }
    const std::string& url() const { return url_; }
    const std::string& body() const { return body_; }
    const std::map<std::string, std::string>& headers() const { return headers_; }
    int timeout_ms() const { return timeout_ms_; }

private:
    HttpRequest() = default;  // Only Builder can construct
    friend class Builder;

    std::string method_;
    std::string url_;
    std::string body_;
    std::map<std::string, std::string> headers_;
    int timeout_ms_ = 30000;
};

class HttpRequest::Builder {
public:
    Builder& method(std::string m) { req_.method_ = std::move(m); return *this; }
    Builder& url(std::string u)    { req_.url_ = std::move(u); return *this; }
    Builder& body(std::string b)   { req_.body_ = std::move(b); return *this; }
    Builder& timeout(int ms)       { req_.timeout_ms_ = ms; return *this; }

    Builder& header(std::string key, std::string val) {
        req_.headers_[std::move(key)] = std::move(val);
        return *this;
    }

    // Validation at build time
    [[nodiscard]] HttpRequest build() {
        if (req_.url_.empty())
            throw std::invalid_argument("URL is required");
        if (req_.method_.empty())
            req_.method_ = "GET";
        if (req_.method_ == "POST" && req_.body_.empty())
            throw std::invalid_argument("POST requires a body");

        return std::move(req_);
    }

private:
    HttpRequest req_;
};

int main() {
    auto req = HttpRequest::Builder()
        .method("POST")
        .url("https://api.example.com/data")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer token123")
        .body(R"({"key": "value"})")
        .timeout(5000)
        .build();

    std::cout << req.method() << " " << req.url() << "\n";
    return 0;
}
```

The `[[nodiscard]]` on `build()` means the compiler will warn you if you call `build()` and throw away the result - a common accident when you forget to assign it. The `std::move(req_)` at the end transfers ownership out of the builder efficiently, so you pay no unnecessary copy cost for the finished object.

### Q2: Show a step builder that enforces construction order at compile time

The fluent builder in Q1 is flexible, but nothing stops you from calling `.build()` before you've set required fields. A step builder solves this by making each step return a *different type* - and each type only exposes the methods that are valid at that stage. If you skip a step, the next step's type simply won't be in scope. The reason this works is that the type system itself is doing the enforcement - there's no runtime check, no exception, just a type that doesn't have the method you tried to call.

**Answer:**

```cpp
#include <string>
#include <iostream>

class Connection {
public:
    std::string host;
    int port;
    std::string user;
    std::string password;
    bool use_ssl;
};

// Step Builder: each step returns a different type
// Enforces: host -> port -> credentials -> build
class ConnFinal {
    Connection c_;
public:
    explicit ConnFinal(Connection c) : c_(std::move(c)) {}
    ConnFinal& ssl(bool v) { c_.use_ssl = v; return *this; }
    Connection build() { return std::move(c_); }
};

class ConnCreds {
    Connection c_;
public:
    explicit ConnCreds(Connection c) : c_(std::move(c)) {}
    ConnFinal credentials(std::string user, std::string pass) {
        c_.user = std::move(user);
        c_.password = std::move(pass);
        return ConnFinal(std::move(c_));
    }
};

class ConnPort {
    Connection c_;
public:
    explicit ConnPort(Connection c) : c_(std::move(c)) {}
    ConnCreds port(int p) {
        c_.port = p;
        return ConnCreds(std::move(c_));
    }
};

class ConnBuilder {
public:
    static ConnPort host(std::string h) {
        Connection c{};
        c.host = std::move(h);
        return ConnPort(std::move(c));
    }
};

int main() {
    // Must follow exact order: host -> port -> credentials -> build
    auto conn = ConnBuilder::host("db.example.com")
        .port(5432)
        .credentials("admin", "secret")
        .ssl(true)
        .build();
    // ConnBuilder::host("x").build();  // WON'T COMPILE!
    std::cout << conn.host << ":" << conn.port << "\n";
    return 0;
}
```

The reason `ConnBuilder::host("x").build()` won't compile is that `host()` returns a `ConnPort`, and `ConnPort` has no `build()` method - only `port()`. The type system itself enforces the construction protocol. You literally cannot misuse the API because the wrong call won't type-check. The trade-off is more boilerplate: you need a separate class for each stage, which adds up quickly for complex protocols.

### Q3: Implement a type-safe SQL query builder

A SQL builder is a classic Builder use case: you want to accumulate clauses in a natural order, validate that the required parts (like `FROM`) are present, and produce a finished query string at the end. Each method name mirrors the SQL keyword it contributes, which makes call sites read almost exactly like SQL itself.

**Answer:**

```cpp
#include <string>
#include <vector>
#include <sstream>
#include <iostream>

class SelectQuery {
public:
    SelectQuery& from(std::string table) {
        table_ = std::move(table);
        return *this;
    }
    SelectQuery& columns(std::initializer_list<std::string> cols) {
        columns_.assign(cols);
        return *this;
    }
    SelectQuery& where(std::string condition) {
        conditions_.push_back(std::move(condition));
        return *this;
    }
    SelectQuery& order_by(std::string col, bool desc = false) {
        order_ = std::move(col) + (desc ? " DESC" : " ASC");
        return *this;
    }
    SelectQuery& limit(int n) {
        limit_ = n;
        return *this;
    }

    [[nodiscard]] std::string build() const {
        std::ostringstream ss;
        ss << "SELECT ";
        if (columns_.empty()) ss << "*";
        else {
            for (size_t i = 0; i < columns_.size(); ++i) {
                if (i > 0) ss << ", ";
                ss << columns_[i];
            }
        }
        ss << " FROM " << table_;
        for (size_t i = 0; i < conditions_.size(); ++i)
            ss << (i == 0 ? " WHERE " : " AND ") << conditions_[i];
        if (!order_.empty()) ss << " ORDER BY " << order_;
        if (limit_ > 0) ss << " LIMIT " << limit_;
        return ss.str();
    }

private:
    std::string table_;
    std::vector<std::string> columns_;
    std::vector<std::string> conditions_;
    std::string order_;
    int limit_ = 0;
};

int main() {
    auto sql = SelectQuery()
        .from("users")
        .columns({"id", "name", "email"})
        .where("active = true")
        .where("role = 'admin'")
        .order_by("name")
        .limit(10)
        .build();

    std::cout << sql << "\n";
    // SELECT id, name, email FROM users WHERE active = true AND role = 'admin' ORDER BY name ASC LIMIT 10
    return 0;
}
```

Each call to `where()` appends another condition, and `build()` joins them with `AND`. The builder doesn't care whether you call `where()` once or five times - it just accumulates. This is much cleaner than trying to handle optional conditions in a constructor, and the resulting call site is readable even to someone who doesn't know C++ particularly well.

---

## Notes

- Reach for Builder when you have five or more constructor parameters, or when some combinations of parameters are invalid and need to be caught before construction completes.
- The fluent style (`builder.x(1).y(2).build()`) maximizes readability - the call site reads almost like a configuration file.
- Step builders enforce parameter ordering at compile time, making it impossible to misuse the API - but they come with more boilerplate to write.
- Make the target class's constructor `private` and declare Builder as a `friend` so the only valid construction path is through the Builder.
- C++20 designated initializers (`{.x=1, .y=2}`) can replace builders for simple aggregate types where no validation is needed.
- Always return `*this` by reference for chaining, and mark `build()` as `[[nodiscard]]` so the compiler warns if the result is accidentally discarded.
