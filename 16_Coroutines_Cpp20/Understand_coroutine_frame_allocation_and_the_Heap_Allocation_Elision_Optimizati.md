# Understand coroutine frame allocation and the Heap Allocation Elision Optimization (HALO)

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

Every coroutine allocates a **coroutine frame** on the heap to store its local variables, parameters, promise object, and suspension point. **HALO** (Heap Allocation Elision Optimization) allows the compiler to eliminate this allocation when it can prove the frame's lifetime fits within the caller's scope.

The reason the frame goes on the heap by default - rather than the stack - is that coroutines can outlive the call that created them. The calling function returns, its stack frame is gone, but the coroutine might still be suspended and waiting to be resumed. So the frame needs to live somewhere the caller's return doesn't destroy. That "somewhere" is the heap.

### Coroutine Frame Contents

When you write a coroutine, the compiler generates a frame object that holds everything the coroutine needs to resume from any of its suspension points:

```cpp
+------------------------------------+
|   Coroutine Frame (heap)           |
|                                    |
|   +- promise_type object           |
|   +- function parameters (copies)  |
|   +- local variables               |
|   +- suspension point index        |
|   +- compiler bookkeeping          |
|                                    |
+------------------------------------+
```

The suspension point index is how the compiler knows where to jump when `handle.resume()` is called - it's essentially a generated state machine with a switch on that index.

### HALO Conditions

HALO is a permitted compiler optimisation (similar to RVO for return values). The compiler is allowed to place the frame on the caller's stack instead of the heap when certain conditions hold. If any of these conditions is violated, the heap allocation is mandatory:

| Condition | HALO possible? |
| --- | --- |
| Coroutine lifetime is within caller scope | Yes |
| Coroutine handle escapes (e.g., stored in container) | No |
| Coroutine resumed on different thread | No |
| Compiler can see entire coroutine and caller | Yes (needs inlining) |
| Virtual function returning coroutine | No (opaque call) |

---

## Self-Assessment

### Q1: Explain when the compiler can elide the coroutine frame heap allocation

The compiler can apply HALO when it can prove all three of these things at once. The reason all three matter is that any one violation means the frame might need to outlive the caller's stack frame, and the compiler can't take that risk:

1. **The coroutine's lifetime does not exceed the caller's scope.** The frame can be placed on the caller's stack.

2. **The coroutine handle does not escape.** If the handle is stored in a global, captured by another thread, or returned to a higher scope, HALO is impossible.

3. **The compiler has full visibility.** The coroutine body and all callers must be visible (same TU or via LTO/inlining).

Here is a HALO-eligible example. The generator is created, used entirely, and destroyed within `main` - the handle never goes anywhere else:

```cpp
#include <coroutine>
#include <iostream>

struct Generator {
    struct promise_type {
        int current;
        Generator get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(int v) { current = v; return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~Generator() { if (handle) handle.destroy(); }
};

Generator count_to(int n) {
    for (int i = 1; i <= n; ++i)
        co_yield i;
}

int main() {
    // HALO CAN apply here: the generator is created, used, and destroyed
    // entirely within main(). The handle never escapes.
    auto gen = count_to(3);
    while (!gen.handle.done()) {
        gen.handle.resume();
        if (!gen.handle.done())
            std::cout << gen.handle.promise().current << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// 1 2 3
```

**HALO applies here because:** `gen` is local, the handle stays in `main()`'s scope, and the compiler can see both `count_to` and `main()`.

### Q2: Show a case where HALO applies: the coroutine doesn't outlive the caller's frame

This example puts both cases side by side so you can see exactly what breaks HALO. The `sum_first_n` function is HALO-eligible because the generator never leaves that function. The `halo_prevented` function kills HALO the moment it pushes the handle into `global_handles`:

```cpp
#include <coroutine>
#include <iostream>
#include <vector>

struct SimpleGen {
    struct promise_type {
        int val;
        SimpleGen get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(int v) { val = v; return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~SimpleGen() { if (handle) handle.destroy(); }
};

// HALO-friendly: generator is consumed inline
int sum_first_n(int n) {
    SimpleGen gen = [](int limit) -> SimpleGen {
        for (int i = 1; i <= limit; ++i)
            co_yield i;
    }(n);

    int total = 0;
    while (!gen.handle.done()) {
        gen.handle.resume();
        if (!gen.handle.done())
            total += gen.handle.promise().val;
    }
    return total;
    // gen destroyed here -- never escaped this scope
}

// HALO-unfriendly: handle escapes to a container
std::vector<std::coroutine_handle<>> global_handles;

void halo_prevented() {
    SimpleGen gen = [](int n) -> SimpleGen {
        for (int i = 0; i < n; ++i)
            co_yield i;
    }(5);

    global_handles.push_back(gen.handle);  // handle escapes!
    // Compiler cannot prove lifetime -> must heap-allocate
}

int main() {
    std::cout << "Sum 1..5 = " << sum_first_n(5) << '\n';
}
// Expected output:
// Sum 1..5 = 15
```

### Q3: Measure the allocation overhead of coroutines vs callbacks when HALO does not apply

This benchmark quantifies the cost you pay for heap-allocated coroutine frames. The callback version never allocates a frame; the coroutine version allocates and frees one for every iteration. The ~10x difference is almost entirely allocation overhead, not the suspension mechanism itself:

```cpp
#include <chrono>
#include <coroutine>
#include <functional>
#include <iostream>

// Coroutine version (heap-allocated frame)
struct CoroTask {
    struct promise_type {
        int result;
        CoroTask get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(int v) { result = v; }
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~CoroTask() { if (handle) handle.destroy(); }
};

CoroTask coro_add(int a, int b) {
    co_return a + b;
}

// Callback version (no allocation)
void callback_add(int a, int b, std::function<void(int)> cb) {
    cb(a + b);
}

int main() {
    constexpr int N = 1'000'000;
    int sink = 0;

    // Benchmark coroutines
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        auto t = coro_add(i, i + 1);
        t.handle.resume();
        sink += t.handle.promise().result;
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark callbacks
    auto t3 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        callback_add(i, i + 1, [&sink](int r) { sink += r; });
    }
    auto t4 = std::chrono::high_resolution_clock::now();

    auto coro_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto cb_ms = std::chrono::duration<double, std::milli>(t4 - t3).count();

    std::cout << "Coroutine: " << coro_ms << " ms\n";
    std::cout << "Callback:  " << cb_ms << " ms\n";
    std::cout << "Overhead:  " << (coro_ms / cb_ms) << "x\n";
    std::cout << "(sink=" << sink << ")\n";  // prevent optimization
}
// Typical output (varies by platform):
// Coroutine: ~50 ms
// Callback:  ~5 ms
// Overhead:  ~10x (without HALO)
```

**When HALO applies, the overhead drops to near-zero.** The frame is on the stack, no `operator new` is called.

If you want to verify empirically whether HALO fired, you can track allocations by overriding `operator new` in the promise type. If `alloc_count` is greater than zero after the run, HALO was not applied:

```cpp
struct AllocTrackingPromise {
    static int alloc_count;
    void* operator new(std::size_t sz) {
        ++alloc_count;
        return ::operator new(sz);
    }
    void operator delete(void* p) { ::operator delete(p); }
    // ... rest of promise_type
};
// If alloc_count > 0 after running, HALO was NOT applied.
```

---

## Notes

- HALO is not guaranteed by the standard - it's a permitted optimization (similar to RVO/NRVO). You cannot rely on it for correctness, only for performance.
- Clang has the best HALO support; GCC and MSVC are improving with each release.
- Use `-O2` or higher for HALO to kick in. Debug builds always heap-allocate because the optimiser is disabled.
- You can provide a custom `operator new`/`operator delete` in the promise type to use pool allocators, which is the practical middle ground when HALO doesn't apply but you still want to avoid the general-purpose allocator's overhead.
- The `promise_type::get_return_object_on_allocation_failure()` static method lets you handle allocation failures gracefully instead of throwing `std::bad_alloc`.
