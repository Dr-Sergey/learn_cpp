# Customize Coroutine Frame Allocation with promise_type::operator new

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** [cppreference — Coroutines (allocation)](https://en.cppreference.com/w/cpp/language/coroutines#Dynamic_allocation)  

---

## Topic Overview

Every coroutine needs a **coroutine frame** - a compiler-generated object that holds the promise, local variables, suspension-point bookkeeping, and copies/moves of function parameters. By default the compiler allocates this frame with `::operator new`. You can override this by defining `promise_type::operator new` (and optionally `operator delete`), giving you full control over where coroutine frames live.

The reason this matters is that coroutines can be very short-lived and numerous - a high-performance network server might create thousands of them per second. Each default `::operator new` call hits the global heap allocator, which means lock contention, fragmentation, and cache misses. By routing allocations through a monotonic arena or a PMR resource, you can make coroutine creation essentially free.

The compiler first attempts to call a `promise_type::operator new` overload that takes the **coroutine's parameters** as extra arguments (after the required `std::size_t`). If no such overload exists, it falls back to `promise_type::operator new(std::size_t)`, then to global `::operator new`. This parameter-forwarding mechanism enables **stateful allocators**: the coroutine function receives an allocator argument, and the promise's `operator new` uses it.

**Heap Allocation eLision Optimization (HALO):** compilers may elide the dynamic allocation entirely when they can prove the coroutine's lifetime is enclosed in the caller's frame. This typically requires the coroutine to be **inlined** and the handle to never escape. When HALO fires, `operator new` is never called - even if you defined it. As of 2025, HALO is applied inconsistently across compilers; always measure.

Here is a summary of how the compiler resolves where to allocate the frame:

| Allocation path | Resolves to |
| --- | --- |
| `promise_type::operator new(size, args...)` exists | Called with coroutine parameters forwarded |
| Only `promise_type::operator new(size)` exists | Called with frame size only |
| Neither exists | `::operator new(size)` |
| HALO applies | No allocation at all |

The decision tree looks like this - read it top to bottom to understand which path the compiler takes:

```cpp
Coroutine call
    │
    ├──[HALO possible?]── yes ──> frame on caller's stack (zero alloc)
    │
    └──[HALO not possible]──> promise_type::operator new(size, args...)
                                  │
                                  ├── found ──> custom allocator
                                  └── not found ──> ::operator new(size)
```

---

## Self-Assessment

### Q1: Implement `promise_type::operator new/delete` using a monotonic arena allocator. Show the allocation being routed through the arena

A monotonic arena is the simplest and most cache-friendly allocator you can use for coroutines. It bumps a pointer forward on each allocation and reclaims everything at once with a single reset. Here is what that looks like wired into a promise type:

```cpp
#include <coroutine>
#include <cstddef>
#include <cstdint>
#include <cstdio>
#include <new>
#include <exception>
#include <memory>

// Simple monotonic arena — no individual deallocation
class Arena {
public:
    explicit Arena(std::size_t capacity)
        : buffer_(new std::byte[capacity]), capacity_(capacity) {}
    ~Arena() { delete[] buffer_; }

    void* allocate(std::size_t size, std::size_t align = alignof(std::max_align_t)) {
        std::size_t space = capacity_ - offset_;
        void* ptr = buffer_ + offset_;
        if (std::align(align, size, ptr, space)) {
            offset_ = capacity_ - space + size;
            return ptr;
        }
        throw std::bad_alloc{};
    }

    void reset() { offset_ = 0; }

private:
    std::byte* buffer_;
    std::size_t capacity_;
    std::size_t offset_ = 0;
};

// Global arena for demonstration
Arena g_arena(1 << 16);  // 64 KiB

struct Task {
    struct promise_type {
        // Custom allocation: route to arena
        static void* operator new(std::size_t size) {
            std::printf("  arena alloc: %zu bytes\n", size);
            return g_arena.allocate(size);
        }

        // Arena is monotonic — delete is a no-op
        static void operator delete(void* /*ptr*/, std::size_t /*size*/) {
            std::printf("  arena dealloc (no-op)\n");
        }

        Task get_return_object() { return {}; }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };
};

Task small_coro() {
    int x = 42;
    co_return;
}

Task bigger_coro() {
    char buf[256]{};
    int arr[64]{};
    co_return;
}

int main() {
    std::puts("=== small_coro ===");
    small_coro();

    std::puts("=== bigger_coro ===");
    bigger_coro();

    g_arena.reset();  // reclaim all at once
}
// Example output:
// === small_coro ===
//   arena alloc: 80 bytes
//   arena dealloc (no-op)
// === bigger_coro ===
//   arena alloc: 592 bytes
//   arena dealloc (no-op)
```

Notice how `bigger_coro` uses far more frame space - the `char buf[256]` and `int arr[64]` both live in the frame, and the compiler adds its own bookkeeping on top. The exact size is implementation-defined and varies between compilers, but the point is you are seeing the real allocation size and routing it entirely away from the global heap.

---

### Q2: Use parameter-forwarding `operator new` to accept a stateful allocator passed as the first coroutine argument

This is the more powerful version of the trick. Instead of using a global arena, the coroutine's caller passes in a memory resource directly, and the promise's `operator new` receives that same argument forwarded by the compiler. The key insight is that the compiler will pass every coroutine argument (after the mandatory `size_t`) through to your `operator new` overload - so if the first parameter is a `pmr::memory_resource*`, your `operator new` sees it too:

```cpp
#include <coroutine>
#include <cstdio>
#include <memory_resource>
#include <utility>

template <typename T>
class Task {
public:
    struct promise_type {
        T value{};
        std::pmr::memory_resource* mr = nullptr;

        // Overload 1: receives coroutine parameters
        // The compiler forwards ALL coroutine args after size_t
        static void* operator new(std::size_t size,
                                   std::pmr::memory_resource* resource,
                                   auto&&... /* remaining args */) {
            std::printf("  pmr alloc: %zu bytes\n", size);
            return resource->allocate(size, alignof(std::max_align_t));
        }

        // Matching delete must accept the same extra params
        // But standard only requires (void*, size_t) — resource is lost.
        // Solution: stash resource pointer at a known offset.
        static void operator delete(void* ptr, std::size_t size) {
            // In production: retrieve resource from a header before the frame
            std::printf("  pmr dealloc: %zu bytes (fallback)\n", size);
            // For demonstration, use default delete
            std::pmr::get_default_resource()->deallocate(ptr, size,
                                                          alignof(std::max_align_t));
        }

        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(T v) { value = std::move(v); }
        void unhandled_exception() { std::terminate(); }
    };

    T get() {
        handle_.resume();
        return std::move(handle_.promise().value);
    }

    explicit Task(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~Task() { if (handle_) handle_.destroy(); }
    Task(Task&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    std::coroutine_handle<promise_type> handle_;
};

// Coroutine taking an allocator as first arg
Task<int> compute(std::pmr::memory_resource* /*mr*/, int a, int b) {
    co_return a + b;
}

int main() {
    std::byte buf[4096];
    std::pmr::monotonic_buffer_resource pool(buf, sizeof(buf));

    auto t = compute(&pool, 10, 32);
    std::printf("result = %d\n", t.get());  // 42
}
```

The lookup order the compiler follows when choosing an `operator new` overload is:

| Candidate | Signature |
| --- | --- |
| 1st (preferred) | `operator new(size_t, pmr::memory_resource*, int, int)` |
| 2nd | `operator new(size_t)` |
| 3rd | `::operator new(size_t)` |

The compiler tries the most-specific overload first, matching the full parameter list. If the specific one is not found, it falls back down the chain.

---

### Q3: When does HALO (Heap Allocation eLision Optimization) kick in, and how can you verify whether allocation was elided

HALO is the coroutine equivalent of NRVO - the compiler can prove that the coroutine frame's lifetime fits entirely inside the caller's frame, so it stack-allocates the frame instead of heap-allocating it. The reason this trips people up is that you cannot rely on it; it is a quality-of-implementation optimization, not a guarantee. The right way to find out whether HALO fired is to instrument `operator new` with a counter and count calls at runtime under optimization:

```cpp
#include <coroutine>
#include <cstdio>
#include <utility>
#include <cstddef>
#include <exception>

struct DetectAllocTask {
    struct promise_type {
        inline static int alloc_count = 0;

        static void* operator new(std::size_t size) {
            ++alloc_count;
            std::printf("  heap alloc #%d (%zu bytes)\n", alloc_count, size);
            return ::operator new(size);
        }
        static void operator delete(void* ptr, std::size_t size) {
            ::operator delete(ptr, size);
        }

        DetectAllocTask get_return_object() {
            return DetectAllocTask{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };

    void run() { if (h_) h_.resume(); }

    std::coroutine_handle<promise_type> h_;
    ~DetectAllocTask() { if (h_) h_.destroy(); }
    DetectAllocTask(DetectAllocTask&& o) noexcept : h_(std::exchange(o.h_, nullptr)) {}

private:
    explicit DetectAllocTask(std::coroutine_handle<promise_type> h) : h_(h) {}
};

// Candidate for HALO: small, non-escaping, called and consumed locally
DetectAllocTask inner() { co_return; }

DetectAllocTask outer() {
    auto t = inner();  // handle does not escape — HALO candidate
    t.run();
    co_return;
}

int main() {
    DetectAllocTask::promise_type::alloc_count = 0;

    auto t = outer();
    t.run();

    std::printf("Total allocations: %d\n",
                DetectAllocTask::promise_type::alloc_count);

    // With HALO:  Total allocations: 1 (only outer)
    // Without HALO: Total allocations: 2 (inner + outer)
}
```

The conditions that must hold for HALO to have a chance are:

| Condition | Why |
| --- | --- |
| Coroutine handle does not escape the caller | Compiler must prove lifetime containment |
| Caller is itself inlined or has visible frame | Frame can be stack-allocated |
| `operator new` is not `noexcept(false)` with side effects | Elision changes observable behaviour |
| Optimisation level >= `-O2` | HALO is an optimisation, not guaranteed |

The verification strategy is exactly what the code above does: instrument `operator new` with a counter and compile at `-O2`. Compare allocation counts. Alternatively, inspect the assembly for `operator new` call sites.

---

## Notes

- `promise_type::operator new` receives the **exact frame size** computed by the compiler - you cannot influence this size, only where it's allocated.
- The sized `operator delete(void*, std::size_t)` variant is preferred when available, matching the frame size passed to `operator new`.
- If `operator new` returns `nullptr` (for the `nothrow` overload `operator new(size_t, nothrow_t)`), the coroutine body is never entered and `get_return_object()` is never called. The caller receives a default-constructed return object.
- Monotonic arenas are ideal for coroutine-heavy workloads (e.g., network servers) where all coroutines in a batch complete together. Reset the arena between batches.
- HALO is **not** mandated by the standard - it is a quality-of-implementation optimisation. Do not design correctness around it; design *performance* around it.
- To force stack allocation when HALO fails, consider using `alloca` (non-standard) or `std::pmr::monotonic_buffer_resource` backed by a stack array, as shown in Q2.
