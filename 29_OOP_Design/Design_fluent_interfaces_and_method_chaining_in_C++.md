# Design fluent interfaces and method chaining in C++

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

**Fluent interfaces** let you write configuration and setup code that reads almost like a sentence by returning `*this` (or a new object) from each method. Instead of calling five separate setters on five separate lines, you chain them together in one readable expression. You've almost certainly seen this pattern in query builders, test assertion libraries, and configuration APIs.

Here's what it looks like in practice - notice how each method returns something chainable, so the whole thing flows left to right:

```cpp
auto query = SelectQuery()
    .from("users")          // returns *this
    .where("age > 18")      // returns *this
    .order_by("name")       // returns *this
    .limit(10);             // returns *this
```

The table below shows your three main design choices. The return type is what determines the flavor of the interface:

| Design Choice | Return Type | Use Case |
| --- | --- | --- |
| Mutable chaining | `T&` | Builder, configuration |
| Immutable chaining | `T` (new copy) | Functional, thread-safe |
| Named parameter idiom | `T&` | Simulating keyword args |

---

## Self-Assessment

### Q1: Implement a fluent configuration builder

This is the most common form - mutable chaining where each setter returns `*this`. The `ServerConfig` class sets sensible defaults in member initializers and lets you override only what you care about, in any order. The whole point is that the call site reads naturally from top to bottom, like a description of what you want.

```cpp
#include <string>
#include <vector>
#include <chrono>
#include <iostream>

class ServerConfig {
public:
    // Each setter returns *this for chaining
    ServerConfig& host(std::string h) { host_ = std::move(h); return *this; }
    ServerConfig& port(int p) { port_ = p; return *this; }
    ServerConfig& threads(int n) { threads_ = n; return *this; }
    ServerConfig& timeout(std::chrono::seconds t) { timeout_ = t; return *this; }
    ServerConfig& enable_ssl(bool v = true) { ssl_ = v; return *this; }
    ServerConfig& add_route(std::string path) {
        routes_.push_back(std::move(path));
        return *this;
    }

    void print() const {
        std::cout << host_ << ":" << port_
                  << " threads=" << threads_
                  << " ssl=" << ssl_
                  << " routes=" << routes_.size() << "\n";
    }

private:
    std::string host_ = "0.0.0.0";
    int port_ = 8080;
    int threads_ = 4;
    std::chrono::seconds timeout_{30};
    bool ssl_ = false;
    std::vector<std::string> routes_;
};

int main() {
    ServerConfig()
        .host("api.example.com")
        .port(443)
        .enable_ssl()
        .threads(8)
        .timeout(std::chrono::seconds{60})
        .add_route("/api/v1")
        .add_route("/health")
        .print();
    return 0;
}
```

The `enable_ssl()` method with a default `true` parameter is a nice touch - it reads naturally at the call site without needing to write `.enable_ssl(true)`. This kind of attention to call-site readability is the whole point of fluent design.

### Q2: Show fluent interface with inheritance (CRTP for proper return type)

Here's the tricky part. If `QueryBase` has a fluent `from()` method that returns `*this`, what type does that return? It returns `QueryBase&` - and if you call `.columns()` on a `QueryBase&`, the compiler complains because `columns()` only exists on `SelectQuery`. The fix is CRTP: make the base class return `Derived&` instead of its own type.

The reason this trips people up is that `return *this` in a templated base class doesn't magically know the derived type - you have to explicitly cast with `static_cast<Derived&>(*this)`. The `self()` helper method wraps that cast cleanly so you don't repeat it in every method.

```cpp
#include <string>
#include <iostream>

// CRTP base: fluent methods return Derived&, not Base&
template<typename Derived>
class QueryBase {
public:
    Derived& from(std::string table) {
        table_ = std::move(table);
        return self();
    }
    Derived& where(std::string cond) {
        where_ = std::move(cond);
        return self();
    }
    Derived& limit(int n) {
        limit_ = n;
        return self();
    }
protected:
    Derived& self() { return static_cast<Derived&>(*this); }
    std::string table_, where_;
    int limit_ = 0;
};

class SelectQuery : public QueryBase<SelectQuery> {
public:
    SelectQuery& columns(std::string cols) {
        cols_ = std::move(cols);
        return *this;
    }
    std::string build() const {
        return "SELECT " + cols_ + " FROM " + table_
             + (where_.empty() ? "" : " WHERE " + where_)
             + (limit_ > 0 ? " LIMIT " + std::to_string(limit_) : "");
    }
private:
    std::string cols_ = "*";
};

class DeleteQuery : public QueryBase<DeleteQuery> {
public:
    std::string build() const {
        return "DELETE FROM " + table_
             + (where_.empty() ? "" : " WHERE " + where_);
    }
};

int main() {
    // Chaining works correctly even through base methods
    auto sql = SelectQuery()
        .columns("id, name")    // SelectQuery&
        .from("users")          // SelectQuery& (not QueryBase&!)
        .where("active = true") // SelectQuery&
        .limit(10)
        .build();
    std::cout << sql << "\n";

    auto del = DeleteQuery()
        .from("sessions")
        .where("expired = true")
        .build();
    std::cout << del << "\n";
    return 0;
}
```

Without CRTP, calling `.from()` on a `SelectQuery` would give you back a `QueryBase<SelectQuery>&`, and you'd lose access to `.columns()`. With CRTP, `.from()` returns a `SelectQuery&`, keeping all methods available throughout the chain.

### Q3: Implement an immutable fluent interface (functional style)

The mutable style modifies `*this` in place, which means you can't safely reuse a partially-built configuration as a base for two different final configurations. The immutable style fixes this: each method returns a new copy of the object with that one field changed, leaving the original untouched. This is the functional programming style applied to C++ builders.

```cpp
#include <string>
#include <vector>
#include <iostream>

class Pipeline {
    std::vector<std::string> stages_;
    int parallelism_ = 1;
    bool verbose_ = false;

public:
    // Each method returns a NEW Pipeline (immutable)
    [[nodiscard]] Pipeline add_stage(std::string name) const {
        Pipeline p = *this;
        p.stages_.push_back(std::move(name));
        return p;
    }

    [[nodiscard]] Pipeline with_parallelism(int n) const {
        Pipeline p = *this;
        p.parallelism_ = n;
        return p;
    }

    [[nodiscard]] Pipeline verbose(bool v = true) const {
        Pipeline p = *this;
        p.verbose_ = v;
        return p;
    }

    void execute() const {
        std::cout << "Pipeline (" << parallelism_ << " threads";
        if (verbose_) std::cout << ", verbose";
        std::cout << "): ";
        for (const auto& s : stages_) std::cout << s << " -> ";
        std::cout << "done\n";
    }
};

int main() {
    Pipeline base;  // Reusable base config
    auto dev = base.verbose().with_parallelism(1);
    auto prod = base.with_parallelism(8);

    dev.add_stage("parse").add_stage("transform").execute();
    prod.add_stage("parse").add_stage("validate").add_stage("store").execute();
    // base is unchanged!
    return 0;
}
```

The `[[nodiscard]]` attribute is important here - without it, a caller could accidentally write `pipeline.add_stage("parse");` (forgetting to capture the result) and silently lose the modification. The attribute turns that mistake into a compiler warning.

---

## Notes

- Return `T&` (reference) for mutable chaining, `T` (value) for immutable chaining.
- Mark immutable-style methods `[[nodiscard]]` to prevent accidentally discarding the result.
- Use CRTP when fluent base classes need to return the derived type.
- Fluent interfaces are great for: builders, queries, configuration, pipeline setup.
- Avoid deep chains in production code - they make debugging harder (breakpoints, stack traces).
- Consider named parameters (C++20 designated initializers) as an alternative for simple cases.
