# Use std::enable_shared_from_this for safe shared_ptr from this

**Category:** Modern OOP Patterns  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/enable_shared_from_this>  

---

## Topic Overview

Sometimes a member function needs to create a `shared_ptr` that shares ownership of `this`. Naively writing `shared_ptr<T>(this)` creates a **second** control block, leading to double-free. `std::enable_shared_from_this<T>` provides a safe `shared_from_this()` method that returns a `shared_ptr` sharing the existing control block.

The reason this trips people up is that a `shared_ptr`'s reference count lives in a separate heap allocation called the **control block**. When you create a new `shared_ptr<T>(this)`, you are not incrementing an existing count - you are creating a completely independent control block that also thinks it owns the same raw pointer. When both go to zero, the object gets deleted twice.

### How It Works

`enable_shared_from_this` stores a `weak_ptr` internally when the object is first managed by a `shared_ptr`. Calling `shared_from_this()` locks that `weak_ptr` and returns a `shared_ptr` that shares the original control block instead of creating a new one:

```cpp
std::make_shared<Widget>()
   |
   v
┌──────────────────────────────┐
│  Control Block               │
│  strong_count = 1            │
│  weak_count = 1 (internal)   │ <- enable_shared_from_this stores
│  ptr -> Widget object        │     a weak_ptr to THIS control block
└──────────────────────────────┘
   |
   v
┌──────────────────────────────┐
│  Widget object               │
│  (inherits from              │
│   enable_shared_from_this)   │
│                              │
│  shared_from_this()          │
│  -> locks internal weak_ptr  │
│  -> returns shared_ptr with  │
│    SAME control block         │
│    strong_count -> 2         │
└──────────────────────────────┘
```

### Precondition

The object **must already be owned by at least one `shared_ptr`** before calling `shared_from_this()`. Otherwise: UB (C++11-14), `std::bad_weak_ptr` exception (C++17+).

---

## Self-Assessment

### Q1: Show a double-free bug from creating `shared_ptr(this)` inside a member function

This is the bug you will encounter if you try the naive approach. Look carefully at the use counts - both `sp1` and `sp2` report a use count of 1, which means they have completely independent control blocks pointing at the same object:

```cpp
#include <iostream>
#include <memory>

struct Logger {
    std::string name;

    Logger(std::string n) : name(std::move(n)) {
        std::cout << "Logger(" << name << ") constructed\n";
    }
    ~Logger() {
        std::cout << "Logger(" << name << ") destroyed\n";
    }

    // BUG: creates a NEW control block for 'this'!
    std::shared_ptr<Logger> get_shared() {
        return std::shared_ptr<Logger>(this);  // <- DOUBLE-FREE!
    }
};

int main() {
    // The problem:
    {
        auto sp1 = std::make_shared<Logger>("FileLog");
        // sp1's control block: strong_count = 1

        auto sp2 = sp1->get_shared();
        // sp2 creates a SECOND control block for the SAME pointer
        // Now TWO control blocks think they own the same object!

        std::cout << "sp1.use_count() = " << sp1.use_count() << "\n";  // 1 (not 2!)
        std::cout << "sp2.use_count() = " << sp2.use_count() << "\n";  // 1

        // When sp2 goes out of scope -> strong_count=0 -> delete Logger
        // When sp1 goes out of scope -> strong_count=0 -> delete Logger AGAIN!
        // === DOUBLE FREE -> UNDEFINED BEHAVIOR ===
    }
    // To safely demonstrate without crash, comment out the block above
    // and use the fixed version in Q2.

    std::cout << "WARNING: The code above causes double-free UB!\n";
}
// Possible output (before crash):
//   Logger(FileLog) constructed
//   sp1.use_count() = 1
//   sp2.use_count() = 1
//   Logger(FileLog) destroyed
//   Logger(FileLog) destroyed    <- DOUBLE FREE!
```

**Memory diagram of the bug:**

```cpp
sp1 --> [Control Block A: count=1] --> Logger object <-- [Control Block B: count=1] <-- sp2
         Two control blocks, one object = disaster!
```

---

### Q2: Fix it using `std::enable_shared_from_this` and `shared_from_this()`

The fix is to inherit from `enable_shared_from_this<Logger>` and call `shared_from_this()` instead of constructing a new `shared_ptr`. Now both `sp1` and `sp2` share the same control block, as the use count correctly shows:

```cpp
#include <iostream>
#include <memory>
#include <vector>

struct Logger : std::enable_shared_from_this<Logger> {
    std::string name;

    Logger(std::string n) : name(std::move(n)) {
        std::cout << "Logger(" << name << ") constructed\n";
    }
    ~Logger() {
        std::cout << "Logger(" << name << ") destroyed\n";
    }

    // SAFE: shares the existing control block
    std::shared_ptr<Logger> get_shared() {
        return shared_from_this();  // <- shares same control block!
    }

    // Common real-world use: register self in a callback/observer
    void register_in(std::vector<std::shared_ptr<Logger>>& registry) {
        registry.push_back(shared_from_this());
    }
};

int main() {
    std::vector<std::shared_ptr<Logger>> registry;

    {
        auto sp1 = std::make_shared<Logger>("FileLog");
        std::cout << "After creation:   use_count = " << sp1.use_count() << "\n";

        auto sp2 = sp1->get_shared();  // SAFE -- same control block
        std::cout << "After get_shared: use_count = " << sp1.use_count() << "\n";
        std::cout << "sp1 == sp2? " << (sp1 == sp2) << "\n";

        sp1->register_in(registry);
        std::cout << "After register:   use_count = " << sp1.use_count() << "\n";

        // sp1 and sp2 go out of scope, but registry keeps Logger alive
    }

    std::cout << "After block: registry[0].use_count = "
              << registry[0].use_count() << "\n";
    std::cout << "Logger still alive: " << registry[0]->name << "\n";

    registry.clear();  // Last reference gone -> Logger destroyed
    std::cout << "After clear: registry empty\n";
}
// Expected output:
//   Logger(FileLog) constructed
//   After creation:   use_count = 1
//   After get_shared: use_count = 2
//   sp1 == sp2? 1
//   After register:   use_count = 3
//   After block: registry[0].use_count = 1
//   Logger still alive: FileLog
//   Logger(FileLog) destroyed
//   After clear: registry empty
```

**Memory diagram (fixed):**

```cpp
sp1 --> [Control Block: count=3] --> Logger object
sp2 --/                              (inherits enable_shared_from_this)
registry[0] --/                       L- internal weak_ptr -> same control block
```

One control block, three owners, one destruction.

---

### Q3: Explain the requirement that the object must already be owned by a `shared_ptr` before calling `shared_from_this`

The reason `shared_from_this()` requires an existing owner is that the internal `weak_ptr` is only initialized when a `shared_ptr` is first constructed for the object. Before that moment, there is nothing to lock, so the call either invokes undefined behavior or throws `bad_weak_ptr`. Here are the common mistakes and the correct patterns side by side:

```cpp
#include <iostream>
#include <memory>
#include <stdexcept>

struct Widget : std::enable_shared_from_this<Widget> {
    std::string name;

    Widget(std::string n) : name(std::move(n)) {}

    std::shared_ptr<Widget> get_ptr() {
        return shared_from_this();
    }
};

int main() {
    // --- WRONG: stack object -- NOT owned by shared_ptr ---
    try {
        Widget w{"StackWidget"};
        // w is on the stack -- no shared_ptr owns it
        // C++17: throws std::bad_weak_ptr
        // C++11/14: undefined behavior!
        auto sp = w.get_ptr();
    } catch (const std::bad_weak_ptr& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    // --- WRONG: raw new -- NOT yet assigned to shared_ptr ---
    try {
        Widget* raw = new Widget{"RawWidget"};
        // raw is not owned by any shared_ptr yet
        auto sp = raw->get_ptr();  // throws!
    } catch (const std::bad_weak_ptr& e) {
        std::cout << "Caught: " << e.what() << "\n";
        // Memory leak! raw was never deleted.
    }

    // --- WRONG: calling in constructor ---
    // struct BadWidget : std::enable_shared_from_this<BadWidget> {
    //     BadWidget() {
    //         auto sp = shared_from_this();  // UB! make_shared hasn't
    //     }                                  // finished setting up yet
    // };

    // --- CORRECT: object owned by shared_ptr FIRST ---
    auto sp1 = std::make_shared<Widget>("GoodWidget");
    auto sp2 = sp1->get_ptr();  // OK -- sp1 already owns it
    std::cout << "sp1.use_count() = " << sp1.use_count() << "\n";
    std::cout << "sp2.use_count() = " << sp2.use_count() << "\n";

    // --- CORRECT: shared_ptr from raw (assign BEFORE calling) ---
    Widget* raw2 = new Widget{"RawGood"};
    std::shared_ptr<Widget> sp3(raw2);  // now raw2 is owned
    auto sp4 = raw2->get_ptr();         // OK!
    std::cout << "sp3 == sp4? " << (sp3 == sp4) << "\n";
}
// Expected output:
//   Caught: bad_weak_ptr
//   Caught: bad_weak_ptr
//   sp1.use_count() = 2
//   sp2.use_count() = 2
//   sp3 == sp4? 1
```

The constructor case is especially easy to miss: even if you eventually call `make_shared<Widget>()`, during the constructor body the control block has not been fully wired up yet, so `shared_from_this()` inside a constructor is always wrong.

**Summary of rules:**

| Scenario | Result |
| --- | --- |
| `make_shared<T>()` -> `shared_from_this()` | OK - correct control block |
| `shared_ptr<T>(new T)` -> `shared_from_this()` | OK - if called after construction |
| Stack object -> `shared_from_this()` | UB / `bad_weak_ptr` |
| Inside constructor -> `shared_from_this()` | UB - shared_ptr not set up yet |
| `unique_ptr<T>` -> `shared_from_this()` | UB - unique_ptr isn't shared_ptr |

---

## Notes

- **Inheritance:** `enable_shared_from_this` should be inherited **publicly**. Private inheritance prevents `make_shared` from initializing the internal `weak_ptr`.
- **C++17 improvement:** `weak_from_this()` added - returns `weak_ptr` without requiring existing `shared_ptr` ownership (returns expired `weak_ptr` if no owner).
- **Thread safety:** `shared_from_this()` is thread-safe (same as `shared_ptr` copy).
- **Multiple inheritance:** If a class inherits from multiple `enable_shared_from_this` bases, behavior is ambiguous - avoid this.
- **Common pattern:** Async callbacks: `auto self = shared_from_this(); io.async_read(buf, [self](auto ec, auto n) { self->on_read(ec, n); });` - prevents the object from being destroyed while the callback is pending.
