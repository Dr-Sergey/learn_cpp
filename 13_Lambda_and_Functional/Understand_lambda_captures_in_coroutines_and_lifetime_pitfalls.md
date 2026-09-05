# Understand lambda captures in coroutines and lifetime pitfalls

**Category:** Lambda and Functional  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

### The Core Problem

When a coroutine suspends, its stack frame is saved on the heap (the coroutine frame). But **lambda captures** and **local variables** that the coroutine references may have different lifetimes than the coroutine frame. This creates **dangling reference** bugs that are notoriously hard to detect - the code looks fine, the compiler says nothing, and the crash only appears at runtime under specific timing conditions.

### Dangling Captures After Suspension

Here's the setup that bites people. A local variable exists, a lambda captures it by reference, and the coroutine suspends. While it's suspended, the caller may continue running - and the local variable may go out of scope:

```cpp
#include <coroutine>
#include <iostream>
#include <string>

// Assume task<T> is a simple coroutine return type

task<void> dangerous_example() {
    std::string message = "hello";

    // Lambda captures 'message' by reference
    auto callback = [&message]() -> task<void> {
        co_await some_async_op();
        // DANGER: 'message' may no longer exist if the outer
        // coroutine has been destroyed or the stack has unwound
        std::cout << message << "\n";  // potential UB!
    };

    co_await callback();
}
```

The key thing to understand here is that `co_await` is the dividing line. Everything before it runs synchronously. But after the suspension, the executing context can be completely different - a different thread, a different stack frame, a completely different moment in time. Any reference that crosses a `co_await` is a potential dangling reference.

### Why Coroutines Make This Worse

In regular (non-coroutine) code, a lambda that captures by reference is safe as long as the referenced variable outlives the lambda's use. The call stack enforces that naturally. But coroutines introduce **suspension points** (`co_await`, `co_yield`) where:

1. Execution leaves the function.
2. Other code runs.
3. Execution resumes - but referenced objects may have been destroyed.

The reason this trips people up is that coroutines look like ordinary sequential code. It's not obvious that `co_await some_async_op()` is an exit point where the world can change around you. But it is - and any reference you held before it could be invalid after it resumes.

### Rule: Capture by Value Before Suspension

The fix is straightforward: capture by value when anything crosses a suspension point. The copy lives inside the closure and belongs to it entirely:

```cpp
task<void> safe_example() {
    std::string message = "hello";

    // Capture by VALUE - the copy lives inside the lambda's closure
    auto callback = [message]() -> task<void> {
        co_await some_async_op();
        std::cout << message << "\n";  // Safe: message is owned by the closure
    };

    co_await callback();
}
```

This copies `message` at capture time, so the closure owns its own `std::string` regardless of what happens to the original.

### The Implicit `this` Capture Trap

A particularly insidious case is when a member function is a coroutine. The function implicitly uses `this` to access member variables, but `this` is just a raw pointer - it's not reference-counted, and it won't keep the object alive:

```cpp
struct Server {
    std::string config_;

    task<void> process() {
        // 'this' is captured implicitly - the coroutine frame stores a pointer to Server
        co_await load_data();

        // If the Server object was destroyed during suspension,
        // accessing config_ is UB
        std::cout << config_ << "\n";  // potential dangling 'this'!
    }
};

void start() {
    auto* s = new Server{"production"};
    auto t = s->process();  // coroutine starts, may suspend
    delete s;               // Server destroyed!
    t.resume();             // UB: coroutine accesses dead object via 'this'
}
```

This pattern is everywhere in async server code, which is exactly why it causes so many bugs. The fix is to use `shared_from_this()` to ensure the object stays alive for the duration of the coroutine:

```cpp
struct SafeServer : std::enable_shared_from_this<SafeServer> {
    std::string config_;

    task<void> process() {
        auto self = shared_from_this();  // prevent destruction
        co_await load_data();
        std::cout << self->config_ << "\n";  // Safe
    }
};
```

The `shared_ptr` stored in `self` keeps the `SafeServer` alive even if all external shared pointers are released during the suspension. This is the canonical pattern for coroutine member functions.

### Lambda-as-Coroutine Lifetime Rules

There's another subtle variant: when the lambda itself is a coroutine. The coroutine frame stores a reference back to the closure object. If the closure is a temporary that gets destroyed, the frame is left with a dangling reference:

```cpp
task<void> broken() {
    int value = 42;

    // The lambda is a temporary - it's destroyed at the semicolon
    // But the coroutine frame holds a reference to the closure object
    auto t = [value]() -> task<int> {
        co_await some_op();
        co_return value;  // 'value' is part of the closure, which may be dead
    }();  // <-- temporary lambda destroyed here!

    co_await t;  // UB: coroutine frame references dead closure
}

task<void> fixed() {
    int value = 42;

    // Keep the lambda alive
    auto lambda = [value]() -> task<int> {
        co_await some_op();
        co_return value;
    };
    auto t = lambda();  // lambda outlives the coroutine

    co_await t;  // Safe
}
```

The difference is just giving the lambda a name so it lives until the end of the scope, rather than letting it be a temporary that vanishes at the semicolon. This is easy to miss when you're used to immediately-invoked lambdas in non-coroutine code.

### Checklist for Safe Lambda + Coroutine Use

The table below summarizes the rules that matter most. The common thread across all of them is: anything that crosses a `co_await` must be owned, not borrowed.

| Rule | Details |
| --- | --- |
| Never capture by reference across suspension | The referenced object may die during `co_await` |
| Keep lambda objects alive | If a lambda is a coroutine, store it in a named variable |
| Beware implicit `this` | Member-function coroutines hold a raw `this` pointer |
| Use `shared_from_this()` | Prevents object destruction during coroutine execution |
| Copy parameters early | Copy all parameters before the first `co_await` |

---

## Self-Assessment

### Q1: Identify the bug in this coroutine lambda

Look carefully at what `item` refers to and what happens after `co_await`. The range-for gives you a reference into the vector, and a reference doesn't move with the coroutine frame:

```cpp
task<void> process(std::vector<std::string> items) {
    for (auto& item : items) {
        // Bug: 'item' is a reference to an element in 'items'
        // which is a parameter - parameters are part of the coroutine frame,
        // BUT 'item' is a reference into the vector's heap memory.
        // If 'items' is reallocated or the coroutine is moved, this could dangle.
        auto worker = [&item]() -> task<void> {
            co_await do_work();
            std::cout << item << "\n";  // potential UB after suspension
        };
        co_await worker();
    }
}

// Fix: capture by value
task<void> process_fixed(std::vector<std::string> items) {
    for (const auto& item : items) {
        auto worker = [item_copy = item]() -> task<void> {
            co_await do_work();
            std::cout << item_copy << "\n";  // Safe: owned copy
        };
        co_await worker();
    }
}
```

### Q2: Why must the lambda object outlive its coroutine frame

A coroutine's parameters and captures reside in the **coroutine frame** allocated on the heap. But the coroutine frame for a **lambda coroutine** stores a reference or pointer to the closure object. If the lambda (closure) is a temporary that's destroyed, the coroutine frame's reference becomes dangling.

The fix is simply to give the lambda a name - a named local variable lives until the end of its enclosing scope, which is long enough:

```cpp
// BAD: temporary lambda
auto t = [x=42]() -> task<int> { co_return x; }();  // closure dies at ';'

// GOOD: named lambda outlives coroutine
auto lam = [x=42]() -> task<int> { co_return x; };
auto t = lam();  // lam lives until end of scope
```

### Q3: Use `shared_from_this` to safely access `this` in a member coroutine

The pattern here is canonical: capture `self` as the very first thing in the coroutine body, before any suspension point. That way, even if every external reference to the object is released while the coroutine is suspended, your `self` keeps it alive:

```cpp
#include <memory>
#include <iostream>

struct Connection : std::enable_shared_from_this<Connection> {
    std::string id_;

    task<void> handle_request() {
        // Prevent destruction by holding a shared_ptr to self
        auto self = shared_from_this();

        co_await read_request();
        // Even if the external shared_ptr is released during the co_await,
        // 'self' keeps the object alive
        std::cout << "Handling connection: " << self->id_ << "\n";

        co_await send_response();
        std::cout << "Done: " << self->id_ << "\n";
    }
};

void run() {
    auto conn = std::make_shared<Connection>(Connection{"conn-42"});
    auto t = conn->handle_request();
    conn.reset();  // release external reference - but coroutine still holds one
    t.resume();    // Safe: Connection kept alive by shared_ptr inside coroutine
}
```

---

## Notes

- Coroutine + lambda lifetime bugs are among the most common sources of UB in async C++ code - and static analysis tools (clang-tidy's `coro-dangling-ref`) can catch some of them, but not all.
- When in doubt, capture by value. The copy cost is almost always less than the cost of debugging a dangling reference at 3am.
- The `co_await` keyword is the danger zone - anything referenced across it must be owned or guaranteed alive through some other means.
- The rule about lambda temporaries being dangerous applies specifically to coroutine lambdas; for regular immediately-invoked lambdas it's fine.
