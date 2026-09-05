# Implement the Decorator pattern using composition and interface forwarding

**Category:** Modern OOP Patterns  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Decorator_pattern>  

---

## Topic Overview

The **Decorator pattern** adds behavior to an object *at runtime* by wrapping it in another object that implements the same interface. The wrapper *forwards* all calls to the wrapped object, adding behavior before/after the forwarded call.

The key insight is "wraps it in another object that implements the same interface". Because the decorator looks identical to the thing it wraps, callers can't tell whether they're talking to the real object or a decorator. This lets you stack decorators arbitrarily, and callers never need to change.

### Decorator vs Inheritance

This is the fundamental contrast. Inheritance locks in the combination at compile time and leads to a combinatorial explosion of subclasses. Decorators compose at runtime in any order:

```cpp
Inheritance (static, compile-time):
  Logger <- TimestampLogger <- FilteredTimestampLogger <- ...
  -> combinatorial explosion of subclasses

Decorator (dynamic, runtime - compose at will):
  Logger <- TimestampDecorator(Logger)
           +-- FilterDecorator(Logger)
                +-- FileDecorator(Logger)

  auto log = FileDecorator(
               FilterDecorator(
                 TimestampDecorator(
                   ConsoleLogger())));
  // Chainable in any order!
```

### Structure

Here's how the pieces relate. `LoggerDecorator` is the important middle layer - it holds the wrapped object and provides default forwarding:

```cpp
+──────────────────────+
│    ILogger (interface)│
│  + log(msg)           │
+──────+───────────────+
       │
  +────+────────────────+
  │                     │
  v                     v
ConsoleLogger      LoggerDecorator
  log(msg) {         ILogger& wrapped_;
    print(msg);      log(msg) {
  }                    // add behavior
                       wrapped_.log(msg);
                     }
                       │
              +────────+──────────+
              v                   v
     TimestampDecorator    FilterDecorator
```

---

## Self-Assessment

### Q1: Wrap a Logger interface with a TimestampDecorator that prepends timestamps without modifying Logger

The important design choice here is `LoggerDecorator` as a base class for all decorators. It forwards the call by default, so a concrete decorator only needs to override the methods it actually cares about.

**Solution:**

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <chrono>
#include <ctime>
#include <iomanip>
#include <sstream>

// Interface
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& message) = 0;
};

// Concrete component
class ConsoleLogger : public ILogger {
public:
    void log(const std::string& message) override {
        std::cout << message << "\n";
    }
};

// Base decorator - forwards everything by default
class LoggerDecorator : public ILogger {
protected:
    std::unique_ptr<ILogger> wrapped_;
public:
    explicit LoggerDecorator(std::unique_ptr<ILogger> logger)
        : wrapped_(std::move(logger)) {}

    void log(const std::string& message) override {
        wrapped_->log(message);  // forward to wrapped object
    }
};

// Concrete decorator: prepends timestamp
class TimestampDecorator : public LoggerDecorator {
    std::string timestamp() const {
        auto now = std::chrono::system_clock::now();
        auto time = std::chrono::system_clock::to_time_t(now);
        std::ostringstream oss;
        oss << std::put_time(std::localtime(&time), "%H:%M:%S");
        return oss.str();
    }

public:
    using LoggerDecorator::LoggerDecorator;

    void log(const std::string& message) override {
        // Add behavior BEFORE forwarding
        std::string decorated = "[" + timestamp() + "] " + message;
        wrapped_->log(decorated);
    }
};

int main() {
    // Plain logger
    auto plain = std::make_unique<ConsoleLogger>();
    plain->log("Hello from plain logger");

    // Decorated with timestamp - ConsoleLogger unchanged!
    auto timestamped = std::make_unique<TimestampDecorator>(
        std::make_unique<ConsoleLogger>()
    );
    timestamped->log("Hello from timestamped logger");
}
// Expected output:
//   Hello from plain logger
//   [14:30:45] Hello from timestamped logger
```

Notice that `ConsoleLogger` was never touched. The timestamp behavior lives entirely in `TimestampDecorator`, which wraps a `ConsoleLogger` via ownership. This is the open/closed principle in action - open for extension, closed for modification.

---

### Q2: Chain multiple decorators: Logger -> Timestamp -> Filter -> LogToFile

Each decorator in the chain only does one thing. When you stack them, the result is a pipeline where each layer processes the message before passing it along. The order of wrapping determines the order of processing.

**Solution:**

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <fstream>
#include <algorithm>
#include <vector>

class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& message) = 0;
};

class ConsoleLogger : public ILogger {
public:
    void log(const std::string& message) override {
        std::cout << "[CONSOLE] " << message << "\n";
    }
};

class LoggerDecorator : public ILogger {
protected:
    std::unique_ptr<ILogger> wrapped_;
public:
    explicit LoggerDecorator(std::unique_ptr<ILogger> l) : wrapped_(std::move(l)) {}
    void log(const std::string& message) override { wrapped_->log(message); }
};

// Decorator 1: Timestamp
class TimestampDecorator : public LoggerDecorator {
public:
    using LoggerDecorator::LoggerDecorator;
    void log(const std::string& message) override {
        wrapped_->log("[2024-01-15 14:30] " + message);
    }
};

// Decorator 2: Severity filter - drops messages below threshold
class FilterDecorator : public LoggerDecorator {
    std::string min_prefix_;  // only pass messages starting with this
public:
    FilterDecorator(std::unique_ptr<ILogger> l, std::string prefix)
        : LoggerDecorator(std::move(l)), min_prefix_(std::move(prefix)) {}

    void log(const std::string& message) override {
        if (message.find(min_prefix_) != std::string::npos) {
            wrapped_->log(message);
        }
        // else: silently drop (filtered out)
    }
};

// Decorator 3: Log to file (in addition to forwarding)
class FileDecorator : public LoggerDecorator {
    std::string filename_;
public:
    FileDecorator(std::unique_ptr<ILogger> l, std::string fname)
        : LoggerDecorator(std::move(l)), filename_(std::move(fname)) {}

    void log(const std::string& message) override {
        // Write to file
        std::ofstream file(filename_, std::ios::app);
        if (file) file << message << "\n";
        // Also forward to next decorator/logger
        wrapped_->log(message);
    }
};

int main() {
    // Chain: ConsoleLogger -> Timestamp -> Filter("ERROR") -> File
    auto logger = std::make_unique<FileDecorator>(
        std::make_unique<FilterDecorator>(
            std::make_unique<TimestampDecorator>(
                std::make_unique<ConsoleLogger>()
            ),
            "ERROR"  // only pass messages containing "ERROR"
        ),
        "app.log"
    );

    // Processing order: File -> Filter -> Timestamp -> Console
    logger->log("ERROR: disk full");      // passes filter, written to file AND console
    logger->log("INFO: user logged in");  // filtered out (no "ERROR")
    logger->log("ERROR: timeout");        // passes filter
}
// Expected output (console):
//   [CONSOLE] [2024-01-15 14:30] ERROR: disk full
//   [CONSOLE] [2024-01-15 14:30] ERROR: timeout
// app.log also contains the same two lines
```

**Decorator chaining order matters:**

```cpp
logger->log("ERROR: disk full")
  |
  v FileDecorator::log()  - writes to app.log, then forwards
  |
  v FilterDecorator::log() - checks for "ERROR", passes through
  |
  v TimestampDecorator::log() - prepends timestamp
  |
  v ConsoleLogger::log() - prints to stdout
```

The filter runs *before* the timestamp is added, so "ERROR" in the filter matches against the raw message, not the decorated one. If you wanted the filter to run after timestamping, just swap their positions in the chain. That kind of flexibility is the whole point of the pattern.

---

### Q3: Compare decorator composition with CRTP mixin composition for the same extensibility goal

Decorators and CRTP mixins both add behavior to a type, but they operate at different times and with different costs.

**Comparison:**

| Aspect | Decorator (runtime) | CRTP Mixin (compile-time) |
| --- | --- | --- |
| **When composed** | Runtime - can change at any point | Compile-time - fixed at compilation |
| **Type** | Single type: `ILogger*` | Distinct types: `Timestamp<Filter<Console>>` |
| **Virtual dispatch** | Yes - one vtable call per decorator | No - all inlined |
| **Overhead** | Heap allocation + pointer chase per layer | Zero overhead (templates) |
| **Flexibility** | Can swap decorators at runtime | Fixed chain, needs recompilation |
| **Open for extension** | New decorators without modifying existing code | Same, via new CRTP mixin |

```cpp
// CRTP Mixin approach (compile-time decorator):
#include <iostream>
#include <string>

template <typename Base>
struct TimestampMixin : Base {
    void log(const std::string& msg) {
        Base::log("[14:30:45] " + msg);  // static dispatch - no virtual!
    }
};

template <typename Base>
struct UppercaseMixin : Base {
    void log(std::string msg) {
        for (auto& c : msg) c = static_cast<char>(std::toupper(c));
        Base::log(msg);
    }
};

struct ConsolePrinter {
    void log(const std::string& msg) {
        std::cout << msg << "\n";
    }
};

int main() {
    // Compose at compile-time:
    TimestampMixin<UppercaseMixin<ConsolePrinter>> logger;
    logger.log("hello world");
    // Output: [14:30:45] HELLO WORLD
    // Zero virtual overhead - all calls inlined
}
```

The CRTP chain `TimestampMixin<UppercaseMixin<ConsolePrinter>>` is a single type. The compiler sees the entire chain at once and can inline everything. But you can't rearrange the chain without recompiling, and you can't do it based on a config file or user input.

**Rule of thumb:**

- Use **Decorator** when you need runtime flexibility (config-driven, user-selected).
- Use **CRTP Mixin** when the composition is known at compile time and you want zero overhead.

---

## Notes

- **`std::istream`/`std::ostream`** use a form of decoration via `std::streambuf`.
- **Decorator + `unique_ptr`** gives value semantics to the chain - easy to move, impossible to accidentally share.
- **Avoid deep chains** (more than 5 decorators): hard to debug, pointer-chase overhead adds up.
- **C++23 `std::generator`** and range adaptors are essentially compile-time decorators for lazy sequences: `views::filter | views::transform | views::take`.
- The base `LoggerDecorator` that forwards all calls is sometimes called the **Transparent Forwarding Decorator** - it exists solely so each concrete decorator only overrides the methods it cares about.
