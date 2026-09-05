# Write Deterministic Execution Paths for Real-Time Code

**Category:** Low Latency and Real Time  
**Standard:** C++17, C++20  
**Reference:** [AUTOSAR C++14 Guidelines](https://www.autosar.org/), [MISRA C++:2023](https://misra.org.uk/)  

---

## Topic Overview

Deterministic code produces identical timing characteristics on every invocation. In hard real-time systems - flight controllers, medical devices, HFT order gateways - the worst-case execution time (WCET) must be bounded and provable. This means eliminating every source of non-determinism from the critical path: syscalls, memory allocation, exceptions, I/O, and data-dependent branching.

The real-time discipline divides the program into a **cold path** (initialization, configuration, logging) where non-deterministic operations are permitted, and a **hot path** (the real-time loop) where they are forbidden. The cold path pre-allocates all memory, pre-faults pages, opens file descriptors, and warms caches. The hot path operates exclusively on pre-allocated resources with bounded-time algorithms.

Exception handling is a particularly subtle source of non-determinism, and the reason trips people up: `throw` allocates memory (`__cxa_allocate_exception`), unwinds the stack (traversing DWARF tables at runtime), and invokes destructors whose count depends on the call depth. None of this has a bounded time. Replace exceptions with error codes, `std::expected` (C++23), or `std::optional` on the hot path. Similarly, virtual dispatch through vtables introduces an indirect branch that may cache-miss; prefer CRTP or `std::variant`-based dispatch in performance-critical code.

| Non-Deterministic Source | Why It's Unpredictable | Deterministic Replacement |
| --- | --- | --- |
| `malloc` / `new` | Lock contention, fragmentation, syscall | Arena, object pool |
| `throw` / `catch` | Stack unwinding, `__cxa_allocate_exception` | `std::expected`, error codes |
| `std::mutex::lock` | OS scheduler, priority inversion | Lock-free, spin with bounded retry |
| `read()` / `write()` | Kernel scheduling, disk I/O | Memory-mapped, pre-loaded buffers |
| `std::map::insert` | O(log n) with rebalancing | Pre-sized `std::array`, flat map |
| Virtual call | Indirect branch, icache miss | CRTP, `std::variant`, template policy |
| Page fault | Lazy page allocation | `mlock`, pre-fault (`memset`) |

Here is the lifecycle picture. Everything non-deterministic is intentionally front-loaded into the cold path, so the hot path can run without surprises. Shutdown is also a non-RT phase - that's where you flush logs and free resources:

```cpp
PROGRAM LIFECYCLE:
┌────────────────┐     ┌─────────────────────────┐     ┌──────────────┐
│   Cold Path    │────>│      Hot Path (RT)       │────>│   Shutdown   │
│ alloc, mlock,  │     │ NO alloc, NO throw,      │     │ free, flush  │
│ prefault, open │     │ NO syscall, NO I/O       │     │ close, log   │
└────────────────┘     └─────────────────────────┘     └──────────────┘
```

---

## Self-Assessment

### Q1: Build a real-time-safe message queue that pre-allocates all storage at construction, uses no exceptions, and has bounded-time push/pop

This is a single-producer single-consumer lock-free ring buffer. The capacity must be a power of two so that wrapping can be done with a bitmask instead of a modulo operation (modulo with a runtime divisor is not O(1) on all hardware). All storage lives inside `buffer_` - no heap allocation after construction:

```cpp
#include <atomic>
#include <array>
#include <optional>
#include <cstddef>
#include <cassert>

// Single-producer, single-consumer bounded queue. O(1) push/pop, no allocation.
template <typename T, std::size_t Capacity>
class RTQueue {
    static_assert((Capacity & (Capacity - 1)) == 0, "Capacity must be power of 2");
    static constexpr std::size_t Mask = Capacity - 1;

    std::array<T, Capacity> buffer_;
    alignas(64) std::atomic<std::size_t> head_{0};  // written by consumer
    alignas(64) std::atomic<std::size_t> tail_{0};  // written by producer

public:
    // Returns false if full - no allocation, no exception
    [[nodiscard]] bool try_push(const T& val) noexcept {
        std::size_t tail = tail_.load(std::memory_order_relaxed);
        std::size_t next = (tail + 1) & Mask;
        if (next == head_.load(std::memory_order_acquire))
            return false;  // full
        buffer_[tail] = val;
        tail_.store(next, std::memory_order_release);
        return true;
    }

    [[nodiscard]] std::optional<T> try_pop() noexcept {
        std::size_t head = head_.load(std::memory_order_relaxed);
        if (head == tail_.load(std::memory_order_acquire))
            return std::nullopt;  // empty
        T val = buffer_[head];
        head_.store((head + 1) & Mask, std::memory_order_release);
        return val;
    }

    std::size_t capacity() const noexcept { return Capacity - 1; }
};

#include <thread>
#include <cstdio>

int main() {
    RTQueue<int, 1024> queue;  // all storage on stack, no heap
    constexpr int N = 100'000;

    std::thread producer([&] {
        for (int i = 0; i < N; ++i)
            while (!queue.try_push(i)) {}  // bounded spin
    });
    std::thread consumer([&] {
        int expected = 0;
        while (expected < N) {
            if (auto val = queue.try_pop()) {
                assert(*val == expected++);
            }
        }
    });
    producer.join();
    consumer.join();
    std::printf("Transferred %d messages deterministically.\n", N);
}
```

The `alignas(64)` on `head_` and `tail_` puts them on separate cache lines, preventing false sharing between the producer and consumer. Without it, each write to `tail_` would invalidate the cache line containing `head_`, causing unnecessary coherency traffic between cores.

### Q2: Replace exception-based error handling with `std::expected` (C++23) in a trading order validation function

`std::expected<T, E>` is a discriminated union - it holds either a `T` on success or an `E` on failure. The critical point for real-time work is that it has **zero heap allocation** and **zero stack unwinding**. Error paths are just data, not control flow. The `[[nodiscard]]` attribute ensures callers can't silently ignore a rejection:

```cpp
#include <cstdint>
#include <cstdio>
#include <expected>
#include <string_view>

enum class OrderError : uint8_t {
    InvalidPrice,
    InvalidQuantity,
    ExceedsRiskLimit,
    UnknownInstrument
};

constexpr std::string_view to_string(OrderError e) noexcept {
    switch (e) {
        case OrderError::InvalidPrice:      return "InvalidPrice";
        case OrderError::InvalidQuantity:   return "InvalidQuantity";
        case OrderError::ExceedsRiskLimit:  return "ExceedsRiskLimit";
        case OrderError::UnknownInstrument: return "UnknownInstrument";
    }
    return "Unknown";
}

struct ValidatedOrder {
    uint64_t instrument_id;
    double price;
    int32_t qty;
};

// Hot-path: no exceptions, no allocation, deterministic branches
[[nodiscard]]
std::expected<ValidatedOrder, OrderError>
validate_order(uint64_t inst_id, double price, int32_t qty,
               double risk_limit) noexcept {
    if (inst_id == 0)
        return std::unexpected(OrderError::UnknownInstrument);
    if (price <= 0.0 || price > 1e9)
        return std::unexpected(OrderError::InvalidPrice);
    if (qty <= 0 || qty > 1'000'000)
        return std::unexpected(OrderError::InvalidQuantity);
    if (price * qty > risk_limit)
        return std::unexpected(OrderError::ExceedsRiskLimit);
    return ValidatedOrder{inst_id, price, qty};
}

int main() {
    auto result = validate_order(42, 150.75, 1000, 1'000'000.0);
    if (result) {
        auto& order = *result;
        std::printf("Valid: inst=%llu price=%.2f qty=%d\n",
                    (unsigned long long)order.instrument_id,
                    order.price, order.qty);
    } else {
        std::printf("Rejected: %.*s\n",
                    (int)to_string(result.error()).size(),
                    to_string(result.error()).data());
    }

    auto bad = validate_order(42, -10.0, 1000, 1'000'000.0);
    if (!bad) {
        std::printf("Rejected: %.*s\n",
                    (int)to_string(bad.error()).size(),
                    to_string(bad.error()).data());
    }
}
```

Compare this to an exception-based version: every `throw` site would potentially allocate the exception object, and every call site would need exception handling overhead even on the success path (in the zero-overhead model, that overhead is at the call site for table setup). `std::expected` has none of that.

### Q3: Pre-fault all pages at startup to eliminate minor page faults during the real-time loop, and verify with `getrusage`

This is the minimal version of the prefaulting pattern: allocate, lock, write every page, then measure faults before and after the hot path. The hot-path loop should report zero faults because every page it touches was already physically mapped during setup:

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
#include <cstdlib>
#include <cstring>
#include <cstdio>
#include <sys/mman.h>
#include <sys/resource.h>

// Pre-fault: write to every page so the OS maps physical memory NOW
void prefault_buffer(void* ptr, std::size_t size) noexcept {
    volatile char* p = static_cast<volatile char*>(ptr);
    const std::size_t page_size = 4096;
    for (std::size_t i = 0; i < size; i += page_size) {
        p[i] = 0;  // trigger page fault here, in cold path
    }
}

long get_minor_faults() {
    struct rusage usage;
    getrusage(RUSAGE_SELF, &usage);
    return usage.ru_minflt;
}

int main() {
    constexpr std::size_t kSize = 64 * 1024 * 1024;  // 64 MB

    // Cold path: allocate and lock
    void* buf = std::aligned_alloc(4096, kSize);
    mlock(buf, kSize);         // prevent swapping
    prefault_buffer(buf, kSize);  // force physical mapping

    long faults_before = get_minor_faults();

    // Hot path: use the buffer - should cause ZERO page faults
    char* data = static_cast<char*>(buf);
    for (std::size_t i = 0; i < kSize; ++i) {
        data[i] = static_cast<char>(i & 0xFF);
    }

    long faults_after = get_minor_faults();
    std::printf("Page faults during hot path: %ld\n",
                faults_after - faults_before);

    munlock(buf, kSize);
    std::free(buf);
}
```

If you see any non-zero fault count in the hot path, it means either the buffer wasn't fully prefaulted, another allocation happened inside the loop (check for hidden `malloc` calls in the libraries you're using), or the OS reclaimed some pages due to memory pressure.

---

## Notes

- Compile with `-fno-exceptions -fno-rtti` on the hot-path translation units to enforce the no-exception discipline at the compiler level.
- **`std::expected`** (C++23) is zero-overhead: it is a discriminated union with no heap allocation and no stack unwinding.
- **Pre-faulting** must write to each page; `memset` works but `mmap` with `MAP_POPULATE` is faster for large regions.
- Use `-Wframe-larger-than=N` to catch accidental large stack allocations in hot-path functions.
- Profile with `perf stat -e minor-faults,major-faults` to verify zero faults in steady state.
- **`mlockall(MCL_CURRENT | MCL_FUTURE)`** is the nuclear option: locks all current and future pages but requires `CAP_IPC_LOCK`.
