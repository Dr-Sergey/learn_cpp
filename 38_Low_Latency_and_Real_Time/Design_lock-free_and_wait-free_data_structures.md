# Design Lock-Free and Wait-Free Data Structures

**Category:** Low Latency and Real Time  
**Standard:** C++20, C++23  
**Reference:** [P0233R0 - Hazard Pointers](https://wg21.link/p0233), [C++ Concurrency in Action, 2nd Ed.](https://www.manning.com/books/c-plus-plus-concurrency-in-action-second-edition)  

---

## Topic Overview

Lock-free data structures guarantee that at least one thread makes progress in any execution, regardless of scheduling. Wait-free structures go further: every thread completes its operation in a bounded number of steps. These properties make them essential in low-latency systems where mutex contention introduces unpredictable tail latency.

The reason this matters isn't just about the average case. A mutex introduces the possibility that your thread will block for an unbounded amount of time - however briefly the lock holder holds the mutex, that's still time your thread spends doing nothing. Worse, if the lock holder gets preempted while holding the mutex, your thread blocks for the entire preemption interval. In a real-time context, that interval can easily be tens of milliseconds. Lock-free structures eliminate that class of failure entirely.

The foundation is the **compare-and-swap (CAS)** primitive - `std::atomic::compare_exchange_weak/strong`. A CAS loop speculatively prepares a new state, then atomically publishes it only if no other thread intervened. Weak CAS may fail spuriously but is cheaper on LL/SC architectures (ARM, POWER); strong CAS never fails spuriously and is preferred when the loop body is expensive.

Memory reclamation is the hardest problem in lock-free design. The reason this trips people up is a subtle race: thread A pops a node from a stack and is about to read `node->data`. Thread B also pops the same node, finishes first, and calls `delete node`. Now thread A reads freed memory - undefined behavior. This race doesn't exist with mutexes because only one thread at a time can be in the critical section. Solutions include epoch-based reclamation, hazard pointers (standardized in C++26 as `std::hazard_pointer`), and RCU-style quiescent-state tracking.

| Property | Mutex-Based | Lock-Free | Wait-Free |
| --- | --- | --- | --- |
| Progress guarantee | None (deadlock possible) | System-wide | Per-thread bounded |
| Typical latency | Low contention: fast; high contention: spikes | Consistent, CAS retry overhead | Most consistent |
| Complexity | Low | High | Very high |
| Memory reclamation | Automatic (scope) | Epoch / Hazard pointers | Epoch / Hazard pointers |
| Use case | General purpose | Hot-path queues, stacks | Hard real-time counters |

Here's the basic CAS retry loop structure. Thread A reads the current head, prepares a new head, and atomically swaps only if the head hasn't changed. If another thread modified the head first, the CAS fails and thread A retries:

```cpp
Thread A ──► read top ──► prepare new_node ──► CAS(top, old, new) ──► success
                                                    │ fail
                                                    ▼
                                               retry (reload top)
```

---

## Self-Assessment

### Q1: Implement a lock-free stack with `push` and `try_pop` using CAS, handling the ABA problem with a tagged pointer

The ABA problem is a subtle correctness issue specific to lock-free structures. Thread A reads head as pointer `X`. Thread B pops `X`, pushes a new node at some other address, then pushes `X` back (it reused that memory). Thread A's CAS succeeds because head is still `X` - but the list structure has changed in ways thread A didn't see. Tagged pointers solve this by pairing each pointer with a monotonically increasing counter. Even if the pointer value repeats, the tag never matches the old value.

```cpp
#include <atomic>
#include <cstdint>
#include <optional>
#include <new>

template <typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        explicit Node(T val) : data(std::move(val)), next(nullptr) {}
    };

    // Tagged pointer: pack a counter into the upper 16 bits
    // (assumes 48-bit virtual addresses on x86-64)
    struct TaggedPtr {
        Node* ptr;
        uint16_t tag;
    };

    static_assert(std::atomic<TaggedPtr>::is_always_lock_free,
                  "TaggedPtr must be lock-free for ABA protection");

    std::atomic<TaggedPtr> head_{TaggedPtr{nullptr, 0}};

public:
    void push(T val) {
        Node* new_node = new Node(std::move(val));
        TaggedPtr old_head = head_.load(std::memory_order_relaxed);
        TaggedPtr new_head;
        do {
            new_node->next = old_head.ptr;
            new_head = TaggedPtr{new_node, static_cast<uint16_t>(old_head.tag + 1)};
        } while (!head_.compare_exchange_weak(old_head, new_head,
                    std::memory_order_release, std::memory_order_relaxed));
    }

    std::optional<T> try_pop() {
        TaggedPtr old_head = head_.load(std::memory_order_acquire);
        TaggedPtr new_head;
        do {
            if (!old_head.ptr) return std::nullopt;
            new_head = TaggedPtr{old_head.ptr->next,
                                 static_cast<uint16_t>(old_head.tag + 1)};
        } while (!head_.compare_exchange_weak(old_head, new_head,
                    std::memory_order_acq_rel, std::memory_order_acquire));
        T val = std::move(old_head.ptr->data);
        delete old_head.ptr;  // simplified; production needs deferred reclamation
        return val;
    }

    ~LockFreeStack() {
        while (try_pop()) {}
    }
};

// Usage
#include <thread>
#include <vector>
#include <cassert>
#include <utility>

int main() {
    LockFreeStack<int> stack;
    constexpr int N = 100'000;

    std::thread producer([&] {
        for (int i = 0; i < N; ++i) stack.push(i);
    });
    std::thread consumer([&] {
        int count = 0;
        while (count < N) {
            if (stack.try_pop()) ++count;
        }
    });
    producer.join();
    consumer.join();
}
```

The comment "production needs deferred reclamation" is important. The `delete old_head.ptr` is technically unsafe in a multi-consumer scenario for the reason described above. In production, you'd use a hazard pointer or epoch-based scheme to defer the free until no other thread can possibly hold a reference to the node.

### Q2: Build a wait-free atomic counter with `fetch_add` and a bounded-time `snapshot()` that returns the exact value with no contention spike

The trick here is to eliminate contention entirely by giving each thread its own counter slot. Incrementing is trivially wait-free: it's a single `fetch_add` on memory only your thread writes. The `snapshot()` reads all slots and sums them - it's bounded by the number of threads, which is a fixed constant.

```cpp
#include <atomic>
#include <array>
#include <numeric>
#include <thread>
#include <cstdint>
#include <cassert>
#include <vector>

// Distributed counter: each thread writes to its own cache line,
// snapshot reads all slots — wait-free for both increment and read.
class WaitFreeCounter {
    static constexpr int kMaxThreads = 64;
    struct alignas(64) Slot {  // one cache line per slot
        std::atomic<int64_t> value{0};
    };
    std::array<Slot, kMaxThreads> slots_{};
    std::atomic<int> next_slot_{0};

    static thread_local int my_slot_;

    int get_slot() {
        if (my_slot_ < 0) {
            my_slot_ = next_slot_.fetch_add(1, std::memory_order_relaxed);
            assert(my_slot_ < kMaxThreads);
        }
        return my_slot_;
    }

public:
    // Wait-free: single atomic fetch_add on thread-local slot
    void increment(int64_t delta = 1) {
        slots_[get_slot()].value.fetch_add(delta, std::memory_order_relaxed);
    }

    // Wait-free: bounded loop (kMaxThreads iterations)
    int64_t snapshot() const {
        int64_t sum = 0;
        int n = next_slot_.load(std::memory_order_acquire);
        for (int i = 0; i < n; ++i) {
            sum += slots_[i].value.load(std::memory_order_relaxed);
        }
        return sum;
    }
};

thread_local int WaitFreeCounter::my_slot_ = -1;

int main() {
    WaitFreeCounter counter;
    constexpr int kThreads = 8;
    constexpr int kOps = 1'000'000;

    std::vector<std::thread> threads;
    for (int t = 0; t < kThreads; ++t) {
        threads.emplace_back([&] {
            for (int i = 0; i < kOps; ++i) counter.increment();
        });
    }
    for (auto& th : threads) th.join();

    assert(counter.snapshot() == int64_t(kThreads) * kOps);
}
```

The `alignas(64)` on each slot is critical. Without it, slots from different threads would share a cache line, and every increment would cause cache-line bouncing - all threads fighting over the same line. The alignment puts each slot on its own cache line, so threads truly don't interfere with each other.

### Q3: Implement exponential backoff for a CAS retry loop and measure its impact on throughput under contention

Under heavy contention, a plain CAS retry loop makes things worse: many threads spin fast, generating constant cache-line invalidations, which slows everyone down. Exponential backoff adds a small delay after each failed CAS and doubles it on the next failure. This spreads out the retry attempts in time, reducing the collision rate.

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <chrono>
#include <cstdio>
#include <immintrin.h>  // _mm_pause

struct BackoffPolicy {
    int spin_min = 1;
    int spin_max = 1024;

    void operator()() {
        thread_local int spins = spin_min;
        for (int i = 0; i < spins; ++i) {
            _mm_pause();  // yield execution resources to sibling hyperthread
        }
        spins = std::min(spins * 2, spin_max);  // exponential backoff
    }

    void reset() {
        // reset on successful CAS — called by user
    }
};

std::atomic<int64_t> shared_counter{0};

template <bool UseBackoff>
void increment_loop(int ops) {
    BackoffPolicy backoff;
    for (int i = 0; i < ops; ++i) {
        int64_t old_val = shared_counter.load(std::memory_order_relaxed);
        while (!shared_counter.compare_exchange_weak(old_val, old_val + 1,
                std::memory_order_relaxed)) {
            if constexpr (UseBackoff) backoff();
        }
    }
}

int main() {
    constexpr int kThreads = 8;
    constexpr int kOps = 500'000;

    auto bench = [&](auto fn, const char* label) {
        shared_counter.store(0);
        auto t0 = std::chrono::high_resolution_clock::now();
        std::vector<std::thread> threads;
        for (int i = 0; i < kThreads; ++i)
            threads.emplace_back(fn, kOps);
        for (auto& t : threads) t.join();
        auto t1 = std::chrono::high_resolution_clock::now();
        double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
        std::printf("%-20s %8.2f ms  counter=%lld\n", label, ms,
                    (long long)shared_counter.load());
    };

    bench(increment_loop<false>, "No backoff");
    bench(increment_loop<true>,  "Exp backoff");
}
```

On a machine with 8 or more threads, the backoff version is typically 2-5x faster in total throughput, even though each individual increment takes slightly longer. Paradoxically, making each thread slower improves overall throughput because threads are no longer constantly invalidating each other's cache lines.

---

## Notes

- **Weak vs strong CAS**: Prefer `compare_exchange_weak` inside loops on ARM/POWER; on x86, both compile identically.
- **ABA problem**: Solved by tagged pointers (DWCAS), hazard pointers, or epoch-based reclamation - never by ignoring it.
- **Wait-free counters** via thread-local slots are the easiest wait-free structure; wait-free queues are research-grade complexity.
- **Memory ordering**: Lock-free structures almost always need at least `acquire`/`release`; `relaxed` is only correct for independent counters.
- **Testing**: Use thread sanitizer (`-fsanitize=thread`) and stress tests with `taskset` to force contention on fewer cores.
- **`std::atomic::wait/notify`** (C++20) can replace spin loops when bounded latency is less critical than power efficiency.
