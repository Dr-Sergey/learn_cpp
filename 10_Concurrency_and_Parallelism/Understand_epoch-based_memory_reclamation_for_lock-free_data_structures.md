# Understand Epoch-Based Memory Reclamation for Lock-Free Data Structures

**Category:** Concurrency and Parallelism  
**Standard:** C++11 and later (uses `std::atomic`, no dedicated standard library component)  
**Reference:** [Keir Fraser - Epoch-Based Reclamation (2004)](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-579.pdf)  

---

## Topic Overview

Lock-free data structures face a fundamental problem: when a thread removes a node, another thread may still be reading it. You cannot call `delete` immediately - that would be a use-after-free. You need a **safe memory reclamation (SMR)** scheme that defers destruction until no thread can possibly hold a reference to the retired node. Epoch-Based Reclamation (EBR) is one of the simplest and most efficient solutions.

EBR maintains a **global epoch counter** and per-thread **local epoch** values. Each thread, before accessing the shared data structure, announces that it has entered the current epoch. When a thread retires a node, it stamps the node with the current epoch and appends it to a thread-local retire list. A node can be freed only when **all threads have advanced past the epoch in which the node was retired** - meaning no thread could still be mid-traversal holding a pointer to it.

The reason this works is a simple invariant: if you are in epoch `E`, you announced that on entry, and the global epoch cannot advance past `E` until you leave. Two full epoch advancements after you retired a node guarantee no one could still be holding it.

Here is how the different SMR schemes compare at a glance:

| SMR Scheme | Overhead per Access | Space Bound | Complexity | Robustness to Stalled Threads |
| --- | --- | --- | --- | --- |
| **Epoch-Based (EBR)** | Very low (one atomic load/store) | Unbounded if thread stalls | Simple | Poor - stalled thread blocks all reclamation |
| **Hazard Pointers** | Moderate (one atomic store per pointer) | O(threads x hazards) | Moderate | Excellent - only stalled thread's pointers are blocked |
| **RCU (Read-Copy-Update)** | Near zero (compiler barrier) | Unbounded until grace period | Moderate | Varies by implementation |
| **Reference Counting** | High (atomic increment/decrement) | Bounded | Simple | Excellent |

### The Three-Epoch Protocol

Here is a timeline diagram of how epochs and retirements interact. The key insight is the two-epoch gap: a node retired in epoch E cannot be freed until after epoch E+2 has started, because that is the earliest point where every thread is guaranteed to have moved past E.

```cpp
Global epoch:  E  (values cycle: 0 -> 1 -> 2 -> 0 -> ...)

Thread enters critical section -> announces local_epoch = global_epoch
Thread exits critical section  -> marks itself as inactive

Retire(node):

  1. Stamp node with current global epoch E
  2. Append to thread-local retire list for epoch E

Try to advance global epoch E -> E+1:

  1. Check: every active thread has local_epoch >= E
  2. If yes: advance E -> E+1
  3. Free all nodes retired in epoch (E - 2)

     (Two full epochs have passed -> no thread can reference them)

Timeline:
  Epoch 0        Epoch 1        Epoch 2        Epoch 0 (wrap)
  ──────────────┼──────────────┼──────────────┼──────────────
  retire(A)     │              │ free(A)       │
                │ retire(B)    │              │ free(B)
```

The key invariant: a node retired in epoch `E` is freed only after the global epoch has advanced **twice** past `E`. Since a thread that entered at epoch `E` must exit its critical section before the epoch can advance past `E`, and it re-announces on re-entry, two epoch advancements guarantee no thread holds a stale reference.

---

## Self-Assessment

### Q1: Implement a minimal epoch-based reclamation system and demonstrate its use with a lock-free stack

The implementation below has two parts: the `EpochBasedReclamation` class that manages epochs and retire lists, and a `LockFreeStack` that uses it. Pay attention to how `enter()` and `leave()` bracket every data structure access:

```cpp
// Compile: g++ -std=c++20 -pthread q1_ebr.cpp -o q1
#include <atomic>
#include <array>
#include <functional>
#include <iostream>
#include <thread>
#include <vector>

class EpochBasedReclamation {
    static constexpr int MAX_THREADS = 64;
    static constexpr int EPOCH_COUNT = 3;

    struct ThreadEntry {
        std::atomic<int> local_epoch{-1};   // -1 = inactive
        std::vector<std::function<void()>> retire_lists[EPOCH_COUNT];
    };

    std::atomic<int> global_epoch_{0};
    std::array<ThreadEntry, MAX_THREADS> threads_{};
    std::atomic<int> thread_count_{0};

public:
    int register_thread() {
        return thread_count_.fetch_add(1, std::memory_order_relaxed);
    }

    void enter(int tid) {
        int e = global_epoch_.load(std::memory_order_relaxed);
        threads_[tid].local_epoch.store(e, std::memory_order_release);
        std::atomic_thread_fence(std::memory_order_seq_cst);
    }

    void leave(int tid) {
        threads_[tid].local_epoch.store(-1, std::memory_order_release);
    }

    void retire(int tid, std::function<void()> deleter) {
        int e = global_epoch_.load(std::memory_order_relaxed);
        threads_[tid].retire_lists[e % EPOCH_COUNT].push_back(std::move(deleter));
        try_advance();
    }

    void try_advance() {
        int e = global_epoch_.load(std::memory_order_relaxed);
        int n = thread_count_.load(std::memory_order_relaxed);

        for (int i = 0; i < n; ++i) {
            int local = threads_[i].local_epoch.load(std::memory_order_acquire);
            if (local != -1 && local != e)
                return;  // thread i is still in an older epoch
        }

        if (global_epoch_.compare_exchange_strong(e, e + 1,
                std::memory_order_acq_rel)) {
            int reclaim_epoch = (e + 1) % EPOCH_COUNT;
            for (int i = 0; i < n; ++i) {
                auto& list = threads_[i].retire_lists[reclaim_epoch];
                for (auto& fn : list) fn();
                list.clear();
            }
        }
    }
};

// ---------- Lock-free stack using EBR ----------
template <typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T d) : data(std::move(d)), next(nullptr) {}
    };

    std::atomic<Node*> head_{nullptr};
    EpochBasedReclamation& ebr_;

public:
    explicit LockFreeStack(EpochBasedReclamation& ebr) : ebr_(ebr) {}

    void push(int tid, T value) {
        Node* n = new Node(std::move(value));
        ebr_.enter(tid);
        n->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(n->next, n,
                std::memory_order_release, std::memory_order_relaxed))
            ;
        ebr_.leave(tid);
    }

    bool pop(int tid, T& out) {
        ebr_.enter(tid);
        Node* old_head = head_.load(std::memory_order_acquire);
        while (old_head) {
            if (head_.compare_exchange_weak(old_head, old_head->next,
                    std::memory_order_release, std::memory_order_relaxed)) {
                out = std::move(old_head->data);
                ebr_.retire(tid, [old_head]() { delete old_head; });
                ebr_.leave(tid);
                return true;
            }
        }
        ebr_.leave(tid);
        return false;
    }
};

int main() {
    EpochBasedReclamation ebr;
    LockFreeStack<int> stack(ebr);

    constexpr int N_THREADS = 4;
    constexpr int OPS = 10000;

    std::vector<std::jthread> threads;
    for (int t = 0; t < N_THREADS; ++t) {
        threads.emplace_back([&, tid = ebr.register_thread()]() {
            for (int i = 0; i < OPS; ++i) {
                stack.push(tid, i);
                int val;
                stack.pop(tid, val);
            }
        });
    }
    threads.clear();
    std::cout << "Completed " << N_THREADS * OPS
              << " push/pop pairs without use-after-free.\n";
    return 0;
}
```

**Key insight:** `enter()` and `leave()` bracket the critical section. Between them, the thread may hold raw pointers into the data structure. `retire()` defers deletion until two epoch advancements guarantee safety. The overhead is minimal - one atomic store on enter, one on leave.

---

### Q2: What happens when a thread stalls inside a critical section, and how can you detect or mitigate it

This is EBR's Achilles' heel, and it is worth understanding deeply. If a thread calls `enter()` and then gets preempted, blocks on I/O, or just takes a very long time, the global epoch cannot advance past the epoch that thread announced. Every node retired since then accumulates in the retire list with no way to free it. The detector below adds timestamps so you can at least see when this is happening:

```cpp
// Compile: g++ -std=c++20 -pthread q2_stall_detection.cpp -o q2
#include <atomic>
#include <chrono>
#include <iostream>
#include <thread>
#include <vector>
#include <array>

struct EBRWithStallDetection {
    static constexpr int MAX_THREADS = 16;

    struct ThreadState {
        std::atomic<int> local_epoch{-1};
        std::atomic<uint64_t> enter_timestamp_ns{0};
    };

    std::atomic<int> global_epoch{0};
    std::array<ThreadState, MAX_THREADS> threads{};
    std::atomic<int> thread_count{0};

    int register_thread() {
        return thread_count.fetch_add(1);
    }

    void enter(int tid) {
        using namespace std::chrono;
        auto now = duration_cast<nanoseconds>(
            steady_clock::now().time_since_epoch()).count();
        threads[tid].enter_timestamp_ns.store(now, std::memory_order_relaxed);
        threads[tid].local_epoch.store(
            global_epoch.load(std::memory_order_relaxed),
            std::memory_order_release);
    }

    void leave(int tid) {
        threads[tid].local_epoch.store(-1, std::memory_order_release);
        threads[tid].enter_timestamp_ns.store(0, std::memory_order_relaxed);
    }

    // Returns list of thread IDs that appear stalled
    std::vector<int> detect_stalls(
            std::chrono::milliseconds threshold) const {
        using namespace std::chrono;
        auto now = duration_cast<nanoseconds>(
            steady_clock::now().time_since_epoch()).count();
        auto thresh_ns = duration_cast<nanoseconds>(threshold).count();

        std::vector<int> stalled;
        int n = thread_count.load(std::memory_order_relaxed);
        for (int i = 0; i < n; ++i) {
            int e = threads[i].local_epoch.load(std::memory_order_acquire);
            if (e == -1) continue;  // not in critical section
            uint64_t ts = threads[i].enter_timestamp_ns.load(
                std::memory_order_relaxed);
            if (ts > 0 && (now - static_cast<int64_t>(ts)) > thresh_ns) {
                stalled.push_back(i);
            }
        }
        return stalled;
    }
};

int main() {
    using namespace std::chrono_literals;
    EBRWithStallDetection ebr;

    int tid_normal = ebr.register_thread();
    int tid_stall = ebr.register_thread();

    // Normal thread: enters and leaves quickly
    ebr.enter(tid_normal);
    ebr.leave(tid_normal);

    // Stalling thread: enters and never leaves
    ebr.enter(tid_stall);

    std::this_thread::sleep_for(150ms);

    auto stalled = ebr.detect_stalls(100ms);
    std::cout << "Stalled threads detected: " << stalled.size() << "\n";
    for (int id : stalled)
        std::cout << "  Thread " << id << " is stalled in critical section\n";

    ebr.leave(tid_stall);  // cleanup
    return 0;
}
```

**Key insight:** EBR's Achilles' heel is that a single stalled thread (blocked on I/O, descheduled, page fault) prevents the global epoch from advancing, causing **unbounded memory accumulation**. Production systems mitigate this by: (1) adding stall detection with timestamps, (2) keeping critical sections extremely short, (3) falling back to hazard pointers for robustness, or (4) using hybrid schemes like **DEBRA+** that can neutralize stalled threads.

---

### Q3: Compare EBR, hazard pointers, and RCU in a concrete scenario - reading from a lock-free linked list

Each SMR scheme has a different API and a different performance profile. Here is a side-by-side sketch of how you would read from a linked list using each approach:

```cpp
// Compile: g++ -std=c++20 -pthread q3_smr_comparison.cpp -o q3
// This file shows three approaches to safe reading from a concurrent list.
// Each approach is sketched to highlight the API differences.
#include <atomic>
#include <iostream>
#include <memory>
#include <thread>
#include <vector>

struct Node {
    int key;
    std::atomic<Node*> next;
    Node(int k) : key(k), next(nullptr) {}
};

// ============ Approach 1: Epoch-Based Reclamation ============
namespace ebr_approach {
    // api: enter() / leave() bracket critical section
    //      retire(node) defers deletion
    void reader_pattern(std::atomic<Node*>& head /*, EBR& ebr, int tid*/) {
        // ebr.enter(tid);
        Node* curr = head.load(std::memory_order_acquire);
        while (curr) {
            // Safe to read curr->key: no one will free curr
            // while any thread is in the current epoch
            [[maybe_unused]] int k = curr->key;
            curr = curr->next.load(std::memory_order_acquire);
        }
        // ebr.leave(tid);
    }
    // Pros: trivial per-access overhead (no per-pointer protection)
    // Cons: one stalled reader blocks all reclamation
}

// ============ Approach 2: Hazard Pointers ============
namespace hp_approach {
    // api: hazard_pointer hp = make_hazard_pointer();
    //      hp.protect(ptr) announces "I'm using this pointer"
    //      hp.reset_protection() clears the announcement
    //      retire(node) checks all hazard pointers before freeing
    void reader_pattern(std::atomic<Node*>& head) {
        // hazard_pointer hp = make_hazard_pointer();
        // Node* curr = hp.protect(head);
        // while (curr) {
        //     int k = curr->key;             // safe: hp protects curr
        //     Node* nxt = curr->next.load(); // read next before protecting
        //     curr = hp.protect(nxt);        // switch protection to next
        // }
        // hp.reset_protection();
    }
    // Pros: bounded memory usage; stalled threads don't block others
    // Cons: atomic store per pointer traversed (expensive on ARM)
}

// ============ Approach 3: User-Space RCU ============
namespace rcu_approach {
    // api: rcu_read_lock() / rcu_read_unlock() bracket read-side
    //      synchronize_rcu() waits for all readers to finish
    //      call_rcu(node, deleter) defers deletion past grace period
    void reader_pattern(std::atomic<Node*>& head) {
        // rcu_read_lock();
        // Node* curr = head.load(std::memory_order_acquire);
        // while (curr) {
        //     int k = curr->key;
        //     curr = curr->next.load(std::memory_order_acquire);
        // }
        // rcu_read_unlock();
    }
    // Pros: near-zero read-side overhead (no atomics in critical section)
    // Cons: complex implementation; write-side must wait for grace period
}

int main() {
    std::cout << "SMR Comparison Summary:\n\n";

    std::cout << "| Property              | EBR         | HP           | RCU         |\n";
    std::cout << "|-----------------------|-------------|--------------|-------------|\n";
    std::cout << "| Read overhead         | 1 atomic    | 1 atomic/ptr | ~0 (fence)  |\n";
    std::cout << "| Memory bound          | Unbounded*  | O(T*H)       | Unbounded*  |\n";
    std::cout << "| Stall tolerance       | Poor        | Excellent    | Varies      |\n";
    std::cout << "| Implementation        | Simple      | Moderate     | Complex     |\n";
    std::cout << "| C++ std support       | No          | C++26 P2530  | No          |\n";
    std::cout << "\n* Unbounded if a thread stalls in a critical section.\n";
    std::cout << "  T = threads, H = hazard pointers per thread.\n";

    return 0;
}
```

**Key insight:** EBR excels when critical sections are short and threads are never preempted for long - common in user-space server applications. Hazard pointers (coming to the standard in C++26 via P2530) are better when threads may stall. RCU dominates in read-heavy kernel workloads but is complex to implement in user-space C++. Choose based on your read/write ratio, stall tolerance requirements, and memory budget.

---

## Notes

- **EBR is not in the C++ standard.** You must implement it yourself or use a library (e.g., `libcds`, `folly::rcu`, `crossbeam-epoch` ports). Hazard pointers are proposed for C++26 (P2530).
- **Critical section length matters.** Keep `enter()`->`leave()` spans as short as possible. Never block (I/O, sleep, mutex) inside an EBR critical section.
- **Amortize reclamation.** Don't call `try_advance()` on every retire. Batch it - e.g., every 100 retires or every millisecond. This reduces contention on the global epoch.
- **Epoch overflow.** Use modular arithmetic (3 epochs cycling 0->1->2->0). Two epoch gaps suffice because a thread in epoch E must leave before E+2 can begin.
- **DEBRA (Distributed Epoch-Based Reclamation)** is a production-grade variant that handles thread blocking by adding per-thread announcement arrays and a quiescent-state mechanism.
- **Testing.** Always run lock-free code under ThreadSanitizer and AddressSanitizer. EBR bugs (premature reclamation) manifest as use-after-free, which ASan catches reliably.
