# Use the Null Object pattern to avoid null pointer checks throughout code

**Category:** Modern OOP Patterns  
**Standard:** C++11  
**Reference:** <https://en.wikipedia.org/wiki/Null_object_pattern>  

---

## Topic Overview

The **Null Object pattern** replaces `nullptr` checks with a concrete implementation that does nothing. Instead of testing `if (logger != nullptr)` at every call site, you inject a `NullLogger` that silently discards all messages. This eliminates defensive `if`-checks, simplifies calling code, and makes interfaces unconditionally callable.

The real insight is that "no logging" and "nil logging" are the same behavior - you just want the code that logs to not have to care which one it's doing. By giving the caller a real object to work with (one that happens to be a no-op), you push the optionality to the construction site and keep the working code clean.

```cpp
WITHOUT Null Object:                   WITH Null Object:

void process(Logger* log) {            void process(Logger& log) {
    do_work();                             do_work();
    if (log) log->info("done");            log.info("done");  // always safe
    if (log) log->debug("x=5");            log.debug("x=5");  // no checks!
}                                      }

// Caller must remember nullptr:       // Caller injects NullLogger if no logging:
process(nullptr);                       NullLogger noop;
                                        process(noop);
```

### When to Use

| Situation | Use Null Object? |
| --- | --- |
| Optional logging/tracing | Yes - silence is a valid behavior |
| Optional audio/rendering | Yes - "do nothing" is common |
| Missing database connection | No - this is an error, not "nothing" |
| User not found in lookup | No - use `std::optional` or error code |

---

## Self-Assessment

### Q1: Implement a NullLogger that satisfies the Logger interface but does nothing

The implementation is intentionally trivial - every method body is empty. That's the whole point. The value comes from having a real object to hand around so callers never have to check for null:

```cpp
#include <iostream>
#include <memory>
#include <string>

// Abstract interface
struct Logger {
    virtual void info(const std::string& msg) = 0;
    virtual void warn(const std::string& msg) = 0;
    virtual void error(const std::string& msg) = 0;
    virtual ~Logger() = default;
};

// Real implementation
struct ConsoleLogger : Logger {
    void info(const std::string& msg)  override { std::cout << "[INFO]  " << msg << "\n"; }
    void warn(const std::string& msg)  override { std::cout << "[WARN]  " << msg << "\n"; }
    void error(const std::string& msg) override { std::cout << "[ERROR] " << msg << "\n"; }
};

// Null Object: satisfies the interface, does NOTHING
struct NullLogger : Logger {
    void info(const std::string&)  override { /* intentionally empty */ }
    void warn(const std::string&)  override { /* intentionally empty */ }
    void error(const std::string&) override { /* intentionally empty */ }
};

// Business logic -- no nullptr checks anywhere!
void process_order(Logger& log, int order_id) {
    log.info("Processing order #" + std::to_string(order_id));
    // ... business logic ...
    log.info("Order #" + std::to_string(order_id) + " complete");
}

int main() {
    ConsoleLogger console;
    NullLogger    noop;

    std::cout << "--- With ConsoleLogger ---\n";
    process_order(console, 42);

    std::cout << "\n--- With NullLogger (silent) ---\n";
    process_order(noop, 99);  // exactly the same call, zero output

    std::cout << "Done.\n";
}
// Expected output:
//   --- With ConsoleLogger ---
//   [INFO]  Processing order #42
//   [INFO]  Order #42 complete
//
//   --- With NullLogger (silent) ---
//   Done.
```

`process_order` calls `log.info` the same way regardless of which logger it receives. The decision about whether to log was made before entering the function, not inside it.

---

### Q2: Show how injecting a NullLogger replaces nullptr-checking guards throughout the code

Here is the before-and-after side by side. Count how many `if (logger_)` guards disappear in the After version:

```cpp
#include <iostream>
#include <memory>
#include <string>

struct Logger {
    virtual void log(const std::string& msg) = 0;
    virtual ~Logger() = default;
};

struct ConsoleLogger : Logger {
    void log(const std::string& msg) override { std::cout << msg << "\n"; }
};

struct NullLogger : Logger {
    void log(const std::string&) override {}
};

// --- BEFORE: Defensive nullptr checks EVERYWHERE ---
class ServiceBefore {
    Logger* logger_ = nullptr;
public:
    void set_logger(Logger* l) { logger_ = l; }

    void step1() {
        if (logger_) logger_->log("step1 start");  // check!
        // ... work ...
        if (logger_) logger_->log("step1 done");   // check!
    }
    void step2() {
        if (logger_) logger_->log("step2 start");  // check!
        // ... work ...
        if (logger_) logger_->log("step2 done");   // check!
    }
    // Every method has 2+ nullptr checks. Fragile and noisy.
};

// --- AFTER: Null Object -- ZERO checks ---
class ServiceAfter {
    inline static NullLogger default_logger_;
    Logger& logger_;
public:
    // Default to NullLogger -- never null!
    explicit ServiceAfter(Logger& log = default_logger_) : logger_(log) {}

    void step1() {
        logger_.log("step1 start");  // always safe
        // ... work ...
        logger_.log("step1 done");   // always safe
    }
    void step2() {
        logger_.log("step2 start");  // no checks needed
        // ... work ...
        logger_.log("step2 done");
    }
};

int main() {
    ConsoleLogger console;

    // Before-style: must remember to check
    ServiceBefore sb;
    sb.step1();  // silent -- logger_ is nullptr

    sb.set_logger(&console);
    sb.step1();  // now logs

    std::cout << "---\n";

    // After-style: clean, no checks
    ServiceAfter sa1;          // uses NullLogger by default
    sa1.step1();               // silent -- no output, no crash

    ServiceAfter sa2(console); // uses ConsoleLogger
    sa2.step1();               // logs normally
}
// Expected output:
//   step1 start
//   step1 done
//   ---
//   step1 start
//   step1 done
```

**Lines of nullptr checks eliminated:** In the After version, every `if (logger_)` test is gone. The code is simpler, less error-prone, and the compiler can optimize the NullLogger calls into no-ops (empty virtual function -> likely inlined away with LTO).

---

### Q3: Explain when the Null Object pattern adds clarity vs hides real error conditions

The pattern is genuinely helpful when "nothing happens" is a correct outcome. It becomes dangerous when "nothing happens" silently covers up a failure that the caller should know about. Here are both scenarios:

```cpp
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>

// --- GOOD: Null Object adds clarity ---
// Audio system -- silence is a VALID output
struct AudioBackend {
    virtual void play(const std::string& sound) = 0;
    virtual ~AudioBackend() = default;
};

struct RealAudio : AudioBackend {
    void play(const std::string& s) override {
        std::cout << "Playing: " << s << "\n";
    }
};

struct NullAudio : AudioBackend {
    void play(const std::string&) override {} // silence is valid!
};

// Game doesn't need to check if audio exists
void game_loop(AudioBackend& audio) {
    audio.play("explosion.wav");  // works with or without audio
    audio.play("music.ogg");
}

// --- BAD: Null Object HIDES errors ---
struct Database {
    virtual bool save(const std::string& key, const std::string& val) = 0;
    virtual ~Database() = default;
};

struct NullDatabase : Database {
    bool save(const std::string&, const std::string&) override {
        return true;  // LIES! Data is silently lost!
    }
};

void save_user_data(Database& db) {
    bool ok = db.save("user:123", "critical_data");
    if (ok)
        std::cout << "Data saved successfully\n";  // FALSE confidence!
}

int main() {
    // GOOD use: NullAudio in headless server / testing
    std::cout << "=== NullAudio (good) ===\n";
    NullAudio silent;
    game_loop(silent);  // works perfectly, silence is intentional

    RealAudio real;
    game_loop(real);

    // BAD use: NullDatabase silently loses data
    std::cout << "\n=== NullDatabase (BAD) ===\n";
    NullDatabase fake_db;
    save_user_data(fake_db);  // "Data saved" -- but nothing was saved!
}
// Expected output:
//   === NullAudio (good) ===
//   Playing: explosion.wav
//   Playing: music.ogg
//
//   === NullDatabase (BAD) ===
//   Data saved successfully
```

The `NullDatabase` is the trap: it returns `true` while doing nothing, which gives the caller false confidence. If the database is unavailable, that is an error condition that deserves an exception or an error code, not silent swallowing.

**Decision framework:**

| Criterion | Null Object OK | Null Object Dangerous |
| --- | --- | --- |
| **Do-nothing is valid behavior?** | Yes -> logging, audio, metrics | No -> database, file write |
| **Caller expects side effects?** | No side effects expected | Caller relies on effect happening |
| **Failure should be silent?** | Yes -> optional diagnostics | No -> data loss, security hole |
| **Testing/mocking?** | Good for test doubles | May mask bugs in tests |
| **Alternatives** | | `std::optional`, exceptions, error codes |

---

## Notes

- **Singleton NullLogger:** A single `NullLogger` instance can be shared (stateless) - make it a `static` member or global `const`.
- **`std::optional` vs Null Object:** Use `optional<T>` when the absence itself carries meaning. Use Null Object when you want polymorphic "no-op" behavior.
- **Performance:** Virtual null object calls have vtable overhead. For hot paths, consider `if constexpr` or compile-time policies instead.
- **`std::monostate`:** In `std::variant`, `monostate` serves a similar role - it represents "no value" as a type rather than a null pointer.
- **Logging libraries** (spdlog, log4cxx) commonly use this pattern internally with sink-based architectures.
