# Use aggregate initialization vs constructor initialization wisely

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

| Aspect | Constructor Init | Aggregate Init | Designated Init (C++20) |
| --- | :---: | :---: | :---: |
| Validation | **Yes** | No | No |
| Default values | In constructor | Default member init | Default member init |
| Self-documenting | Named args? No | No | **Yes** |
| Narrowing protection | Braces only | **Always** | **Always** |
| Class invariants | **Enforced** | Not enforced | Not enforced |

### When to Use Each

The choice boils down to whether your type needs to protect any invariants. If the answer is yes - if there is such a thing as an invalid state - use a constructor. If the type is just a bag of named fields with no rules between them, an aggregate is simpler and clearer.

```cpp
Aggregate init:     Simple data carriers, configs, POD-like structs
                    No invariants to protect

Constructor init:   Classes with invariants, validation, resource management
                    Need to ensure consistent state

Designated init:    Named parameters for aggregates (C++20)
                    Many optional fields with defaults
```

---

## Self-Assessment

### Q1: Show the different initialization styles and their trade-offs

Here are the three approaches side by side. `Endpoint` is a pure data carrier with no rules between its fields, so aggregate init is perfect. `PortRange` has a real invariant (`min <= max`) that must be checked, so it needs a constructor. `NetworkConfig` is a hybrid: it is an aggregate at heart, but a static factory method adds the validation layer without giving up the aggregate's flexibility.

```cpp
#include <string>
#include <stdexcept>
#include <iostream>

// AGGREGATE: simple data, no invariants
struct Endpoint {
    std::string host = "localhost";
    int port = 80;
    bool ssl = false;
};
// Great with designated initializers:
// Endpoint ep{.host = "api.com", .port = 443, .ssl = true};

// CONSTRUCTOR: has invariants to protect
class PortRange {
    int min_;
    int max_;
public:
    PortRange(int min_port, int max_port)
        : min_(min_port), max_(max_port) {
        if (min_port < 0 || max_port > 65535)
            throw std::out_of_range("Invalid port");
        if (min_port > max_port)
            throw std::invalid_argument("min > max");
    }
    int min() const { return min_; }
    int max() const { return max_; }
};

// HYBRID: aggregate + factory for validation
struct NetworkConfig {
    std::string host;
    int port;
    int timeout_ms = 5000;
    int max_retries = 3;

    // Static factory validates the aggregate
    static NetworkConfig validated(std::string host, int port,
                                   int timeout = 5000, int retries = 3) {
        if (port < 0 || port > 65535)
            throw std::out_of_range("Invalid port");
        if (timeout <= 0)
            throw std::invalid_argument("Timeout must be positive");
        return {std::move(host), port, timeout, retries};
    }
};

int main() {
    // Aggregate: fast, no overhead
    Endpoint ep{.port = 443, .ssl = true};

    // Constructor: safe, validated
    PortRange range(1024, 8080);
    // PortRange bad(8080, 1024);  // throws!

    // Hybrid: validated aggregate
    auto cfg = NetworkConfig::validated("api.com", 443);
    std::cout << cfg.host << ":" << cfg.port << "\n";
    return 0;
}
```

### Q2: Show narrowing and initialization gotchas

Brace initialization is generally safer than parenthesis initialization, but it comes with a few surprises of its own. The most important ones are listed here. The `vector{5, 1}` vs `vector(5, 1)` difference trips up nearly everyone at least once.

```cpp
#include <vector>
#include <string>
#include <iostream>

struct Size {
    int width;
    int height;
};

void demo_gotchas() {
    // GOTCHA 1: Narrowing in braces is an error
    // Size s{1.5, 2.5};  // ERROR: double -> int narrows
    Size s(1.5, 2.5);     // OK but truncates silently (parentheses)

    // GOTCHA 2: Most vexing parse
    // Widget w();  // Declares a FUNCTION, not an object!
    // Widget w{};  // Creates default-constructed object

    // GOTCHA 3: initializer_list vs constructor
    std::vector<int> v1(5, 1);   // 5 elements, value 1: {1,1,1,1,1}
    std::vector<int> v2{5, 1};   // 2 elements: {5, 1}
    std::cout << v1.size() << " vs " << v2.size() << "\n";  // 5 vs 2

    // GOTCHA 4: Empty braces
    struct Widget { int x = 0; };
    Widget w1;    // Default init: x MAY be uninitialized (if not defaulted)
    Widget w2{};  // Value init: x = 0 (guaranteed)
}

// GOTCHA 5: Aggregate vs non-aggregate changes between standards
struct MaybeAggregate {
    int x;
    MaybeAggregate() = default;
    // C++14: aggregate (= default is OK)
    // C++20: NOT aggregate (= default counts as user-declared!)
};

int main() {
    demo_gotchas();
    return 0;
}
```

### Q3: Design a config system using designated initializers

Designated initializers are one of the most practically useful C++20 features for API design. They give you named parameters - something C++ has historically lacked. Notice how the `start_app` call at the bottom reads almost like a configuration file: you see exactly which fields you are setting, the rest silently pick up their defaults, and the compiler will reject you if you try to set a field that does not exist.

```cpp
#include <string>
#include <optional>
#include <chrono>
#include <iostream>

struct DatabaseConfig {
    std::string host = "localhost";
    int port = 5432;
    std::string database = "test";
    std::string user = "postgres";
    std::optional<std::string> password;
    int pool_size = 10;
    std::chrono::seconds connect_timeout{5};
    std::chrono::seconds query_timeout{30};
    bool ssl = false;
    bool auto_reconnect = true;
};

struct LoggingConfig {
    enum Level { Trace, Debug, Info, Warn, Error };
    Level level = Info;
    std::string file = "/var/log/app.log";
    size_t max_size_mb = 100;
    int rotate_count = 5;
    bool console_output = true;
};

struct AppConfig {
    DatabaseConfig db;
    LoggingConfig logging;
    int worker_threads = 4;
    bool debug_mode = false;
};

void start_app(const AppConfig& config) {
    std::cout << "DB: " << config.db.user << "@" << config.db.host
              << ":" << config.db.port << "/" << config.db.database
              << " pool=" << config.db.pool_size
              << " ssl=" << config.db.ssl << "\n";
    std::cout << "Log: " << config.logging.file
              << " level=" << config.logging.level << "\n";
    std::cout << "Workers: " << config.worker_threads << "\n";
}

int main() {
    // Designated initializers: self-documenting, only set what you need
    start_app({
        .db = {
            .host = "db.production.com",
            .port = 5433,
            .database = "analytics",
            .user = "app_service",
            .password = "secret",
            .pool_size = 20,
            .ssl = true,
        },
        .logging = {
            .level = LoggingConfig::Warn,
            .max_size_mb = 500,
            .console_output = false,
        },
        .worker_threads = 8,
    });
    return 0;
}
```

---

## Notes

- Use **aggregates** for data carriers with no invariants (config structs, DTOs).
- Use **constructors** when the class has invariants that must be validated.
- C++20 **designated initializers** are the best alternative to named parameters - use them.
- Brace init `{}` catches narrowing conversions; parenthesis `()` does not.
- Watch for `initializer_list` hijacking constructors (the `vector{5, 1}` trap).
- The aggregate rules changed in C++20: `= default` constructors now disqualify a type from being aggregate.
