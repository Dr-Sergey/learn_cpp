# Design callback interfaces using std::function, templates, and type erasure

**Category:** OOP Design  
**Standard:** C++11  

---

## Topic Overview

There are several ways to accept a callable in C++, and the right choice depends on whether you need to store it, how important performance is, and whether the caller's type needs to be known at compile time. Here's the quick summary:

| Approach | Overhead | Flexibility | Inline-able? |
| --- | --- | --- | --- |
| Function pointer | Zero | C-style, no state | Yes |
| Template param | Zero | Compile-time binding | Yes |
| `std::function` | Heap alloc possible | Any callable | No |
| Type-erased (custom) | Configurable (SBO) | Any callable | Possible |

### Decision Guide

When you're not sure which to reach for, walk through this decision tree:

```cpp
Do you need to store the callback?     No  -> Template parameter
                                        Yes ->
Is the callback type known at compile?  Yes -> Template member
                                        No  -> std::function or type erasure
Is performance critical?                Yes -> Custom type erasure with SBO
                                        No  -> std::function
```

---

## Self-Assessment

### Q1: Compare all callback approaches with working examples

Let's look at all three main approaches side by side. Notice how the template version with a lambda gets inlined completely, while `std::function` trades that inlinability for the ability to store callbacks of different types in the same container.

**Answer:**

```cpp
#include <functional>
#include <iostream>
#include <vector>

// 1. Function pointer (zero overhead, no state)
using Comparator = bool(*)(int, int);
void sort_c_style(int* arr, size_t n, Comparator cmp) {
    // Uses cmp as function pointer
    for (size_t i = 0; i < n; ++i)
        for (size_t j = i+1; j < n; ++j)
            if (cmp(arr[j], arr[i])) std::swap(arr[i], arr[j]);
}

// 2. Template parameter (zero overhead, inlineable)
template<typename Callback>
void for_each_tmpl(const std::vector<int>& v, Callback cb) {
    for (int x : v) cb(x);  // cb inlined if lambda/functor
}

// 3. std::function (flexible, may allocate)
class EventEmitter {
    std::vector<std::function<void(int)>> listeners_;
public:
    void on(std::function<void(int)> cb) {
        listeners_.push_back(std::move(cb));
    }
    void emit(int event_id) {
        for (auto& cb : listeners_) cb(event_id);
    }
};

int main() {
    // Function pointer
    int arr[] = {3, 1, 2};
    sort_c_style(arr, 3, [](int a, int b) { return a < b; });  // Stateless lambda -> fn ptr

    // Template: zero-overhead
    std::vector<int> v{1, 2, 3};
    int sum = 0;
    for_each_tmpl(v, [&sum](int x) { sum += x; });  // Inlined
    std::cout << "Sum: " << sum << "\n";

    // std::function: stored callback
    EventEmitter emitter;
    emitter.on([](int id) { std::cout << "Event: " << id << "\n"; });
    emitter.on([](int id) { std::cout << "Log: " << id << "\n"; });
    emitter.emit(42);
    return 0;
}
```

The function pointer approach only works for stateless lambdas - the moment your lambda captures a variable, the compiler can't reduce it to a plain function pointer. The template approach handles both stateless and capturing lambdas with zero cost, but it requires the type to be known at compile time. `std::function` is the most general but may allocate on the heap if the callable is larger than its internal small-buffer optimization threshold.

### Q2: Build a type-erased callback with SBO (no heap allocation)

`std::function` is convenient but its allocation behavior is implementation-defined. If you're on a hot path and your callable is small, you can write a type-erased function wrapper that guarantees no heap allocation by storing the callable directly in an aligned internal buffer. This is called Small Buffer Optimization (SBO).

**Answer:**

```cpp
#include <cstddef>
#include <new>
#include <utility>
#include <iostream>

// Compact type-erased callable with Small Buffer Optimization
template<typename Signature>
class SmallFunction;  // Primary template

template<typename R, typename... Args>
class SmallFunction<R(Args...)> {
    static constexpr size_t BufSize = 32;

    using InvokeFn = R(*)(void*, Args...);
    using DestroyFn = void(*)(void*);

    alignas(std::max_align_t) char buffer_[BufSize];
    InvokeFn invoke_ = nullptr;
    DestroyFn destroy_ = nullptr;

public:
    SmallFunction() = default;

    template<typename F>
    SmallFunction(F f) {
        static_assert(sizeof(F) <= BufSize, "Callable too large for SBO");
        new (buffer_) F(std::move(f));
        invoke_ = [](void* buf, Args... args) -> R {
            return (*static_cast<F*>(buf))(std::forward<Args>(args)...);
        };
        destroy_ = [](void* buf) {
            static_cast<F*>(buf)->~F();
        };
    }

    ~SmallFunction() { if (destroy_) destroy_(buffer_); }

    SmallFunction(const SmallFunction&) = delete;  // Simplified: move-only
    SmallFunction(SmallFunction&& o) noexcept = default;

    R operator()(Args... args) {
        return invoke_(buffer_, std::forward<Args>(args)...);
    }

    explicit operator bool() const { return invoke_ != nullptr; }
};

int main() {
    SmallFunction<int(int, int)> add = [](int a, int b) { return a + b; };
    std::cout << add(3, 4) << "\n";  // 7

    int factor = 10;
    SmallFunction<int(int)> mul = [factor](int x) { return x * factor; };
    std::cout << mul(5) << "\n";     // 50
    // No heap allocation! All fits in 32-byte buffer
    return 0;
}
```

The key trick is placement new: `new (buffer_) F(std::move(f))` constructs the callable directly into the fixed-size character array inside the object. The `invoke_` function pointer holds a lambda that knows the concrete type `F` and casts the `void*` buffer back. The `static_assert` on `sizeof(F) <= BufSize` gives you a clean compile-time error if someone tries to store a callable that doesn't fit.

### Q3: Show a real-world async callback pattern with cancellation

A common need in async code is the ability to cancel a pending callback - for example, if the UI that registered the callback has been destroyed. A shared cancellation token lets the caller signal "don't bother" without having to reach into the callback mechanism directly.

**Answer:**

```cpp
#include <functional>
#include <memory>
#include <iostream>
#include <vector>

// Cancellation token
class CancelToken {
    std::shared_ptr<bool> cancelled_ = std::make_shared<bool>(false);
public:
    void cancel() { *cancelled_ = true; }
    bool is_cancelled() const { return *cancelled_; }
};

// Async-style callback with lifetime management
template<typename T>
class Future {
public:
    using Callback = std::function<void(T)>;
    using ErrorCb = std::function<void(std::string)>;

    Future& then(Callback cb) {
        success_ = std::move(cb);
        return *this;
    }
    Future& on_error(ErrorCb cb) {
        error_ = std::move(cb);
        return *this;
    }
    Future& with_cancel(CancelToken token) {
        cancel_ = std::move(token);
        return *this;
    }

    // Simulate resolution
    void resolve(T value) {
        if (cancel_.is_cancelled()) return;  // Dropped
        if (success_) success_(std::move(value));
    }
    void reject(std::string error) {
        if (cancel_.is_cancelled()) return;
        if (error_) error_(std::move(error));
    }

private:
    Callback success_;
    ErrorCb error_;
    CancelToken cancel_;
};

int main() {
    CancelToken token;

    Future<int> f;
    f.then([](int v) { std::cout << "Result: " << v << "\n"; })
     .on_error([](std::string e) { std::cout << "Error: " << e << "\n"; })
     .with_cancel(token);

    f.resolve(42);       // "Result: 42"

    token.cancel();
    f.resolve(99);       // Silently dropped
    return 0;
}
```

The `CancelToken` uses a `shared_ptr<bool>` so both the token holder and the `Future` share the same cancelled flag. When `cancel()` is called on any copy of the token, the next `resolve()` or `reject()` sees the flag and exits early. This is a lightweight substitute for more elaborate cancellation frameworks.

---

## Notes

- **Template callbacks** for hot paths: zero overhead, inlined by the compiler.
- **`std::function`** for stored callbacks: flexible but may heap-allocate (has SBO for small callables in most implementations).
- Custom type erasure gives you control over the allocation strategy, SBO size, and move-only semantics.
- Always accept callbacks by template or `std::function`; avoid a virtual interface unless polymorphism is genuinely needed.
- `std::move_only_function` (C++23) is a better fit than `std::function` for move-only callables.
- Watch for dangling captures: if a callback captures references, make sure the referenced objects outlive the callback.
