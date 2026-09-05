# Use early return to reduce nesting and improve readability (guard clauses)

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Res-nested>  

---

## Topic Overview

**Guard clauses** are early returns that deal with the error and edge cases up front, so that the happy path runs at the lowest nesting level with nothing else to worry about. The alternative - wrapping everything in nested `if` blocks - produces what's commonly called the "arrow anti-pattern," where the code drifts steadily to the right as conditions pile up.

The shape comparison says it all:

```cpp
Arrow anti-pattern:              Guard clauses:
if (a) {                         if (!a) return error;
  if (b) {                       if (!b) return error;
    if (c) {                     if (!c) return error;
      if (d) {                   if (!d) return error;
        // actual work           // actual work (no nesting!)
      }
    }
  }
}
```

The guard-clause version is flat. You read down a list of "bail out if this is wrong" checks, and when you reach the end of the list you know everything is fine and you can focus entirely on the logic.

---

## Self-Assessment

### Q1: Refactor deeply nested code into flat guard clauses

The `process_bad` function below nests five conditions inside each other. Finding which `}` closes which `if` is genuinely hard, and each new requirement adds another indent level. The `process_good` version says the same thing in a flat list of preconditions followed by one line of actual work.

```cpp
#include <iostream>
#include <optional>
#include <string>

struct User { std::string name; int age; bool active; };
struct Order { double amount; std::string product; };
struct Payment { bool valid; double balance; };

// BAD: 5 levels of nesting (arrow anti-pattern)
std::string process_bad(User* user, Order* order, Payment* payment) {
    if (user != nullptr) {
        if (user->active) {
            if (order != nullptr) {
                if (order->amount > 0) {
                    if (payment != nullptr && payment->valid) {
                        if (payment->balance >= order->amount) {
                            return "Order processed for " + user->name;
                        } else {
                            return "Insufficient balance";
                        }
                    } else {
                        return "Invalid payment";
                    }
                } else {
                    return "Invalid order amount";
                }
            } else {
                return "No order";
            }
        } else {
            return "User inactive";
        }
    } else {
        return "No user";
    }
}

// GOOD: flat guard clauses
std::string process_good(User* user, Order* order, Payment* payment) {
    if (!user)                              return "No user";
    if (!user->active)                      return "User inactive";
    if (!order)                             return "No order";
    if (order->amount <= 0)                 return "Invalid order amount";
    if (!payment || !payment->valid)        return "Invalid payment";
    if (payment->balance < order->amount)   return "Insufficient balance";

    return "Order processed for " + user->name;
}

int main() {
    User user{"Alice", 30, true};
    Order order{49.99, "Widget"};
    Payment payment{true, 100.0};

    std::cout << process_good(&user, &order, &payment) << '\n';

    User inactive{"Bob", 25, false};
    std::cout << process_good(&inactive, &order, &payment) << '\n';
    std::cout << process_good(nullptr, &order, &payment) << '\n';
}
// Expected output:
// Order processed for Alice
// User inactive
// No user
```

Notice how each guard clause lines up at the same indent level. You can scan straight down the left margin and read off the complete set of failure conditions, then find the success case immediately below them.

### Q2: Show that early return is compatible with RAII

A common objection to early return is "but I need to clean up before I leave." The answer is: use RAII, and cleanup happens automatically on every return path whether you think about it or not. Locks, file handles, database connections - all of them get destroyed when the scope exits, no matter how many early returns there are.

```cpp
#include <fstream>
#include <iostream>
#include <memory>
#include <mutex>
#include <string>

std::mutex mtx;

class DatabaseConnection {
public:
    DatabaseConnection() { std::cout << "DB connected\n"; }
    ~DatabaseConnection() { std::cout << "DB disconnected\n"; }
    bool query(const std::string& q) { return q.size() > 0; }
};

std::string process_request(const std::string& request) {
    // RAII: all resources cleaned up on ANY return path
    std::lock_guard<std::mutex> lock(mtx);  // released on return
    auto db = std::make_unique<DatabaseConnection>();  // destroyed on return

    if (request.empty())
        return "Error: empty request";  // lock released, db destroyed

    if (request.size() > 1000)
        return "Error: request too long";  // lock released, db destroyed

    if (!db->query(request))
        return "Error: query failed";  // lock released, db destroyed

    return "Success";  // lock released, db destroyed
    // NO manual cleanup needed! RAII handles everything.
}

int main() {
    std::cout << process_request("SELECT * FROM users") << '\n';
    std::cout << "---\n";
    std::cout << process_request("") << '\n';
}
// Expected output:
// DB connected
// DB disconnected
// Success
// ---
// DB connected
// DB disconnected
// Error: empty request
```

Every return path in `process_request` - success or failure - produces the same "DB disconnected" output, proving the destructor fires every time. This is exactly why the combination of RAII and early return is so clean: neither pattern fights the other.

### Q3: Explain the arrow anti-pattern and how guard clauses eliminate it

The **arrow anti-pattern** gets its name from the shape of the code when viewed from the side - the indentation keeps growing to the right, forming a visual arrowhead that points at the actual logic buried at the deepest level.

```cpp
if (a) {
    if (b) {
        if (c) {
            if (d) {
                // logic here  <--- the "tip" of the arrow
            }
        }
    }
}
```

**Problems:**

1. Hard to tell which `}` matches which `if`.
2. Error handling is far from the check that triggers it - the error message for the first condition is physically separated from the check by all the nested code.
3. Main logic is buried at the deepest level, where you have to hold four conditions in your head just to understand what the code is doing.
4. Every new condition adds another level, making the problem progressively worse.

**Guard clauses fix all of these:**

```cpp
if (!a) return;  // error: handle immediately
if (!b) return;  // error: handle immediately
if (!c) return;  // error: handle immediately
if (!d) return;  // error: handle immediately
// Main logic is at the TOP level, not buried deep
```

Each check and its response are right next to each other. The main logic sits at the top indent level, uncluttered, with no braces to count.

---

## Notes

- Guard clauses work perfectly with RAII - destructors run on every return path automatically.
- Don't use early return with raw `new`/`delete` - use smart pointers instead, and then early return is safe.
- `std::optional` and `std::expected` (C++23) work great with guard clauses as return types.
- Some style guides limit function length, making guard clauses even more valuable for keeping the happy path short and readable.
