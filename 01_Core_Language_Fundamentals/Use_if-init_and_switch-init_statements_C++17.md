# Use if-init and switch-init statements (C++17)

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#selection-statements-with-initializer>  

---

## Topic Overview

C++17 added **init-statements** to `if` and `switch`, letting you declare variables
scoped tightly to the conditional block. The motivating problem is simple: you often need a variable only for the duration of an `if` check, but without this feature, the variable leaks into the surrounding scope.

### if With Init Statement

Here's the before-and-after. The C++17 form is cleaner and signals the reader that `it` has no meaning outside this block:

```cpp
// Before C++17:
{
    auto it = map.find(key);
    if (it != map.end()) {
        use(it->second);
    }
}   // need extra braces to limit 'it' scope

// C++17:
if (auto it = map.find(key); it != map.end()) {
    use(it->second);
}
// 'it' is out of scope here - clean!
```

### More Examples

The syntax works with any declaration - lock guards, optionals, error codes, structured bindings:

```cpp
// With lock_guard:
if (std::lock_guard lock(mtx); shared_data.ready) {
    process(shared_data);
}
// lock is released here

// With optional:
if (auto opt = try_parse(input); opt.has_value()) {
    use(*opt);
} else {
    std::cerr << "parse failed\n";
    // 'opt' is still accessible here in the else branch!
}

// With error codes:
if (auto ec = do_operation(); ec != ErrorCode::OK) {
    log_error(ec);
    return ec;
}

// Nested:
if (auto [it, ok] = cache.try_emplace(key, value); !ok) {
    std::cout << "key already exists with value: " << it->second;
}
```

### switch With Init Statement

The same idea applies to `switch` - the init variable is scoped to the entire switch block:

```cpp
switch (auto ch = read_char(); ch) {
    case 'q': quit(); break;
    case 'h': help(); break;
    default:  process(ch); break;
}
// 'ch' is out of scope here

// With enum:
enum class State { Idle, Running, Done };
switch (State s = get_state(); s) {
    case State::Idle:    start(); break;
    case State::Running: wait();  break;
    case State::Done:    clean(); break;
}
```

### Why It Matters

1. **Reduced scope** - variables don't leak into surrounding code.
2. **Fewer name collisions** - temporary variables stay local.
3. **Cleaner code** - declaration and condition on one line.
4. **RAII-friendly** - lock_guard scoped to exactly the if/else block.

---

## Self-Assessment

### Q1: Rewrite the if-init pattern without init form and explain scope differences

This comparison makes the scope difference explicit. Notice that without `if`-init you have to either accept scope leakage or add ugly extra braces:

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> scores = {{"Alice", 95}, {"Bob", 87}, {"Charlie", 72}};

    // === C++17 if-init form ===
    if (auto it = scores.find("Bob"); it != scores.end()) {
        std::cout << "Found: " << it->second << "\n";  // 87
    }
    // 'it' does NOT exist here - scope is limited to the if/else block

    // === Pre-C++17 equivalent (without extra braces) ===
    auto it2 = scores.find("Bob");
    if (it2 != scores.end()) {
        std::cout << "Found: " << it2->second << "\n";  // 87
    }
    // 'it2' STILL EXISTS here - scope leaks into surrounding code!
    // This can cause name collisions and accidental reuse:
    it2 = scores.find("NonExistent");  // accidentally reusing it2

    // === Pre-C++17 equivalent (with extra braces to limit scope) ===
    {
        auto it3 = scores.find("Alice");
        if (it3 != scores.end()) {
            std::cout << "Found: " << it3->second << "\n";  // 95
        }
    }
    // 'it3' is out of scope now - but the extra braces are ugly!

    // Scope comparison:
    // C++17 if-init: variable scoped to if + else block (clean)
    // Pre-C++17:     variable leaks OR needs extra braces (ugly)

    // The else branch has access to the init variable too:
    if (auto it = scores.find("Dave"); it != scores.end()) {
        std::cout << "Found Dave: " << it->second << "\n";
    } else {
        std::cout << "Dave not found (it is valid but == end())\n";
        // 'it' is accessible here in else - useful for error reporting
    }
}
```

### Q2: Show how if-init reduces the scope of a lock_guard inside an if block

This is one of the most compelling real-world uses. The lock is held for exactly as long as the critical section lasts - no wider, no narrower:

```cpp
#include <iostream>
#include <mutex>
#include <string>

struct SharedState {
    std::mutex mtx;
    std::string data = "Hello, World!";
    bool ready = true;
};

SharedState shared;

void process_with_init() {
    // C++17 if-init: lock_guard scoped EXACTLY to if/else
    if (std::lock_guard lock(shared.mtx); shared.ready) {
        // lock is held here - safe to access shared.data
        std::cout << "Processing: " << shared.data << "\n";
    }
    // lock is released HERE - as soon as the if/else block ends
    // Minimal lock duration!

    std::cout << "Lock is released, doing other work...\n";
}

void process_without_init() {
    // Pre-C++17: lock scope is wider than necessary
    {
        std::lock_guard<std::mutex> lock(shared.mtx);
        if (shared.ready) {
            std::cout << "Processing: " << shared.data << "\n";
        }
    }  // lock released here - needed extra braces

    // Or worse - lock held too long:
    // std::lock_guard<std::mutex> lock(shared.mtx);
    // if (shared.ready) { ... }
    // do_expensive_unrelated_work();  // Still holding the lock!
}

int main() {
    process_with_init();
    process_without_init();
}
```

### Q3: Demonstrate switch-init with an enum and explain scope hygiene

The switch-init form is especially useful when the switched-on value carries information you need in the case bodies - here, the full `Response` object is available inside every case:

```cpp
#include <iostream>
#include <string>

enum class HttpStatus { OK = 200, NotFound = 404, ServerError = 500, BadRequest = 400 };

struct Response {
    HttpStatus status;
    std::string body;
};

Response fetch_page() {
    return {HttpStatus::NotFound, "Page not found"};
}

int main() {
    // C++17 switch-init: response scoped to the switch block
    switch (auto resp = fetch_page(); resp.status) {
        case HttpStatus::OK:
            std::cout << "Success: " << resp.body << "\n";
            break;
        case HttpStatus::NotFound:
            std::cout << "404: " << resp.body << "\n";
            break;
        case HttpStatus::ServerError:
            std::cout << "500: Server Error\n";
            break;
        default:
            std::cout << "Unhandled status: " << static_cast<int>(resp.status) << "\n";
            break;
    }
    // 'resp' is out of scope here - clean!

    // Without init form, 'resp' would leak:
    // auto resp = fetch_page();  // exists for rest of function
    // switch (resp.status) { ... }

    // Another example: switch on character with init
    std::string input = "quit";
    switch (char cmd = input[0]; cmd) {
        case 'q': std::cout << "Quit\n"; break;
        case 'h': std::cout << "Help\n"; break;
        default:  std::cout << "Unknown: " << cmd << "\n"; break;
    }
    // 'cmd' is out of scope - no collision with other 'cmd' variables
}
```

---

## Notes

- The init variable is accessible in both `if` and `else` branches - useful for logging errors with the original value.
- Combines naturally with structured bindings: `if (auto [it, ok] = map.try_emplace(k, v); !ok) { ... }`
- C++20 extends this concept to range-for: `for (auto v = get(); auto x : v) { ... }` (range-for init-statement).
- Works with any declaration, not just `auto`: `if (int err = check(); err != 0) { ... }`
- Prefer if-init over separate declaration + if when the variable is only needed inside the if/else block.
