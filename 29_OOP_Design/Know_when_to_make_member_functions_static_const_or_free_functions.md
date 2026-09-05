# Know when to make member functions static, const, or free functions

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

One of the most common design decisions you face when writing a class is where to put a function. Should it be a member? Should it be `const`? Should it be `static`, or not attached to the class at all? Getting this right keeps your API clean and your encapsulation intact.

Here is the quick reference. Don't worry if the distinctions feel blurry - the decision tree below the table will walk you through the reasoning step by step.

| Function Type | Access `this`? | Access private? | ADL-found? | Use When |
| --- | :---: | :---: | :---: | --- |
| Non-const member | Yes (mutable) | Yes | N/A | Modifies object state |
| `const` member | Yes (read-only) | Yes | N/A | Reads object state |
| `static` member | No | Yes (class-level) | N/A | Utility, factory, no instance needed |
| Free function | No | No (unless `friend`) | Yes | Operators, generic algorithms |
| Hidden friend | No | Yes | Yes (ADL) | Operators, `swap`, stream I/O |

### Decision Tree

Work through these questions top to bottom whenever you are deciding where a function lives:

```cpp
Does it need private access?
├─ No  -> Free function (better encapsulation)
└─ Yes -> Does it need 'this' (instance state)?
         ├─ No  -> Static member function
         └─ Yes -> Does it modify state?
                  ├─ No  -> const member function
                  └─ Yes -> Non-const member function
```

If you remember only one rule from this whole topic, make it this: **free functions are often the better default**, because they reduce the number of things that can touch the class internals.

---

## Self-Assessment

### Q1: Show when each function type is appropriate

Notice how `UserAccount` below neatly separates the four function types. Pay attention to *why* each function belongs where it is - especially `transfer` and `print_rich_users`, which live outside the class entirely.

```cpp
#include <string>
#include <vector>
#include <algorithm>
#include <iostream>

class UserAccount {
    std::string name_;
    double balance_;
    static inline int next_id_ = 0;
    int id_;

public:
    UserAccount(std::string name, double balance)
        : name_(std::move(name)), balance_(balance), id_(next_id_++) {}

    // CONST member: reads state, doesn't modify
    double balance() const noexcept { return balance_; }
    const std::string& name() const noexcept { return name_; }
    int id() const noexcept { return id_; }

    // NON-CONST member: modifies state
    void deposit(double amount) { balance_ += amount; }
    void withdraw(double amount) {
        if (amount > balance_) throw std::runtime_error("Insufficient funds");
        balance_ -= amount;
    }

    // STATIC member: class-level utility, no instance needed
    static int total_accounts() { return next_id_; }
    static UserAccount create_default() {
        return UserAccount("Guest", 0.0);
    }
};

// FREE function: doesn't need private access
void transfer(UserAccount& from, UserAccount& to, double amount) {
    from.withdraw(amount);
    to.deposit(amount);  // Uses public API only
}

// FREE function: generic, works with any container of UserAccount
void print_rich_users(const std::vector<UserAccount>& users, double threshold) {
    for (const auto& u : users)
        if (u.balance() > threshold)
            std::cout << u.name() << ": " << u.balance() << "\n";
}

// HIDDEN FRIEND: operator needs private access + ADL
// (defined inside class, see hidden friends topic)

int main() {
    auto alice = UserAccount("Alice", 1000);
    auto bob = UserAccount("Bob", 500);
    transfer(alice, bob, 200);
    std::cout << "Accounts: " << UserAccount::total_accounts() << "\n";
    return 0;
}
```

`transfer` does not need private access, so it stays outside the class and works through the public API. That means one less function touching internals - a smaller attack surface for bugs.

### Q2: Show the "prefer non-member non-friend" principle (Scott Meyers)

Scott Meyers argued that the best way to *increase* encapsulation is to move functions out of the class whenever they don't need private access. This feels counterintuitive - surely putting something inside the class makes it more "part of the class"? - but the real measure of encapsulation is how many functions can reach the private data. Fewer is better.

```cpp
#include <string>
#include <iostream>

class Widget {
    int value_;
public:
    explicit Widget(int v) : value_(v) {}
    int value() const { return value_; }
    void set_value(int v) { value_ = v; }
};

// Meyers' principle: if a function can be a non-member non-friend,
// make it one. This INCREASES encapsulation.

// BAD: Adding convenience members bloats the class
// class Widget {
//     ...
//     bool is_positive() const { return value_ > 0; }  // Doesn't need private access!
//     Widget doubled() const { return Widget(value_ * 2); }
// };

// GOOD: Free functions using public API
bool is_positive(const Widget& w) { return w.value() > 0; }
Widget doubled(const Widget& w) { return Widget(w.value() * 2); }

// This matters because:
// 1. Fewer functions with private access = fewer places bugs can hide
// 2. Free functions can be extended by anyone without modifying Widget
// 3. Template algorithms work naturally with free functions

template<typename T>
void process(const T& obj) {
    if (is_positive(obj))  // Works with any type that has is_positive()
        std::cout << "Positive!\n";
}

int main() {
    Widget w(42);
    process(w);
    auto w2 = doubled(w);
    std::cout << w2.value() << "\n";  // 84
    return 0;
}
```

Notice that `process` is a template that works with *any* type exposing `is_positive`. That kind of generic reuse only happens naturally when your helper functions live outside the class.

### Q3: Show static members for factory, configuration, and registry patterns

Static members are the right tool when you need something that belongs to the *class as a whole* rather than to any specific instance. The classic use cases are factory methods (named constructors), shared configuration, counters, and registries.

```cpp
#include <unordered_map>
#include <string>
#include <memory>
#include <functional>
#include <iostream>

class Logger {
public:
    enum class Level { Debug, Info, Warn, Error };

    // Static: configuration shared across all instances
    static void set_global_level(Level l) { global_level_ = l; }
    static Level global_level() { return global_level_; }

    // Static: factory method (named constructor)
    static Logger for_module(const std::string& module) {
        return Logger(module);
    }

    // Non-const: logging modifies internal state
    void info(const std::string& msg) {
        if (global_level_ <= Level::Info)
            std::cout << "[" << module_ << "] INFO: " << msg << "\n";
        ++message_count_;
    }

    // Const: read-only access
    int message_count() const { return message_count_; }

private:
    explicit Logger(std::string module) : module_(std::move(module)) {}
    std::string module_;
    int message_count_ = 0;
    static inline Level global_level_ = Level::Info;
};

int main() {
    Logger::set_global_level(Logger::Level::Debug);  // Static: no instance
    auto log = Logger::for_module("network");         // Static factory
    log.info("Connected");                            // Non-const: logs
    std::cout << log.message_count() << "\n";         // Const: reads
    return 0;
}
```

The static factory `for_module` is a named constructor - it gives the creation operation a meaningful name and can enforce invariants before constructing the object. That is much cleaner than a constructor overload soup.

---

## Notes

- Default to `const` for member functions - mark non-const only when mutation is needed.
- Scott Meyers' rule: prefer non-member non-friend functions to increase encapsulation.
- Static members are the right fit for: factories, global config, registries, counters, and utility functions.
- Free functions enable ADL, which is essential for generic code and operator overloading.
- If a function needs private access but not `this`, make it a static member.
- Adding `const` to a method is a contract: callers can trust it won't modify state.
