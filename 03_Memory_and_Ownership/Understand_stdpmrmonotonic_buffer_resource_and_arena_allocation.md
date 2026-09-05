# Understand std::pmr::monotonic_buffer_resource and Arena Allocation

**Category:** Memory and Ownership  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/memory/monotonic_buffer_resource>  

---

## Topic Overview

### What Is Arena / Monotonic Allocation

An arena allocator hands out memory sequentially from a buffer - think of a bump pointer that just moves forward. Individual deallocations are no-ops, and the entire buffer is freed at once when the arena is destroyed. That single bulk-free is the whole point.

Here is a mental picture of what that looks like in memory:

```cpp
Buffer: [████████████████████████████░░░░░░░░░░░░░░░░]
         ^alloc1 ^alloc2 ^alloc3     ^next free

// No per-object free — entire buffer reset at destruction
```

The filled blocks are live allocations sitting side by side. There is no freelist, no per-block header, and no fragmentation. When the arena goes away, so does everything.

### How `monotonic_buffer_resource` Works

The table below covers the key characteristics. The two that catch people off guard are the no-op deallocation and the thread-safety limitation.

| Feature | Detail |
| --- | --- |
| **Header** | `<memory_resource>` |
| **Allocation** | Bump pointer - O(1), no fragmentation |
| **Deallocation** | No-op (memory recycled only at destruction) |
| **Upstream** | Optional fallback allocator when buffer is exhausted |
| **Thread safety** | **Not thread-safe** - single-threaded use only |
| **Best for** | Short-lived, burst allocations (parsing, frame processing) |

### Key PMR Classes

The typical setup is three lines: give the resource a buffer, then pass it to any PMR container. All containers sharing the same resource draw from the same arena.

```cpp
// 1. Provide a stack buffer
char buf[4096];
std::pmr::monotonic_buffer_resource mbr(buf, sizeof(buf));

// 2. Use it with PMR containers
std::pmr::vector<int> vec(&mbr);       // allocates from buf
std::pmr::string str(&mbr);            // shares same arena

// 3. All memory released when mbr is destroyed
```

Notice that both `vec` and `str` share the same underlying buffer. Their allocations are simply laid out consecutively inside it.

---

## Self-Assessment

### Q1: Rewrite a function that creates many short-lived objects using a `monotonic_buffer_resource`

The "bad" version is the default you write without thinking about it. The "good" version trades per-object heap calls for a single stack buffer and one cleanup at the end. The nested arena example at the bottom shows that arenas can be layered - an inner arena can use an outer one as its upstream fallback when it runs out of space.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <string>
#include <numeric>

// BAD: default allocator — many small heap allocations
std::vector<std::string> parse_tokens_default(const std::vector<std::string>& raw) {
    std::vector<std::string> result;
    for (const auto& s : raw) {
        result.push_back("[" + s + "]");  // Each string = heap alloc
    }
    return result;
}

// GOOD: monotonic arena — all allocations from one buffer
void parse_tokens_arena(const std::vector<std::string>& raw) {
    // Stack buffer for small workloads; upstream heap for overflow
    char buffer[4096];
    std::pmr::monotonic_buffer_resource arena(buffer, sizeof(buffer));

    // All allocations come from the arena
    std::pmr::vector<std::pmr::string> result(&arena);
    result.reserve(raw.size());

    for (const auto& s : raw) {
        result.emplace_back("[" + s + "]", &arena);
    }

    std::cout << "Tokens parsed: " << result.size() << "\n";
    for (const auto& t : result) {
        std::cout << "  " << t << "\n";
    }
    // All arena memory freed here — one deallocation, not N
}

int main() {
    std::vector<std::string> input = {"hello", "world", "foo", "bar", "baz"};

    std::cout << "=== Arena-based parsing ===\n";
    parse_tokens_arena(input);

    // Nested arena example
    std::cout << "\n=== Nested arenas ===\n";
    char outer_buf[8192];
    std::pmr::monotonic_buffer_resource outer(outer_buf, sizeof(outer_buf));

    {
        // Inner arena with outer as upstream fallback
        char inner_buf[256];
        std::pmr::monotonic_buffer_resource inner(inner_buf, sizeof(inner_buf), &outer);

        std::pmr::vector<int> nums(&inner);
        for (int i = 0; i < 100; ++i) nums.push_back(i);

        std::cout << "Sum: " << std::accumulate(nums.begin(), nums.end(), 0) << "\n";
        // inner destroyed — but outer memory still available
    }

    std::pmr::vector<double> data(&outer);
    data.push_back(3.14);
    std::cout << "Outer still alive: " << data[0] << "\n";

    return 0;
}
// Expected output:
// === Arena-based parsing ===
// Tokens parsed: 5
//   [hello]
//   [world]
//   [foo]
//   [bar]
//   [baz]
//
// === Nested arenas ===
// Sum: 4950
// Outer still alive: 3.14
```

### Q2: Measure the speedup of an arena allocator vs the default allocator for a parse workload

The benchmark here is straightforward: do the same work N times, once with the default allocator and once with an arena. The arena version re-uses the same stack buffer on every iteration instead of going to the heap. Watch especially the comments explaining the four reasons why it wins.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <string>
#include <chrono>

template<typename Func>
long long bench(const char* label, Func f, int iterations) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) {
        f();
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    std::cout << label << ": " << us << " us (" << iterations << " iterations)\n";
    return us;
}

int main() {
    constexpr int N = 10000;
    constexpr int TOKENS = 50;

    std::cout << "=== Benchmark: default vs arena allocator ===\n";
    std::cout << "Each iteration: create " << TOKENS << " strings in a vector\n\n";

    // Default allocator: many small heap allocs
    auto t_default = bench("Default alloc", [&] {
        std::vector<std::string> v;
        v.reserve(TOKENS);
        for (int i = 0; i < TOKENS; ++i) {
            v.push_back("token_" + std::to_string(i));
        }
    }, N);

    // Arena allocator: bump pointer from stack buffer
    auto t_arena = bench("Arena alloc  ", [&] {
        char buf[8192];
        std::pmr::monotonic_buffer_resource arena(buf, sizeof(buf));
        std::pmr::vector<std::pmr::string> v(&arena);
        v.reserve(TOKENS);
        for (int i = 0; i < TOKENS; ++i) {
            v.emplace_back("token_" + std::to_string(i), &arena);
        }
        // All memory freed at once when arena goes out of scope
    }, N);

    std::cout << "\nSpeedup: " << (double)t_default / (t_arena > 0 ? t_arena : 1)
              << "x faster with arena\n";

    // Why the arena is faster:
    // 1. No per-object malloc/free — just bump a pointer
    // 2. No deallocation overhead — destructor frees everything at once
    // 3. Better cache locality — allocations are contiguous
    // 4. No fragmentation from interleaved alloc/free

    return 0;
}
```

### Q3: Explain why `monotonic_buffer_resource` is not thread-safe and how to handle multi-threaded arenas

The reason this is not thread-safe is fundamental: the bump-pointer advance is not atomic. Two threads bumping it at the same time produce a data race - that is undefined behavior, and it does not take much load to trigger it. The code below shows three strategies for dealing with this, ordered from best performance to most flexible.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <thread>
#include <mutex>

// monotonic_buffer_resource is NOT thread-safe because:
// 1. Internal pointer bump is not atomic
// 2. No mutex on allocate/deallocate
// 3. Concurrent bumps = data race = UB

// Strategy 1: Thread-local arenas (best performance)
void strategy_thread_local() {
    std::cout << "=== Strategy 1: Thread-local arenas ===\n";

    auto worker = [](int id) {
        // Each thread has its own arena — no contention
        char buf[4096];
        std::pmr::monotonic_buffer_resource local_arena(buf, sizeof(buf));
        std::pmr::vector<int> data(&local_arena);

        for (int i = 0; i < 100; ++i)
            data.push_back(id * 1000 + i);

        std::cout << "Thread " << id << ": " << data.size() << " items\n";
    };

    std::thread t1(worker, 1);
    std::thread t2(worker, 2);
    t1.join();
    t2.join();
}

// Strategy 2: synchronized_pool_resource (thread-safe from PMR)
void strategy_synchronized() {
    std::cout << "\n=== Strategy 2: synchronized_pool_resource ===\n";

    // Thread-safe by design — uses internal synchronization
    std::pmr::synchronized_pool_resource pool;

    auto worker = [&pool](int id) {
        std::pmr::vector<int> data(&pool);
        for (int i = 0; i < 100; ++i)
            data.push_back(id * 1000 + i);
        std::cout << "Thread " << id << ": " << data.size() << " items\n";
    };

    std::thread t1(worker, 1);
    std::thread t2(worker, 2);
    t1.join();
    t2.join();
}

// Strategy 3: Mutex-protected monotonic (if you need monotonic specifically)
class ThreadSafeArena : public std::pmr::memory_resource {
    std::pmr::monotonic_buffer_resource arena_;
    std::mutex mtx_;

protected:
    void* do_allocate(size_t bytes, size_t align) override {
        std::lock_guard lock(mtx_);
        return arena_.allocate(bytes, align);
    }
    void do_deallocate(void* p, size_t bytes, size_t align) override {
        std::lock_guard lock(mtx_);
        arena_.deallocate(p, bytes, align);
    }
    bool do_is_equal(const std::pmr::memory_resource& other) const noexcept override {
        return this == &other;
    }
public:
    ThreadSafeArena(void* buf, size_t sz)
        : arena_(buf, sz) {}
};

int main() {
    strategy_thread_local();
    strategy_synchronized();

    std::cout << "\n=== Summary ===\n";
    std::cout << "Thread-local arena: best perf, no sharing\n";
    std::cout << "synchronized_pool:  thread-safe, pooled\n";
    std::cout << "Mutex wrapper:      thread-safe monotonic (adds contention)\n";

    return 0;
}
```

---

## Notes

- `monotonic_buffer_resource` is ideal for "allocate many, free all at once" patterns (parsing, frame processing).
- The stack buffer avoids heap allocation entirely for small workloads.
- Upstream resource (default: `new_delete_resource()`) handles overflow when the buffer is full.
- Typical speedups: 2x-10x over default allocator for burst allocation patterns.
- `deallocate()` is a no-op - individual frees do nothing. Memory is only reclaimed at destruction.
