# Use the type-safe builder pattern with designated initializers

**Category:** Best Practices and Idioms  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/aggregate_initialization>  

---

## Topic Overview

**Designated initializers** (C++20) let you name each field when you initialize an aggregate. This gives you the ergonomics of named parameters - self-documenting call sites, easy defaults for unspecified fields - without writing a builder class. Any field you leave out is zero-initialized (or uses its default member initializer if it has one).

```cpp
struct Config {
    int timeout = 30;
    int retries = 3;
    bool verbose = false;
};

auto cfg = Config{.timeout = 60, .verbose = true};  // retries = 3 (default)
```

It's a small syntactic feature, but it completely changes how readable configuration-style code looks at the call site.

---

## Self-Assessment

### Q1: Designated initializers vs positional constructors

The problem with positional constructors is that the reader has to count arguments and cross-reference the declaration to understand what each value means. That's not a theoretical concern - it's a real bug source when someone passes two booleans in the wrong order and the compiler can't tell.

```cpp
#include <iostream>
#include <string>

struct ServerConfig {
    std::string host = "localhost";
    int port = 8080;
    int max_connections = 100;
    int timeout_ms = 5000;
    bool use_tls = false;
    bool verbose = false;
};

// BAD: positional constructor - what does each number mean?
void start_server_bad(std::string host, int port, int max_conn,
                      int timeout, bool tls, bool verbose) {
    std::cout << "[BAD] " << host << ":" << port << '\n';
}

// GOOD: designated initializers - self-documenting!
void start_server(const ServerConfig& cfg) {
    std::cout << "Server: " << cfg.host << ":" << cfg.port
              << " (max=" << cfg.max_connections
              << ", timeout=" << cfg.timeout_ms << "ms"
              << ", tls=" << cfg.use_tls
              << ", verbose=" << cfg.verbose << ")\n";
}

int main() {
    // BAD: which bool is TLS? which is verbose?
    start_server_bad("example.com", 443, 200, 3000, true, false);

    // GOOD: crystal clear
    start_server({
        .host = "example.com",
        .port = 443,
        .max_connections = 200,
        .timeout_ms = 3000,
        .use_tls = true  // verbose defaults to false
    });

    // Minimal config: only override what matters
    start_server({.port = 9090, .verbose = true});
}
// Expected output:
// [BAD] example.com:443
// Server: example.com:443 (max=200, timeout=3000ms, tls=1, verbose=0)
// Server: localhost:9090 (max=100, timeout=5000ms, tls=0, verbose=1)
```

The third call is the most telling: `{.port = 9090, .verbose = true}` leaves four fields at their defaults without the reader needing to know what those defaults are or count positional slots.

### Q2: Named-parameter ergonomics without a builder class

The classic builder pattern requires a whole class with a setter for each field and a terminating method. That's a lot of boilerplate for something whose only job is to name the parameters. Designated initializers give you the same call-site clarity for free.

```cpp
#include <iostream>
#include <string>

// Traditional Builder pattern (lots of boilerplate)
class HttpRequestBuilder {
    std::string url_;
    std::string method_ = "GET";
    int timeout_ = 30;
public:
    HttpRequestBuilder& url(std::string u) { url_ = std::move(u); return *this; }
    HttpRequestBuilder& method(std::string m) { method_ = std::move(m); return *this; }
    HttpRequestBuilder& timeout(int t) { timeout_ = t; return *this; }
    void send() const {
        std::cout << method_ << " " << url_ << " (timeout=" << timeout_ << ")\n";
    }
};

// C++20: designated initializers = zero-cost "builder"
struct HttpRequest {
    std::string url;
    std::string method = "GET";
    int timeout = 30;
    bool follow_redirects = true;
};

void send(const HttpRequest& req) {
    std::cout << req.method << " " << req.url
              << " (timeout=" << req.timeout
              << ", redirect=" << req.follow_redirects << ")\n";
}

int main() {
    // Builder pattern: verbose
    HttpRequestBuilder()
        .url("https://api.example.com")
        .method("POST")
        .timeout(10)
        .send();

    // Designated initializers: clean and simple
    send({
        .url = "https://api.example.com",
        .method = "POST",
        .timeout = 10
    });

    // Minimal:
    send({.url = "https://example.com"});
}
// Expected output:
// POST https://api.example.com (timeout=10)
// POST https://api.example.com (timeout=10, redirect=1)
// GET https://example.com (timeout=30, redirect=1)
```

The builder version needs a separate class, `return *this` chains, and a terminating `.send()`. The designated-initializer version just needs a struct. If you don't need validation logic in the construction step, the struct approach wins on every dimension.

### Q3: Unspecified fields get zero/default-initialized

This is worth testing concretely because people sometimes worry that unspecified fields contain garbage. They don't - they get either the default member initializer if there is one, or zero-initialization otherwise.

```cpp
#include <iostream>
#include <string>

struct Particle {
    double x = 0.0;
    double y = 0.0;
    double z = 0.0;
    double vx = 0.0;
    double vy = 0.0;
    double vz = 0.0;
    double mass = 1.0;
    int id = -1;
};

struct RawData {
    int values[4];    // no default: zero-initialized
    char name[16];    // zero-initialized
    double factor;    // zero-initialized to 0.0
};

int main() {
    // Only set what you care about; rest gets defaults
    Particle p{.x = 1.0, .vy = 2.5, .mass = 0.5};
    std::cout << "Particle: pos=(" << p.x << "," << p.y << "," << p.z << ")"
              << " vel=(" << p.vx << "," << p.vy << "," << p.vz << ")"
              << " mass=" << p.mass << " id=" << p.id << '\n';

    // Aggregate without defaults: zero-initialized
    RawData d{.values = {10, 20}};  // values[2]=0, values[3]=0
    std::cout << "values: ";
    for (int v : d.values) std::cout << v << ' ';
    std::cout << "\nfactor: " << d.factor << '\n';

    // Empty init: ALL fields get defaults/zero
    Particle zero{};
    std::cout << "Zero mass: " << zero.mass << '\n';  // 1.0 (has default)
    std::cout << "Zero x: " << zero.x << '\n';         // 0.0 (default)
}
// Expected output:
// Particle: pos=(1,0,0) vel=(0,2.5,0) mass=0.5 id=-1
// values: 10 20 0 0
// factor: 0
// Zero mass: 1
// Zero x: 0
```

Notice `id=-1` even though we didn't specify it - that comes from the default member initializer `int id = -1`. Fields without a default member initializer, like `RawData::factor`, get zero-initialized instead.

---

## Notes

- Designated initializers require C++20 and only work with **aggregates** (no private members, no user constructors).
- Fields must be initialized in declaration order (C++ requirement, unlike C).
- Cannot mix positional and designated initializers.
- For validation, add a factory function that takes the aggregate and validates.
- Nested designated initializers: `{{.x = 1}, .name = "test"}`.
