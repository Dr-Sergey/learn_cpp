# Use `std::function_ref` (C++26) as a Lightweight Non-Owning Callable Reference

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** [cppreference - std::function_ref](https://en.cppreference.com/w/cpp/utility/functional/function_ref)  

---

## Topic Overview

`std::function_ref`, defined in `<functional>`, is a non-owning, lightweight reference to a callable. It never allocates heap memory, never copies or moves the target callable, and has a fixed size - typically two pointers. It is the right tool when a function accepts a callback that it will only invoke during its own execution and never needs to store beyond that call.

The key distinction from `std::function` is ownership. `function_ref` does not own the callable. Think of it the same way you think about `std::string_view` for strings or `std::span` for arrays - it is a non-owning view into something that already exists elsewhere. Because there is no ownership, there is no allocation, no copying of the callable, and almost no overhead beyond the two-pointer representation.

| Property | `std::function` | `std::move_only_function` | `std::copyable_function` | `std::function_ref` |
| --- | --- | --- | --- | --- |
| Ownership | Owning | Owning | Owning | **Non-owning** |
| Heap allocation | Possible | Possible | Possible | **Never** |
| Copyable | Yes | No | Yes | Yes (trivially) |
| Const-correct | No | Yes | Yes | Yes |
| Size | ~32-64 bytes | ~32-64 bytes | ~32-64 bytes | **~16 bytes** |
| Use case | Store callback | Store move-only cb | Store copyable cb | **Pass callback** |

```cpp
Caller                          Callee
┌──────────────┐   function_ref   ┌─────────────────┐
│ lambda lives │ ─────────────>  │ invokes via ref  │
│ on stack     │   (2 pointers)  │ during call only │
└──────────────┘                  └─────────────────┘
     ^                                    │
     │           returns                  │
     └────────────────────────────────────┘
     lambda still alive — no dangling reference
```

**Dangling safety rule:** Never store a `function_ref` beyond the scope where the referenced callable lives. It is a parameter type, not a member type. The lambda on the caller's stack must outlive the `function_ref` that refers to it - and since `function_ref` is meant for synchronous use, this is a straightforward constraint to follow.

---

## Self-Assessment

### Q1: When should you use `std::function_ref` instead of `std::function` or a template parameter

There are three common ways to accept a callback in C++, and each fits a different situation. The example below shows all three side by side so you can compare them directly. The comments in the decision table in the code explain when to reach for each one.

<!-- compile: needs C++26 library support for `std::function_ref` (not in the CI standard library yet) -->
```cpp
#include <functional>
#include <vector>
#include <iostream>
#include <string>

// BEST: template parameter — zero overhead, but not type-erasable
template <typename F>
void for_each_template(const std::vector<int>& v, F&& f) {
    for (int x : v) f(x);
}

// GOOD: function_ref — type-erased, zero allocation, non-owning
void for_each_ref(const std::vector<int>& v,
                  std::function_ref<void(int) const> f) {
    for (int x : v) f(x);
}
// Advantages over template:
// - Not a template -> can be in .cpp file, no header bloat
// - Can be virtual, stored in vtable, used in C API callbacks
// - Fixed ABI — no template instantiation per lambda

// AVOID: std::function for non-stored callbacks — unnecessary overhead
void for_each_function(const std::vector<int>& v,
                       const std::function<void(int)>& f) {
    for (int x : v) f(x);
}
// Problems: potential heap allocation, type-erased copy, not const-correct

// Decision table:
// | Scenario                        | Use                    |
// |---------------------------------|------------------------|
// | Header-only, max performance    | Template parameter     |
// | ABI-stable, non-stored callback | std::function_ref      |
// | Stored callback, copyable       | std::copyable_function |
// | Stored callback, move-only      | std::move_only_function|

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    // All three work the same way from the caller's perspective
    auto print = [](int x) { std::cout << x << ' '; };

    for_each_template(data, print);   std::cout << '\n';
    for_each_ref(data, print);        std::cout << '\n';
    for_each_function(data, print);   std::cout << '\n';

    // function_ref works with any callable: lambda, function pointer, functor
    for_each_ref(data, [](int x) { std::cout << x * x << ' '; });
    std::cout << '\n';
}
```

If you are writing a library function that must live in a `.cpp` file (for compilation speed or ABI stability), or if the function will appear in a virtual table, `function_ref` is your friend. Template parameters do not survive those constraints.

---

### Q2: How does `std::function_ref` enforce const-correctness and `noexcept`

One of the underappreciated strengths of `function_ref` over plain `std::function` is that the qualifiers in the type signature actually mean something. A `function_ref<void(int) const>` will refuse to bind a mutable lambda at compile time. A `function_ref<void(int) const noexcept>` will additionally refuse to bind a lambda that might throw. The compiler enforces your API contract for you.

<!-- compile: needs C++26 library support for `std::function_ref` (not in the CI standard library yet) -->
```cpp
#include <functional>
#include <iostream>

// Const-correct callback API
struct Processor {
    // const: the callback must not mutate its captures
    void process(int value,
                 std::function_ref<void(int) const> on_result) const {
        on_result(value * 2);
    }

    // Non-const: the callback MAY mutate its captures
    void accumulate(int value,
                    std::function_ref<void(int)> on_value) {
        on_value(value);
    }

    // noexcept: the callback must be noexcept
    void process_safe(int value,
                      std::function_ref<void(int) const noexcept> on_result) const noexcept {
        on_result(value * 2);
    }
};

int main() {
    Processor p;

    // const callback — OK
    p.process(21, [](int r) { std::cout << "Result: " << r << '\n'; });

    // Mutable callback with accumulate
    int sum = 0;
    auto accumulator = [&sum](int x) mutable { sum += x; };
    p.accumulate(10, accumulator);
    p.accumulate(20, accumulator);
    std::cout << "Sum: " << sum << '\n';  // 30

    // noexcept callback
    p.process_safe(5, [](int r) noexcept {
        // guaranteed not to throw
    });

    // ERROR: mutable lambda cannot bind to const function_ref
    // int counter = 0;
    // p.process(1, [counter]() mutable { ++counter; });

    // ERROR: throwing lambda cannot bind to noexcept function_ref
    // p.process_safe(1, [](int) { throw 42; });
}
```

**Key insight:** `function_ref<void(int) const noexcept>` enforces both const-correctness and exception guarantees at the API boundary - the compiler rejects non-conforming callables. This is not true of `std::function`, where `const` on the wrapper does not mean the stored callable is called as `const`.

---

### Q3: What are the dangling pitfalls and how does `function_ref` compare in size and performance

The dangling rule is simple but critical: a `function_ref` is only safe for the duration of the call that receives it. The example below shows a bad event system design and a correct one, followed by a micro-benchmark that illustrates the overhead difference.

<!-- compile: needs C++26 library support for `std::copyable_function` (not in the CI standard library yet) -->
```cpp
#include <functional>
#include <iostream>
#include <chrono>
#include <vector>
#include <cstddef>
#include <utility>

// DANGER: storing function_ref leads to dangling
class Bad_EventSystem {
    // DO NOT store function_ref as a member!
    // std::function_ref<void()> handler_;  // Dangling after caller returns
};

// CORRECT: use function_ref only as parameter
class Good_EventSystem {
    // Store an owning wrapper for persistence
    std::vector<std::copyable_function<void() const>> handlers_;

public:
    void subscribe(std::copyable_function<void() const> h) {
        handlers_.push_back(std::move(h));
    }

    // Use function_ref for one-shot visitation (no storage)
    void visit_all(std::function_ref<void(std::size_t,
                   const std::copyable_function<void() const>&) const> visitor) const {
        for (std::size_t i = 0; i < handlers_.size(); ++i) {
            visitor(i, handlers_[i]);
        }
    }
};

// Size comparison
static_assert(sizeof(std::function_ref<void()>) <= 2 * sizeof(void*),
    "function_ref should be at most 2 pointers");
// std::function is typically 32-64 bytes

// Performance: function_ref has no allocation overhead
void benchmark_callback_overhead() {
    constexpr int N = 10'000'000;
    volatile int sink = 0;

    auto callback = [&sink](int x) noexcept { sink = x; };

    // function_ref — no allocation, direct call
    auto ref_call = [&]() {
        std::function_ref<void(int) const noexcept> ref = callback;
        for (int i = 0; i < N; ++i) ref(i);
    };

    // std::function — potential allocation, virtual dispatch
    auto func_call = [&]() {
        std::function<void(int)> func = callback;
        for (int i = 0; i < N; ++i) func(i);
    };

    using Clock = std::chrono::high_resolution_clock;

    auto t0 = Clock::now();
    ref_call();
    auto t1 = Clock::now();
    func_call();
    auto t2 = Clock::now();

    auto ref_ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    auto func_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();

    std::cout << "function_ref: " << ref_ms << " ms\n";
    std::cout << "function:     " << func_ms << " ms\n";
}

int main() {
    Good_EventSystem sys;
    sys.subscribe([]() { std::cout << "Handler A\n"; });
    sys.subscribe([]() { std::cout << "Handler B\n"; });

    sys.visit_all([](std::size_t idx, const auto& h) {
        std::cout << "Handler " << idx << ": ";
        h();
    });

    benchmark_callback_overhead();
}
```

The `Good_EventSystem` demonstrates the correct split: when you need to keep a callback around after the call returns, use `std::copyable_function` as the stored type. When you need to iterate over stored callbacks and let a caller inspect them, accept a `function_ref` as the visitor parameter - it is non-owning and only needs to live for the duration of `visit_all`.

---

## Notes

- `std::function_ref` is in `<functional>` (C++26). It is **non-owning** - never store it as a data member.
- Typical size: 2 pointers (16 bytes on 64-bit). No heap allocation ever.
- Supports qualifier combinations: `R(Args...)`, `R(Args...) const`, `R(Args...) noexcept`, `R(Args...) const noexcept`.
- Use it as a **parameter type** for callbacks that are invoked synchronously and not stored beyond the call.
- Analogy: `function_ref` is to `copyable_function`/`function` as `string_view` is to `string`.
- It binds to any callable: lambdas, function pointers, functors, member function pointers (via wrapper).
- **Dangling risk:** The referenced callable must outlive the `function_ref`. Never return or store a `function_ref` to a temporary.
- Feature-test macro: `__cpp_lib_function_ref >= 202306L`.
