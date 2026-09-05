# Use `std::copyable_function` (C++26) as a `std::function` Replacement

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** [cppreference — std::copyable_function](https://en.cppreference.com/w/cpp/utility/functional/copyable_function)  

---

## Topic Overview

`std::copyable_function`, defined in `<functional>`, is a drop-in improvement over `std::function` that fixes its long-standing const-correctness bug. The bug: `std::function<R(Args...)>`'s `operator()` is always `const`, yet it can wrap mutable callables - which means you can mutate state through a `const` reference without any compiler warning. That is a silent violation of const-correctness that has existed since C++11.

`std::copyable_function` fixes this by encoding the `const` and `noexcept` qualifiers directly in the type signature, the same way you write them on a member function:

The key difference: `std::copyable_function<int(int) const>` has a `const operator()` and only accepts const-invocable callables. Without `const`, it has a non-const `operator()` and correctly prevents invocation through a `const` reference.

| Feature | `std::function` | `std::copyable_function` |
| --- | --- | --- |
| Const-correct `operator()` | No - always const | Yes - qualifiers in signature |
| `noexcept` propagation | No | Yes - `R(Args...) noexcept` |
| Move-only callable support | No | No (see `std::move_only_function`) |
| Copyable requirement | Yes | Yes |
| Small-buffer optimization | Implementation-defined | Implementation-defined |
| Null state | Yes (throws on call) | Yes (throws on call) |

Here is a quick read of what the different signature forms mean:

```cpp
  std::function<void()>                   - operator() is const (broken)
  std::copyable_function<void()>          - operator() is non-const
  std::copyable_function<void() const>    - operator() is const (correct)
  std::copyable_function<void() noexcept> - operator() is non-const noexcept
```

The C++26 callable type hierarchy, from most restrictive to most permissive:

```cpp
+--------------------------------------+
| std::move_only_function (C++23)      | <- move-only, no copy
+--------------------------------------+
| std::copyable_function (C++26)       | <- copyable, const-correct
+--------------------------------------+
| std::function_ref (C++26)            | <- non-owning, lightweight
+--------------------------------------+
```

---

## Self-Assessment

### Q1: What is the const-correctness bug in `std::function` that `std::copyable_function` fixes

The bug is subtle and easy to miss. A `const std::function<void()>&` still lets you call the underlying mutable lambda - because `operator()` is declared `const` on `std::function` unconditionally. `std::copyable_function` makes the const-ness of `operator()` match what you wrote in the type signature:

<!-- compile: needs C++26 library support for `std::copyable_function` (not in the CI standard library yet) -->
```cpp
#include <functional>
#include <iostream>

// THE BUG in std::function
void demonstrate_function_bug() {
    int counter = 0;
    std::function<void()> f = [counter]() mutable {
        ++counter;  // mutates the captured copy
        std::cout << "counter = " << counter << '\n';
    };

    // f is const-invocable even though the lambda is mutable!
    const auto& cf = f;
    cf();  // COMPILES! Modifies state through const reference
    cf();  // counter = 2 - const violation
}

// THE FIX with std::copyable_function
void demonstrate_copyable_function() {
    int counter = 0;

    // Non-const signature: callable is mutable
    std::copyable_function<void()> f = [counter]() mutable {
        ++counter;
        std::cout << "counter = " << counter << '\n';
    };

    f();  // OK: non-const call

    const auto& cf = f;
    // cf();  // ERROR: operator() is non-const, cannot call through const ref

    // Const signature: only accepts const-invocable callables
    std::copyable_function<void() const> g = [](){
        std::cout << "const-correct\n";
    };

    const auto& cg = g;
    cg();  // OK: operator() is const, callable is const-invocable

    // This would NOT compile:
    // std::copyable_function<void() const> bad = [counter]() mutable {
    //     ++counter;  // ERROR: mutable lambda cannot be stored in const signature
    // };
}

int main() {
    demonstrate_function_bug();
    demonstrate_copyable_function();
}
```

`std::function`'s `operator()` being unconditionally `const` is a design defect from C++11. `copyable_function` fixes it by making the const qualifier explicit in the type signature - the type itself tells you whether calling through a `const` reference is allowed.

---

### Q2: How do you use `std::copyable_function` with noexcept and in callback-based APIs

The `noexcept` qualifier in the signature is enforced at construction time: if you try to store a potentially-throwing callable in a `noexcept`-qualified `copyable_function`, the compiler rejects it. This is genuinely useful for safety-critical or lock-held callbacks where a thrown exception would be catastrophic:

<!-- compile: needs C++26 library support for `std::copyable_function` (not in the CI standard library yet) -->
```cpp
#include <functional>
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>
#include <ranges>
#include <string_view>
#include <utility>

// noexcept-aware callback storage
class EventBus {
public:
    // Only accepts noexcept handlers - crash-safe guarantee
    using Handler = std::copyable_function<void(std::string_view) const noexcept>;

    void subscribe(Handler h) {
        handlers_.push_back(std::move(h));
    }

    void publish(std::string_view event) const {
        for (const auto& h : handlers_) {
            h(event);  // guaranteed noexcept - no try/catch needed
        }
    }

private:
    std::vector<Handler> handlers_;
};

// Const-correct predicate storage
template <typename T>
class FilterChain {
    // const: predicates don't mutate state
    std::vector<std::copyable_function<bool(const T&) const>> filters_;

public:
    void add_filter(std::copyable_function<bool(const T&) const> f) {
        filters_.push_back(std::move(f));
    }

    bool passes(const T& val) const {
        return std::ranges::all_of(filters_, [&](const auto& f) {
            return f(val);
        });
    }
};

int main() {
    // Event bus with noexcept handlers
    EventBus bus;
    bus.subscribe([](std::string_view ev) noexcept {
        // This handler is guaranteed not to throw
        // If we tried a throwing lambda, it wouldn't compile
    });
    bus.publish("user.login");

    // Filter chain with const-correct predicates
    FilterChain<int> chain;
    chain.add_filter([](const int& x) { return x > 0; });
    chain.add_filter([](const int& x) { return x < 100; });
    chain.add_filter([](const int& x) { return x % 2 == 0; });

    std::cout << std::boolalpha;
    std::cout << chain.passes(42) << '\n';   // true
    std::cout << chain.passes(-1) << '\n';   // false
    std::cout << chain.passes(101) << '\n';  // false
}
```

---

### Q3: How does `std::copyable_function` compare with `std::move_only_function` and `std::function`

This question is about picking the right tool. `std::function` is the broadest but has the const bug. `std::copyable_function` fixes the bug but still requires copyable callables. `std::move_only_function` (C++23) handles move-only callables like those capturing `unique_ptr`. For non-owning use, `std::function_ref` (C++26) avoids allocation entirely:

<!-- compile: needs C++26 library support for `std::copyable_function` (not in the CI standard library yet) -->
```cpp
#include <functional>
#include <memory>
#include <iostream>
#include <vector>
#include <string>
#include <utility>

// Comparison: three callable wrappers

struct CopyableCallable {
    std::shared_ptr<int> state = std::make_shared<int>(0);
    void operator()() { ++(*state); std::cout << *state << '\n'; }
};

struct MoveOnlyCallable {
    std::unique_ptr<int> state = std::make_unique<int>(0);
    void operator()() { ++(*state); std::cout << *state << '\n'; }
};

void compare_wrappers() {
    // std::function - copyable, NOT const-correct
    std::function<void()> f1 = CopyableCallable{};
    auto f1_copy = f1;  // OK: copyable
    // const_cast-like invocation is possible (bug)

    // std::copyable_function - copyable, const-correct
    std::copyable_function<void()> f2 = CopyableCallable{};
    auto f2_copy = f2;  // OK: copyable
    // const auto& cf2 = f2; cf2(); // ERROR: non-const signature

    // std::move_only_function - move-only, const-correct
    std::move_only_function<void()> f3 = MoveOnlyCallable{};
    // auto f3_copy = f3;  // ERROR: not copyable
    auto f3_moved = std::move(f3);  // OK: movable

    // std::copyable_function CANNOT hold move-only callables:
    // std::copyable_function<void()> bad = MoveOnlyCallable{};  // ERROR
}

// Migration guide:
//
// | Old code                     | New code                                |
// |------------------------------|-----------------------------------------|
// | std::function<void()>        | std::copyable_function<void() const>    |
// | std::function (mutable)      | std::copyable_function<void()>          |
// | std::function + unique_ptr   | std::move_only_function<void()>         |
// | const std::function& param   | std::function_ref<void() const>         |

// Practical: callback registry that needs copyability
class Scheduler {
    struct Task {
        std::string name;
        std::copyable_function<void() const> action;
    };
    std::vector<Task> tasks_;

public:
    void add(std::string name, std::copyable_function<void() const> action) {
        tasks_.push_back({std::move(name), std::move(action)});
    }

    // Can copy the entire scheduler (tasks are copyable)
    void run_all() const {
        for (const auto& [name, action] : tasks_) {
            std::cout << "Running: " << name << '\n';
            action();  // const call - guaranteed safe
        }
    }
};

int main() {
    compare_wrappers();

    Scheduler sched;
    sched.add("greet", []() { std::cout << "Hello!\n"; });
    sched.add("count", []() { std::cout << "1 2 3\n"; });

    auto sched_copy = sched;  // deep copy of all tasks
    sched_copy.run_all();
}
```

The migration guide comment in the middle is worth bookmarking. The general rule of thumb: prefer `copyable_function<... const>` for anything that does not mutate state, plain `copyable_function<...>` for mutable callables, and `move_only_function` whenever `unique_ptr` or other move-only resources are captured.

---

## Notes

- `std::copyable_function` is in `<functional>` (C++26). It supersedes `std::function` for new code.
- The signature encodes qualifiers: `R(Args...)`, `R(Args...) const`, `R(Args...) noexcept`, `R(Args...) const noexcept`.
- Unlike `std::function`, the `const` qualifier on `operator()` must match the stored callable - no silent const violations.
- `std::copyable_function` requires copyable callables. For move-only callables, use `std::move_only_function` (C++23).
- For non-owning callable references with no allocation, use `std::function_ref` (C++26).
- Migration path: replace `std::function<R(Args...)>` with `std::copyable_function<R(Args...) const>` for const-correct semantics.
- Feature-test macro: `__cpp_lib_copyable_function >= 202306L`.
