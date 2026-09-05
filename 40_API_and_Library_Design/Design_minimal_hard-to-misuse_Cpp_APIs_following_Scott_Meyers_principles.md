# Design minimal, hard-to-misuse C++ APIs following Scott Meyers' principles

**Category:** API and Library Design  
**Standard:** C++17  
**Reference:** <https://www.aristeia.com/Papers/IEEE_Software_JulAug_2004_revised.pdf>  

---

## Topic Overview

Scott Meyers' core principle is: **"Make interfaces easy to use correctly and hard to use incorrectly."** That deceptively simple sentence drives every decision discussed in this topic.

The key insight is that misuse should be a compile error whenever possible - not a runtime crash, not a code review comment, not documentation the user skips. The C++ type system is your enforcement mechanism. If a user has to work hard to misuse your API, they usually will not.

### Key Principles

Here is a summary of the techniques that follow from this principle. They all share the same goal: moving the detection of mistakes from runtime to compile time.

| Principle | Technique |
| --- | --- |
| Prevent misuse at compile time | Strong types, `[[nodiscard]]`, deleted overloads |
| Make defaults safe | Default-construct to valid state |
| Minimize API surface | Fewer functions = fewer mistakes |
| Express preconditions in types | Enums over bools, units in names |

### Strong Types to Prevent Parameter Confusion

One of the most common API bugs is passing arguments in the wrong order. When a function takes `(int port, int timeout_ms, bool use_ssl)`, the compiler accepts all six orderings of the two integers and both boolean values. That is a lot of silent bugs waiting to happen.

Strong types fix this by making each parameter a distinct type. The compiler rejects the wrong order before the code even runs:

```cpp
#include <chrono>
#include <string>
#include <iostream>

// BAD: easy to swap arguments
void connect_bad(const std::string& host, int port, int timeout_ms, bool use_ssl);
// connect_bad("localhost", 443, 1, true);  <- is 1 the port or timeout?

// GOOD: strong types prevent argument swapping
struct Host { std::string value; };
struct Port { uint16_t value; };
enum class UseTLS { No, Yes };

void connect_good(Host host, Port port,
                   std::chrono::milliseconds timeout,
                   UseTLS tls = UseTLS::No) {
    std::cout << "Connecting to " << host.value
              << ":" << port.value
              << " timeout=" << timeout.count() << "ms"
              << " TLS=" << (tls == UseTLS::Yes ? "yes" : "no") << "\n";
}

int main() {
    using namespace std::chrono_literals;
    connect_good(Host{"localhost"}, Port{443}, 5000ms, UseTLS::Yes);
    // Compile error if you swap Host and Port!
}
```

Notice that `UseTLS` is an enum class rather than a `bool`. This makes the call site self-documenting - `UseTLS::Yes` tells you what you are enabling, while `true` tells you nothing without reading the declaration.

### [[nodiscard]] to Prevent Ignored Returns

When a function returns an error code or a handle, forgetting to check the return value is a common bug. `[[nodiscard]]` turns that oversight into a compiler warning, which becomes a build error with `-Werror`.

```cpp
#include <system_error>
#include <string>

// BAD: caller can ignore the error
std::error_code remove_file_bad(const std::string& path);

// GOOD: compiler warns if error is ignored
[[nodiscard]] std::error_code remove_file(const std::string& path);

// Even better: nodiscard on the TYPE
struct [[nodiscard("Check the error code")]] Result {
    std::error_code ec;
    explicit operator bool() const { return !ec; }
};

Result create_directory(const std::string& path);

void example() {
    create_directory("/tmp/test"); // Warning: nodiscard
    if (auto r = create_directory("/tmp/test"); !r) {
        // Handle error
    }
}
```

Putting `[[nodiscard]]` on the type itself (as shown on `Result`) is stronger than putting it on the function, because it applies to every function that returns that type - you only have to remember to annotate it once.

### Deleted Overloads to Block Dangerous Conversions

C++ will silently convert between numeric types unless you stop it. A `Buffer(int)` constructor accidentally created from a negative `int` is a classic source of subtle bugs. Deleting the dangerous overloads turns those silent conversions into compile errors:

```cpp
#include <cstddef>
#include <cstdint>

class Buffer {
public:
    explicit Buffer(size_t size);
    // Block implicit conversions that could be bugs
    Buffer(int) = delete;      // Prevent negative sizes
    Buffer(double) = delete;   // Prevent fractional sizes
    Buffer(nullptr_t) = delete; // Prevent null construction
};
```

The user sees a clear "use of deleted function" error and knows they need to either use `size_t` explicitly or reconsider what they are passing. This is better than a runtime crash when `size` silently wraps around from `-1` to `SIZE_MAX`.

---

## Self-Assessment

### Q1: Redesign a function with bool parameters to be self-documenting

A function call like `render(scene, true, false, true)` communicates nothing about what the three booleans control. Replace each bool with a named enum and the meaning becomes obvious at every call site:

```cpp
// BEFORE: What do the bools mean?
// render(scene, true, false, true);

enum class Antialiasing { Off, On };
enum class VSync { Off, On };
enum class Fullscreen { Windowed, Fullscreen };

void render(const Scene& scene,
            Antialiasing aa = Antialiasing::On,
            VSync vsync = VSync::On,
            Fullscreen fs = Fullscreen::Windowed);

// AFTER: self-documenting at call site
// render(scene, Antialiasing::On, VSync::Off, Fullscreen::Fullscreen);
```

The caller no longer needs to look up the declaration to know what the arguments mean, and swapping two of them is now a type error instead of a silent bug.

### Q2: Show how to use factory functions instead of multi-argument constructors

When a constructor takes two parameters that look similar (like `duration` and a `bool`), it is easy to get them mixed up or misread. Named factory functions (static methods) express intent at the call site and are much harder to misuse:

```cpp
#include <chrono>
#include <memory>

class Timer {
    std::chrono::milliseconds duration_;
    bool repeating_;
    Timer(std::chrono::milliseconds d, bool r) : duration_(d), repeating_(r) {}
public:
    static Timer one_shot(std::chrono::milliseconds d) { return {d, false}; }
    static Timer repeating(std::chrono::milliseconds d) { return {d, true}; }
    // Named constructors are clearer than Timer(5000ms, true)
};

auto t = Timer::repeating(std::chrono::seconds{5});
```

`Timer::repeating(5s)` is unambiguous. `Timer(5s, true)` requires the reader to remember which argument is which.

### Q3: Design a builder for complex configuration

When a type has many optional settings, a builder pattern with method chaining gives users a safe, readable way to configure it. Every method returns `*this`, and the defaults are all safe values so you can call `build()` with zero configuration and get something sensible:

```cpp
class ServerConfig {
    std::string host_ = "0.0.0.0";
    uint16_t port_ = 8080;
    int max_connections_ = 1000;
    bool tls_ = false;

public:
    ServerConfig& host(std::string h) { host_ = std::move(h); return *this; }
    ServerConfig& port(uint16_t p) { port_ = p; return *this; }
    ServerConfig& max_connections(int n) { max_connections_ = n; return *this; }
    ServerConfig& enable_tls() { tls_ = true; return *this; }

    // All defaults are safe - you can build with zero configuration
};

auto cfg = ServerConfig{}.host("example.com").port(443).enable_tls();
```

Notice that `enable_tls()` takes no arguments - there is no `tls(bool)` setter that a user could call with `false` and be confused about the default. The method name carries the intent.

---

## Notes

- **Prefer enums over bools** for function parameters - the enum value names the intent at the call site.
- **Use `[[nodiscard]]`** on any function whose return value must be checked (error codes, handles, resources).
- **Delete dangerous overloads** to catch implicit conversion bugs at compile time rather than runtime.
- Named factory functions (static methods) are clearer than overloaded constructors when the constructor arguments have similar types.
- Follow the "principle of least surprise" - API should behave as users expect, and what users expect is usually informed by the names you give things.
