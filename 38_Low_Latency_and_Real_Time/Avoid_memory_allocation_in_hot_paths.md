# Avoid Memory Allocation in Hot Paths

**Category:** Low Latency and Real Time  
**Standard:** C++17, C++20  
**Reference:** [P0211R3 - Allocator-aware library wrappers](https://wg21.link/p0211), [CppCon 2017 - John Lakos: Local Allocators](https://www.youtube.com/watch?v=nZNd5FjSquk)  

---

## Topic Overview

Every call to `operator new` or `malloc` enters a global heap allocator, which typically acquires a lock, walks free-lists, and may trigger a `brk`/`mmap` syscall. In low-latency hot paths - order matching engines, audio callbacks, signal handlers - a single allocation can add 1-50µs of jitter. The zero-allocation discipline mandates that the critical path touches only pre-allocated memory.

The reason this matters so much is that the heap allocator is shared state. Every thread in your process competes for the same allocator lock, and the time spent waiting for that lock is completely unpredictable. Even on an uncontended path, the allocator has to search free-lists whose traversal time depends on the current state of the heap - which changes continuously. You have essentially no control over how long an allocation will take.

The primary tools are: **arena (monotonic) allocators** that bump a pointer forward with no per-object overhead, **object pools** that pre-construct a fixed set of reusable objects, and **stack-based containers** like `std::array`, `boost::static_vector`, or `std::pmr::monotonic_buffer_resource` backed by a stack buffer. C++17's `<memory_resource>` standardizes polymorphic allocators that can be swapped without changing container types.

Hidden allocations are insidious. `std::string` may allocate on construction if the string exceeds the SSO buffer (typically 15-22 chars). `std::function` heap-allocates when the callable exceeds its small-buffer (often 2-3 pointers). Even `std::vector::push_back` allocates when capacity is exhausted. Audit every type on the hot path.

The reason this trips people up is that the standard library is wonderfully convenient in normal code, and the allocation cost is invisible in the source. You write `std::string name = ...;` and it looks like one line. But whether that one line triggers a system call depends on string length, SSO size, and the current state of the heap. In a hot path, that uncertainty is unacceptable.

| Source | Hidden Allocation? | Mitigation |
| --- | --- | --- |
| `std::string` > SSO limit | Yes | Use `std::string_view`, fixed `char[]`, or SSO-aware sizing |
| `std::function<void()>` | Depends on callable size | Use `function_ref`, template parameter, or `inplace_function` |
| `std::vector::push_back` | On realloc | `reserve()` upfront or use `static_vector` |
| `std::map` / `std::unordered_map` | Per-node | Use `pmr::` variants with pool resource or flat maps |
| `std::shared_ptr` (make_shared) | One allocation | Pre-allocate, use intrusive ref count |
| Exception throwing | Yes (`__cxa_allocate_exception`) | Avoid exceptions on hot path |

Here's the mental model for hot-path memory. You set up your arenas and pools during the cold startup phase, where allocating from the system heap is completely fine. Then, once you enter the hot loop, you never touch the global heap again - everything goes through your pre-allocated buffers:

```cpp
HOT PATH MEMORY FLOW:
┌──────────────┐
│ Pre-allocated │──► Arena / Pool (bump pointer, O(1))
│   buffer      │
└──────────────┘
       ▲  
       │  setup phase (cold path: malloc OK)
┌──────┴───────┐
│ System heap  │
└──────────────┘
```

---

## Self-Assessment

### Q1: Implement a monotonic arena allocator compatible with `std::pmr::memory_resource` and use it to back a `pmr::vector` on the hot path

The key insight here is that a monotonic (bump-pointer) allocator can allocate in O(1) time with zero synchronization - it just advances a pointer. The `std::pmr::memory_resource` interface lets you plug this into any `pmr`-aware container without changing the container's type.

```cpp
#include <memory_resource>
#include <vector>
#include <cstdio>
#include <cstdlib>
#include <cassert>

class ArenaResource final : public std::pmr::memory_resource {
    char* buffer_;
    std::size_t capacity_;
    std::size_t offset_ = 0;

    void* do_allocate(std::size_t bytes, std::size_t alignment) override {
        std::size_t aligned = (offset_ + alignment - 1) & ~(alignment - 1);
        if (aligned + bytes > capacity_)
            throw std::bad_alloc{};  // never happens if sized correctly
        void* ptr = buffer_ + aligned;
        offset_ = aligned + bytes;
        return ptr;
    }

    void do_deallocate(void* /*p*/, std::size_t /*bytes*/,
                       std::size_t /*alignment*/) override {
        // monotonic: no individual deallocation
    }

    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }

public:
    ArenaResource(void* buf, std::size_t cap)
        : buffer_(static_cast<char*>(buf)), capacity_(cap) {}

    void reset() noexcept { offset_ = 0; }
    std::size_t used() const noexcept { return offset_; }
};

// Hot-path function: ZERO heap allocations
void process_batch(ArenaResource& arena) {
    arena.reset();
    std::pmr::vector<double> prices(&arena);
    prices.reserve(1024);  // uses arena memory

    for (int i = 0; i < 1000; ++i)
        prices.push_back(i * 1.01);

    double sum = 0;
    for (double p : prices) sum += p;
    std::printf("Sum: %.2f, arena used: %zu bytes\n", sum, arena.used());
}

int main() {
    // Cold path: allocate arena once
    alignas(64) char buffer[64 * 1024];
    ArenaResource arena(buffer, sizeof(buffer));

    for (int iteration = 0; iteration < 10; ++iteration)
        process_batch(arena);  // zero allocations per call
}
```

Notice that `do_deallocate` is a no-op. Freeing individual objects is meaningless in a monotonic arena - you "free" everything at once by calling `reset()`. That simplicity is exactly what makes this O(1) and thread-safe (for a single-threaded hot path).

### Q2: Build a typed object pool that pre-allocates N objects and provides O(1) `acquire`/`release` with no heap allocation

A pool is the right tool when you need actual object lifetimes - you want to acquire an object, use it, and return it. The trick is to store the free-list index inside the same memory as the object itself, using a `union`. That way the pool's bookkeeping costs zero extra memory.

```cpp
#include <array>
#include <cstdint>
#include <cassert>
#include <new>
#include <utility>
#include <optional>
#include <cstdio>

template <typename T, std::size_t N>
class ObjectPool {
    union Slot {
        T object;
        std::size_t next_free;  // index of next free slot
        Slot() noexcept : next_free(0) {}
        ~Slot() {}
    };

    std::array<Slot, N> slots_;
    std::size_t free_head_ = 0;
    std::size_t alive_ = 0;

public:
    ObjectPool() {
        for (std::size_t i = 0; i < N - 1; ++i)
            slots_[i].next_free = i + 1;
        slots_[N - 1].next_free = N;  // sentinel
    }

    template <typename... Args>
    T* acquire(Args&&... args) {
        if (free_head_ == N) return nullptr;  // pool exhausted
        std::size_t idx = free_head_;
        free_head_ = slots_[idx].next_free;
        T* obj = ::new (&slots_[idx].object) T(std::forward<Args>(args)...);
        ++alive_;
        return obj;
    }

    void release(T* obj) {
        assert(obj);
        std::size_t idx = reinterpret_cast<Slot*>(obj) - slots_.data();
        assert(idx < N);
        obj->~T();
        slots_[idx].next_free = free_head_;
        free_head_ = idx;
        --alive_;
    }

    std::size_t alive() const { return alive_; }
    std::size_t available() const { return N - alive_; }

    ~ObjectPool() {
        // Note: user must release all objects before destruction
    }
};

struct Order {
    uint64_t id;
    double price;
    int qty;
};

int main() {
    ObjectPool<Order, 4096> pool;

    // Hot path: acquire and release with zero allocations
    Order* o1 = pool.acquire(Order{1001, 45.50, 100});
    Order* o2 = pool.acquire(Order{1002, 46.00, 200});
    std::printf("Alive: %zu, Available: %zu\n", pool.alive(), pool.available());

    pool.release(o1);
    pool.release(o2);
    std::printf("Alive: %zu, Available: %zu\n", pool.alive(), pool.available());
}
```

If `acquire` returns `nullptr`, you've exhausted the pool. In production, that's a design bug - your pool should be sized to the worst-case concurrent object count, and you should never silently fall back to `malloc`.

### Q3: Demonstrate hidden allocations from `std::function` and `std::string`, and show zero-allocation alternatives

The best way to prove that code is allocation-free is to override `operator new` and count calls. That's exactly what this example does. Watch the allocation count go from non-zero to zero.

```cpp
#include <cstdio>
#include <cstdlib>
#include <functional>
#include <string>
#include <string_view>
#include <array>

// Override global new to detect allocations
static thread_local int alloc_count = 0;
void* operator new(std::size_t sz) {
    ++alloc_count;
    return std::malloc(sz);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

// Lightweight non-owning function_ref (C++26 std::function_ref)
template <typename Sig>
class function_ref;

template <typename R, typename... Args>
class function_ref<R(Args...)> {
    void* obj_;
    R (*invoke_)(void*, Args...);
public:
    template <typename F>
    function_ref(F& f) noexcept
        : obj_(const_cast<void*>(static_cast<const void*>(&f)))
        , invoke_([](void* o, Args... args) -> R {
              return (*static_cast<F*>(o))(std::forward<Args>(args)...);
          }) {}
    R operator()(Args... args) const {
        return invoke_(obj_, std::forward<Args>(args)...);
    }
};

void hot_path_bad() {
    alloc_count = 0;

    // Hidden alloc 1: string exceeds SSO
    std::string instrument = "VERY_LONG_INSTRUMENT_IDENTIFIER_XYZ_2025";

    // Hidden alloc 2: std::function with large capture
    std::array<char, 64> big_capture{};
    std::function<void()> callback = [big_capture]() {
        (void)big_capture;
    };

    std::printf("[BAD]  Allocations: %d\n", alloc_count);
}

void hot_path_good() {
    alloc_count = 0;

    // No alloc: string_view for non-owning reference
    std::string_view instrument = "VERY_LONG_INSTRUMENT_IDENTIFIER_XYZ_2025";

    // No alloc: function_ref is non-owning, no heap
    auto lambda = []() {};
    function_ref<void()> callback{lambda};

    std::printf("[GOOD] Allocations: %d\n", alloc_count);
}

int main() {
    hot_path_bad();
    hot_path_good();
}
```

The `hot_path_good` function does exactly the same logical work but with zero heap allocations. `std::string_view` is a non-owning reference to the string literal (which lives in read-only memory). `function_ref` stores a pointer to the lambda rather than copying it - it's non-owning, so no heap is needed. The trade-off is that you have to ensure the original lambda outlives the `function_ref`.

---

## Notes

- **`std::pmr::monotonic_buffer_resource`** is the standard arena; prefer it over hand-rolled arenas unless you need custom alignment or reset semantics.
- **Object pools** should be sized to the worst-case count. If the pool is exhausted on the hot path, you have a design bug - never fall back to `malloc`.
- **SSO threshold** varies: libstdc++ uses 15 chars, MSVC uses 15, libc++ uses 22. Know your platform.
- **`std::inplace_function`** (proposed) or Boost `function` with `SBO_SIZE` template parameter eliminates `std::function` heap allocation for known-size callables.
- **Overriding `operator new`** in a test build is the simplest way to assert zero allocations on a hot path - any allocation triggers a breakpoint or counter.
- Run under `perf stat -e page-faults` to verify no page faults during steady state.
