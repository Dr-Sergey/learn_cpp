# Understand std::hazard_pointer and std::rcu (C++26 preview) for lock-free reclamation

**Category:** Concurrency and Parallelism  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/thread/hazard_pointer>  

---

## Topic Overview

In lock-free data structures, the fundamental problem is: **when can you safely delete a node that was just removed?** Other threads might still be reading it. Two solutions are coming to standard C++ in C++26:

### The Reclamation Problem

```cpp

Thread A: pop() → reads node X → gets node X's data
Thread B: pop() → removes node X → deletes node X
Thread A: accesses node X → USE-AFTER-FREE! 💥

Timeline:
  Thread A:  read X ──────── use X.data
  Thread B:       remove X ── delete X
                              ↑
                        crash/corruption

```

### Hazard Pointers vs RCU

| Feature | Hazard Pointers | RCU (Read-Copy-Update) |
| --- | --- | --- |
| Read overhead | Set/clear per-pointer protection | Enter/exit read-side critical section |
| Write overhead | Scan all hazard pointers before reclaim | Wait for grace period |
| Memory bound | O(threads × pointers_per_thread) | Unbounded during grace period |
| Latency | Deterministic reclamation | Batched/deferred |
| Best for | Few shared pointers, frequent updates | Read-mostly data structures |
| Origin | Maged Michael (2004) | Linux kernel (2002) |

### Hazard Pointer Lifecycle

```cpp

1. PROTECT:  hp.protect(ptr)  → "I'm reading this, don't delete it"
2. ACCESS:   use ptr->data     → safe, pointer is protected
3. RELEASE:  hp.reset_protection() → "I'm done with this pointer"
4. RETIRE:   retire(old_ptr)  → "Delete this when safe"
5. RECLAIM:  scan all hazard pointers, delete unprotected retired nodes

```

---

## Self-Assessment

### Q1: Explain the ABA problem and why hazard pointers are one solution

**Answer:**

```cpp

THE ABA PROBLEM:
═══════════════
Lock-free algorithms use CAS (compare-and-swap) to update pointers.
CAS succeeds if the pointer value hasn't changed. But what if:

1. Thread A reads head = X (node X points to Y)
2. Thread A is preempted
3. Thread B pops X, pops Y, pushes a NEW node at the SAME address as X
4. Thread A resumes, CAS sees head == X → succeeds!

   But X.next is now GARBAGE (X was deleted and reallocated)

    head → [X|→Y] → [Y|→Z] → [Z|→∅]
    
    Thread A reads: head=X, X.next=Y
    Thread B: pop X, pop Y, push "new X" at same address
    
    head → [X'|→?] → ???
    
    Thread A: CAS(head, X, Y) → succeeds (X' has same address as X!)
    head → [Y] but Y was already deleted → 💥

WHY HAZARD POINTERS SOLVE IT:
═════════════════════════════
Thread A protects pointer X with a hazard pointer BEFORE reading it.
Thread B cannot RECLAIM X while Thread A's hazard pointer is set.
Since X is never freed while A holds it, X can never be reallocated
to a different node → ABA is impossible.

    Thread A: hp.protect(X)  → X cannot be freed
    Thread B: retire(X)      → X goes to "retired" list, NOT freed
    Thread B: allocate()     → gets a DIFFERENT address, never X
    Thread A: CAS(head, X, X.next) → safe, X still valid
    Thread A: hp.reset()     → X can now be freed

```

**Key insight:** Hazard pointers prevent reclamation, which prevents address reuse, which prevents ABA. They don't prevent the CAS from succeeding with the "same" pointer — they ensure the pointer's node hasn't been freed and reallocated.

### Q2: Show the lifecycle of a hazard pointer: protect, check, and reclaim

**Answer:**

```cpp

#include <atomic>
#include <vector>
#include <thread>
#include <iostream>
#include <cassert>
#include <functional>

// === Simplified hazard pointer implementation for demonstration ===
// (The real C++26 API is std::hazard_pointer from <hazard_pointer>)

template<typename T>
class HazardPointerDomain {
    static constexpr int MAX_THREADS = 16;
    static constexpr int HP_PER_THREAD = 2;

    // Each thread's hazard pointer slots (visible to all threads)
    std::atomic<T*> hazard_ptrs_[MAX_THREADS * HP_PER_THREAD]{};

    // Per-thread retired lists
    struct RetiredNode {
        T* ptr;
        std::function<void(T*)> deleter;
    };
    thread_local static std::vector<RetiredNode> retired_;
    static std::atomic<int> thread_counter_;

    static int get_thread_id() {
        thread_local int id = thread_counter_.fetch_add(1);
        return id;
    }

public:
    // STEP 1: PROTECT — announce "I'm reading this pointer"
    T* protect(int slot, const std::atomic<T*>& source) {
        int id = get_thread_id();
        T* ptr;
        do {
            ptr = source.load(std::memory_order_acquire);
            hazard_ptrs_[id * HP_PER_THREAD + slot]
                .store(ptr, std::memory_order_release);
            // Re-check: source might have changed between load and store
        } while (source.load(std::memory_order_acquire) != ptr);
        return ptr;
    }

    // STEP 2: RELEASE — announce "I'm done with this pointer"
    void release(int slot) {
        int id = get_thread_id();
        hazard_ptrs_[id * HP_PER_THREAD + slot]
            .store(nullptr, std::memory_order_release);
    }

    // STEP 3: RETIRE — "delete this when safe"
    void retire(T* ptr) {
        retired_.push_back({ptr, [](T* p) { delete p; }});
        if (retired_.size() > MAX_THREADS * HP_PER_THREAD * 2) {
            reclaim(); // batch reclamation for efficiency
        }
    }

    // STEP 4: RECLAIM — scan hazard pointers, delete unprotected nodes
    void reclaim() {
        // Collect all currently protected pointers
        std::vector<T*> protected_ptrs;
        for (int i = 0; i < MAX_THREADS * HP_PER_THREAD; ++i) {
            T* p = hazard_ptrs_[i].load(std::memory_order_acquire);
            if (p) protected_ptrs.push_back(p);
        }

        // Delete retired nodes that are NOT protected
        auto it = retired_.begin();
        while (it != retired_.end()) {
            bool is_protected = false;
            for (auto* hp : protected_ptrs) {
                if (hp == it->ptr) { is_protected = true; break; }
            }
            if (!is_protected) {
                it->deleter(it->ptr); // safe to delete
                it = retired_.erase(it);
            } else {
                ++it; // keep in retired list — still in use
            }
        }
    }
};

template<typename T>
thread_local std::vector<typename HazardPointerDomain<T>::RetiredNode>
    HazardPointerDomain<T>::retired_;

template<typename T>
std::atomic<int> HazardPointerDomain<T>::thread_counter_{0};

// === Usage: lock-free stack with hazard pointer protection ===
template<typename T>
struct Node {
    T data;
    Node* next;
};

int main() {
    HazardPointerDomain<Node<int>> hp_domain;
    std::atomic<Node<int>*> head{nullptr};

    // Push some nodes
    for (int i = 0; i < 5; ++i) {
        auto* n = new Node<int>{i, head.load()};
        head.store(n);
    }

    // Safe pop using hazard pointer:
    auto safe_pop = [&]() -> int {
        Node<int>* old_head;
        do {
            // PROTECT the head pointer
            old_head = hp_domain.protect(0, head);
            if (!old_head) { hp_domain.release(0); return -1; }
        } while (!head.compare_exchange_weak(old_head, old_head->next));

        int value = old_head->data;
        hp_domain.release(0);    // RELEASE protection
        hp_domain.retire(old_head); // RETIRE for deferred deletion
        return value;
    };

    for (int i = 0; i < 5; ++i)
        std::cout << "Popped: " << safe_pop() << "\n";
    // Output: Popped: 4, 3, 2, 1, 0
}

```

**Explanation:** The protect-access-release-retire lifecycle ensures no node is freed while any thread might be reading it. The reclaim step scans all hazard pointer slots to find which retired nodes are safe to delete. This is the core mechanism that makes lock-free data structures safe without garbage collection.

### Q3: Compare hazard pointers with epoch-based reclamation (EBR) for throughput vs latency

**Answer:**

```cpp

═══════════════════════════════════════════════════════════════
HAZARD POINTERS vs EPOCH-BASED RECLAMATION (EBR) vs RCU
═══════════════════════════════════════════════════════════════

HAZARD POINTERS:
  Reader:  hp.protect(ptr); use(ptr); hp.release();
  Writer:  retire(old_ptr);
  
  + Bounded memory: at most O(T×H + R) nodes unreclaimed

    (T=threads, H=hazard_ptrs/thread, R=retired_list_threshold)

  + Deterministic latency — reclaim happens in known bounds
  + Per-pointer granularity — precise protection
  - Read overhead: atomic store + atomic load per protect() call
  - Scan cost: must check all hazard pointer slots during reclaim
  
EPOCH-BASED RECLAMATION (EBR):
  Reader:  enter_epoch(); use(ptrs); leave_epoch();
  Writer:  retire(old_ptr); advance_epoch_if_safe();
  
  + Near-zero read overhead — just increment a counter
  + Higher throughput for read-heavy workloads
  - UNBOUNDED memory if one thread stalls in an epoch

    (all retired nodes from that epoch accumulate)

  - One slow reader blocks ALL reclamation
  
RCU (Read-Copy-Update):
  Reader:  rcu_read_lock(); use(ptrs); rcu_read_unlock();
  Writer:  publish(new_version); synchronize_rcu(); delete(old);
  
  + Zero read-side overhead (no atomic operations)
  + Best for read-dominated workloads (99% reads)
  - Writers must wait for grace period (all readers finish)
  - Memory grows during grace period

COMPARISON TABLE:
┌──────────────────┬─────────────┬──────────────┬──────────────┐
│                  │ Hazard Ptrs │ EBR          │ RCU          │
├──────────────────┼─────────────┼──────────────┼──────────────┤
│ Read overhead    │ Medium      │ Very low     │ Near zero    │
│ Write overhead   │ Scan all HP │ Check epochs │ Grace period │
│ Memory bound     │ Bounded     │ Unbounded    │ Unbounded    │
│ Stalled reader   │ Fine        │ Blocks all   │ Blocks writer│
│ Throughput       │ Good        │ Better       │ Best reads   │
│ Implementation   │ Complex     │ Moderate     │ Complex      │
│ C++ standard     │ C++26       │ Not standard │ C++26        │
└──────────────────┴─────────────┴──────────────┴──────────────┘

WHEN TO USE WHICH:

- Hazard pointers: real-time systems where memory must be bounded
- EBR: high-throughput servers where stalls are unlikely
- RCU: read-dominated data structures (routing tables, config)

```

---

## Notes

- **C++26 API:** `std::hazard_pointer` from `<hazard_pointer>` and `std::rcu_obj_base` from `<rcu>`. Not yet available in any production compiler as of 2024.
- **Existing libraries:** Folly (Facebook) provides `folly::hazard_pointer`; libcds provides epoch-based and hazard pointer implementations.
- **Hazard pointer overhead:** Each `protect()` involves 2 atomic operations (store + verify load). For data structures with many pointers (trees), this cost multiplies.
- **RCU in Linux kernel:** Used extensively for routing tables, file system structures, and module lists. The kernel implementation blocks in `synchronize_rcu()` waiting for all readers.
- **Hybrid approaches:** Some systems use EBR for fast paths and fall back to hazard pointers when a thread is about to block.
- These features require `-std=c++26` (when available) or library alternatives.
