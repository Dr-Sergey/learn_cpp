# Use the NVI pattern (Non-Virtual Interface) for controlled polymorphism

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

The **Non-Virtual Interface (NVI)** pattern makes all public methods non-virtual, and provides private or protected virtual methods as customization points. The base class controls the algorithm (pre/post conditions, logging, locking), while derived classes only customize specific steps.

The key idea is that when a method is public and virtual, derived classes can override the *entire* behavior including the scaffolding the base class wants to enforce. With NVI, the public method is a non-virtual wrapper that always runs - derived classes get to fill in only the specific step that varies. This is why NVI pairs naturally with the Template Method design pattern.

```cpp
Traditional:                        NVI:
class Base {                        class Base {
public:                             public:
  virtual void process() = 0;        void process() {        // NON-virtual public
  // Derived controls everything      pre_check();
};                                    do_process();           // calls virtual
                                      post_check();
                                    }
                                    private:
                                      virtual void do_process() = 0;  // virtual private
                                    };
```

### Benefits of NVI

| Benefit | How NVI achieves it |
| --- | --- |
| Enforce preconditions | Non-virtual wrapper checks before calling virtual |
| Enforce postconditions | Wrapper checks/logs after virtual returns |
| Add instrumentation | Logging, metrics in wrapper - all derived classes get it |
| Thread safety | Wrapper acquires lock, virtual runs under lock |
| Maintain invariants | Wrapper validates state before/after customization |

---

## Self-Assessment

### Q1: Implement the NVI pattern for a document processor

Here the public interface is just `process()` - a single non-virtual entry point that the base class fully controls. The pre/post conditions, timing, and logging all happen in the wrapper, automatically, for every derived class. Derived classes only implement the specific virtual steps (`validate` and `transform`) and never touch the surrounding machinery:

**Answer:**

```cpp
#include <string>
#include <iostream>
#include <chrono>
#include <stdexcept>

class DocumentProcessor {
public:
    // PUBLIC NON-VIRTUAL interface - controls the workflow
    std::string process(const std::string& input) {
        // Pre-condition: input must not be empty
        if (input.empty())
            throw std::invalid_argument("Empty input document");

        auto start = std::chrono::steady_clock::now();
        log("Processing started, input size: " + std::to_string(input.size()));

        // Step 1: Validate (customizable)
        validate(input);

        // Step 2: Transform (customizable)
        std::string result = transform(input);

        // Post-condition: output must not be empty
        if (result.empty())
            throw std::runtime_error("Processor produced empty output");

        auto elapsed = std::chrono::steady_clock::now() - start;
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(elapsed).count();
        log("Processing complete in " + std::to_string(ms) + "ms, output size: "
            + std::to_string(result.size()));

        return result;
    }

    virtual ~DocumentProcessor() = default;

private:
    // PRIVATE VIRTUAL customization points
    virtual void validate(const std::string& input) = 0;
    virtual std::string transform(const std::string& input) = 0;
    virtual void log(const std::string& msg) {
        std::cout << "[DocProcessor] " << msg << "\n";
    }
};

// Derived: ONLY customizes behavior, not control flow
class MarkdownToHtml : public DocumentProcessor {
    void validate(const std::string& input) override {
        if (input.find('#') == std::string::npos &&
            input.find('*') == std::string::npos)
            throw std::invalid_argument("Not valid Markdown");
    }

    std::string transform(const std::string& input) override {
        std::string html = "<html><body>";
        // ... markdown parsing
        html += input;
        html += "</body></html>";
        return html;
    }
};

class CsvToJson : public DocumentProcessor {
    void validate(const std::string& input) override {
        if (input.find(',') == std::string::npos)
            throw std::invalid_argument("No CSV delimiters found");
    }

    std::string transform(const std::string& input) override {
        return R"({"data": ")" + input + R"("})";
    }

    void log(const std::string& msg) override {
        std::cerr << "[CSV->JSON] " << msg << "\n";  // Custom logging
    }
};

int main() {
    MarkdownToHtml md;
    auto html = md.process("# Hello *world*");
    // Logging, validation, timing all happen automatically

    CsvToJson csv;
    auto json = csv.process("name,age\nAlice,30");
    return 0;
}
```

Notice that `CsvToJson` overrides `log` to redirect to `std::cerr` - it can customize that step too, because `log` is virtual. But it cannot skip the pre/post checks or the timing code, because those live in the non-virtual `process` wrapper. Every processor, regardless of how it is implemented, gets consistent instrumentation for free.

### Q2: Show NVI with thread safety and state machine control

NVI is especially powerful when the base class needs to enforce things that every derived class must get right but that derived classes really should not have to reimplement themselves - thread safety and state machine transitions are perfect examples. Here, the public methods handle all the locking and state checking, and the private virtual `do_*` methods contain only the protocol-specific work:

**Answer:**

```cpp
#include <mutex>
#include <string>
#include <stdexcept>

// NVI with thread-safe state machine
class Connection {
public:
    enum class State { Disconnected, Connecting, Connected, Error };

    // Public non-virtual: controls state transitions + locking
    void connect(const std::string& address) {
        std::lock_guard lock(mutex_);
        if (state_ != State::Disconnected)
            throw std::logic_error("Already connected or connecting");

        state_ = State::Connecting;
        try {
            do_connect(address);   // Virtual customization
            state_ = State::Connected;
        } catch (...) {
            state_ = State::Error;
            throw;
        }
    }

    void send(const std::string& data) {
        std::lock_guard lock(mutex_);
        if (state_ != State::Connected)
            throw std::logic_error("Not connected");
        do_send(data);             // Virtual customization
    }

    std::string receive() {
        std::lock_guard lock(mutex_);
        if (state_ != State::Connected)
            throw std::logic_error("Not connected");
        return do_receive();       // Virtual customization
    }

    void disconnect() {
        std::lock_guard lock(mutex_);
        if (state_ == State::Connected || state_ == State::Error) {
            do_disconnect();       // Virtual customization
            state_ = State::Disconnected;
        }
    }

    State state() const {
        std::lock_guard lock(mutex_);
        return state_;
    }

    virtual ~Connection() = default;

private:
    // Customization points - derived classes focus on protocol
    virtual void do_connect(const std::string& addr) = 0;
    virtual void do_send(const std::string& data) = 0;
    virtual std::string do_receive() = 0;
    virtual void do_disconnect() = 0;

    mutable std::mutex mutex_;
    State state_ = State::Disconnected;
};

// Derived: only implements protocol specifics
class TcpConnection : public Connection {
    int socket_fd_ = -1;

    void do_connect(const std::string& addr) override {
        // socket(), connect()...
        socket_fd_ = 42;
    }
    void do_send(const std::string& data) override {
        // ::send(socket_fd_, ...)
    }
    std::string do_receive() override {
        // ::recv(socket_fd_, ...)
        return "data";
    }
    void do_disconnect() override {
        // ::close(socket_fd_)
        socket_fd_ = -1;
    }
};

class SerialConnection : public Connection {
    void do_connect(const std::string& port) override { /* open /dev/ttyUSB0 */ }
    void do_send(const std::string& data) override { /* write() */ }
    std::string do_receive() override { return "serial data"; }
    void do_disconnect() override { /* close() */ }
};

// Both TcpConnection and SerialConnection get:
// Thread-safe (mutex in base)
// State-machine enforced (can't send before connect)
// Exception-safe (error state on failure)
// WITHOUT duplicating any of that logic
```

Both `TcpConnection` and `SerialConnection` get thread safety, state validation, and exception safety completely for free - they didn't write a single line of mutex or state-checking code. That is the power of NVI when there is genuine cross-cutting logic that every subclass needs to have applied consistently.

### Q3: Compare NVI with traditional virtual and explain when to use each

Not every method needs to go through NVI. Sometimes a plain public virtual is perfectly fine. The table and example below show the thought process for choosing between approaches:

**Answer:**

| Approach | When to use | Trade-off |
| --- | --- | --- |
| **Public virtual** | Simple case, no invariants | No control over pre/post |
| **NVI (private virtual)** | Base needs to enforce behavior | Slight complexity |
| **Protected virtual** | Derived needs to call base impl | `do_draw()` calls `Base::do_draw()` |
| **CRTP** | Compile-time, zero overhead | No runtime polymorphism |

```cpp
// When NVI is overkill - simple case, public virtual is fine:
class Drawable {
public:
    virtual ~Drawable() = default;
    virtual void draw(Canvas& c) const = 0;  // No pre/post needed
};

// When NVI shines - complex lifecycle:
class Transaction {
public:
    // Non-virtual: enforces ACID properties
    void execute() {
        begin_transaction();     // Always runs
        try {
            do_execute();        // Customizable
            commit();            // Always runs on success
        } catch (...) {
            rollback();          // Always runs on failure
            throw;
        }
    }
private:
    virtual void do_execute() = 0;  // Only the work itself is customizable
    void begin_transaction() { /* ... */ }
    void commit() { /* ... */ }
    void rollback() { /* ... */ }
};
```

The `Transaction` example is the clearest illustration of when NVI is the right call. The begin/commit/rollback sequence is non-negotiable - if you let derived classes override `execute` directly, nothing stops them from accidentally skipping `rollback` on an exception. NVI makes that omission structurally impossible.

---

## Notes

- NVI is Herb Sutter's recommendation from GotW #18 - "make virtual functions private".
- `private virtual` is legal in C++ - the base class can call it but derived classes can override it.
- Use `protected virtual` when derived classes need to call the base implementation (`Base::do_thing()`).
- NVI pairs perfectly with the **Template Method** design pattern.
- In embedded: NVI is excellent for hardware abstraction - base controls timing/safety, derived controls registers.
