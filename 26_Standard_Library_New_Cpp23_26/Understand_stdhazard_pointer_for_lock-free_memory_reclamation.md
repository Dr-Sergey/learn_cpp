# Understand std::hazard_pointer for lock-free memory reclamation

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/thread/hazard_pointer>  

---

## Topic Overview

Lock-free data structures face **the reclamation problem**: a thread may `delete` a node while another thread still holds a raw pointer to it. This is the kind of bug that is nearly impossible to catch in testing because it requires a precise interleaving of thread operations to trigger.

**Hazard pointers** (C++26, `<hazard_pointer>`) solve this by letting each reader *announce* which pointers it is currently accessing. A writer that wants to reclaim memory checks all published hazard pointers and defers deletion until no reader holds that address. The "hazard" in the name refers to the pointer itself being dangerous to delete - the hazard pointer slot is the reader's way of saying "this address is still in use by me."

### Protocol (3 steps)

Here is the flow at a glance. The reader publishes its intent, verifies the world has not changed under it, and only then accesses the data safely:

```cpp
Reader thread                             Writer thread
-------------                             -------------

1. hp.protect(ptr)  <- publish pointer     1. Swap ptr out (CAS)
2. Verify ptr still current               2. hp_obj.retire()
   (re-read atomic, compare)                 -> deferred delete

3. Access *ptr safely                     3. Scan hazard list:
4. hp.reset_protection()                     if no reader holds ptr -> delete
```

The reason step 2 (verify) exists is subtle but critical. There is a window between when you read the atomic pointer and when you publish it to the hazard slot. In that window, a writer could have swapped the pointer out and retired the old node. The verify step catches this: if the atomic has changed between your initial load and your hazard slot publish, you retry the whole sequence. Once you confirm the atomic still holds your pointer after you have published, you are safe - any retire happening after that point will see your hazard slot and defer deletion.

### Key API (C++26)

| API | Purpose |
| --- | --- |
| `std::hazard_pointer hp = make_hazard_pointer()` | Acquire a per-thread hazard slot |
| `hp.protect(src)` | Atomically load `src` and publish it |
| `hp.reset_protection()` | Clear the hazard slot |
| `obj->retire()` | Schedule deferred reclamation of `obj` |
| `std::hazard_pointer_obj_base<T>` | CRTP base; adds `retire()` to your node |

---

## Self-Assessment

### Q1: Explain the hazard pointer protocol: protect a pointer, verify it is still current, then access

The code below implements a minimal hazard slot manually (since `<hazard_pointer>` is C++26 and may not be in your toolchain yet). Study the `protect` loop - that loop is the entire protocol compressed into a few lines:

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>
#include <cassert>
// Note: <hazard_pointer> is C++26; this demonstrates the protocol manually.

// Simplified hazard pointer slot
// In real C++26: std::hazard_pointer hp = std::make_hazard_pointer();
template<typename T>
struct HazardSlot {
    std::atomic<T*> protected_ptr{nullptr};

    // Step 1: Protect - publish the pointer we're about to read
    T* protect(const std::atomic<T*>& source) {
        T* ptr = nullptr;
        do {
            ptr = source.load(std::memory_order_acquire);
            protected_ptr.store(ptr, std::memory_order_release);
            // Step 2: Verify - re-read source; if it changed, retry
        } while (ptr != source.load(std::memory_order_acquire));
        return ptr;  // Now safe to dereference
    }

    // Step 4: Reset - allow reclamation
    void reset() {
        protected_ptr.store(nullptr, std::memory_order_release);
    }
};

struct Node {
    int value;
    Node* next;
};

int main() {
    std::atomic<Node*> head{new Node{42, nullptr}};
    HazardSlot<Node> hp;

    // Reader: protect -> verify (inside protect loop) -> access
    Node* safe = hp.protect(head);
    if (safe) {
        std::cout << "Safely read value: " << safe->value << '\n';  // 42
    }
    hp.reset();

    // Cleanup
    delete head.load();
    std::cout << "Protocol: protect -> verify -> access -> reset\n";
}
```

**The three guarantees:**

1. **Protect** - store the pointer in a hazard slot visible to all threads.
2. **Verify** - re-read the atomic source to confirm it has not been swapped out.
3. **Access** - now safe because any `retire()` will see your hazard slot and defer deletion.

Once `reset()` is called, you are saying you are done with this pointer and the system is free to delete it if no one else is protecting it.

### Q2: Implement a lock-free linked list pop using hazard pointers to safely reclaim nodes

This is a more complete example showing how hazard pointers integrate into a real lock-free stack. Pay attention to the protect-verify loop inside `pop` - it is the same pattern from Q1, now embedded in a concurrent data structure:

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>
#include <array>
#include <algorithm>

// Manual hazard pointer infrastructure
// (C++26 provides std::hazard_pointer; this is the equivalent logic)

constexpr int MAX_THREADS = 8;

template<typename T>
struct HazardPointerDomain {
    struct HPRecord {
        std::atomic<T*> hp{nullptr};
        std::atomic<bool> active{false};
    };
    std::array<HPRecord, MAX_THREADS> records;

    HPRecord& acquire() {
        for (auto& r : records) {
            bool expected = false;
            if (r.active.compare_exchange_strong(expected, true))
                return r;
        }
        throw std::runtime_error("No free hazard pointer slots");
    }

    void release(HPRecord& r) {
        r.hp.store(nullptr, std::memory_order_release);
        r.active.store(false, std::memory_order_release);
    }

    bool is_protected(T* ptr) const {
        for (auto& r : records) {
            if (r.active.load(std::memory_order_acquire) &&
                r.hp.load(std::memory_order_acquire) == ptr)
                return true;
        }
        return false;
    }
};

// Lock-free stack with safe reclamation
struct Node {
    int data;
    std::atomic<Node*> next;
    Node(int d) : data(d), next(nullptr) {}
};

class LockFreeStack {
    std::atomic<Node*> head_{nullptr};
    HazardPointerDomain<Node> hp_domain_;

public:
    void push(int value) {
        Node* new_node = new Node(value);
        Node* old_head = head_.load(std::memory_order_relaxed);
        do {
            new_node->next.store(old_head, std::memory_order_relaxed);
        } while (!head_.compare_exchange_weak(old_head, new_node,
                    std::memory_order_release, std::memory_order_relaxed));
    }

    bool pop(int& result) {
        auto& hp_rec = hp_domain_.acquire();
        Node* old_head;

        // Protect-verify loop
        do {
            old_head = head_.load(std::memory_order_acquire);
            hp_rec.hp.store(old_head, std::memory_order_release);
            // Verify: re-read head; if changed, retry
        } while (old_head != head_.load(std::memory_order_acquire));

        if (!old_head) {
            hp_domain_.release(hp_rec);
            return false;
        }

        Node* next = old_head->next.load(std::memory_order_relaxed);
        if (head_.compare_exchange_strong(old_head, next,
                std::memory_order_acq_rel)) {
            result = old_head->data;
            hp_domain_.release(hp_rec);

            // Retire: delete only if no other thread protects this node
            if (!hp_domain_.is_protected(old_head)) {
                delete old_head;
            }
            // else: defer to a retire list (simplified here)
            return true;
        }

        hp_domain_.release(hp_rec);
        return pop(result);  // Retry
    }
};

int main() {
    LockFreeStack stack;
    stack.push(10);
    stack.push(20);
    stack.push(30);

    int val;
    while (stack.pop(val)) {
        std::cout << "Popped: " << val << '\n';
    }
    // Output: 30, 20, 10 (LIFO order)
}
```

### Q3: Compare hazard pointers with epoch-based reclamation for latency vs throughput tradeoffs

Hazard pointers and epoch-based reclamation (EBR) are the two dominant approaches to lock-free memory management. They make different tradeoffs, and understanding those tradeoffs tells you which to reach for:

| Aspect | Hazard Pointers | Epoch-Based Reclamation (EBR) |
| --- | --- | --- |
| **Reclamation latency** | Bounded - O(H x P) deferred nodes | Unbounded - blocked by slowest reader |
| **Throughput** | Moderate - scanning hazard list per retire | High - just increment a counter |
| **Per-op overhead** | Store + fence per access | Epoch enter/exit (very cheap) |
| **Stalled thread impact** | Blocks only nodes that specific thread protects | Blocks ALL reclamation for the epoch |
| **Memory bound** | O(T^2) where T = thread count | Unbounded if a thread stalls |
| **Implementation** | More complex | Simpler |
| **C++ standard** | `std::hazard_pointer` (C++26) | Not standardized (libraries) |

The comment below shows the mental model for each approach side by side. Notice how EBR's stall behavior differs sharply from hazard pointers - one stalled thread in EBR blocks everybody's garbage collection:

```cpp
// Conceptual comparison:

// Hazard Pointer: per-access protection
// Reader:
//   hp.protect(ptr);       // announce "I'm using ptr"
//   use(*ptr);
//   hp.reset_protection(); // done
// Writer:
//   old->retire();         // checks ALL hp slots before delete
// -> Worst case: T threads protect T nodes each -> T*T deferred nodes

// Epoch-Based: generation counting
// Reader:
//   guard.enter();         // increment local epoch (1 atomic)
//   use(*ptr);
//   guard.leave();         // decrement
// Writer:
//   retire_list[current_epoch].push(old);
//   if (all_threads_past(epoch - 2))
//       delete_epoch(epoch - 2);
// -> Worst case: 1 stalled thread in epoch 5 -> epochs 6,7,8... pile up

// When to choose:
// - Hard real-time, bounded memory -> Hazard Pointers
// - Maximum throughput, cooperative threads -> EBR
// - Mixed: use EBR + timeout that falls back to HP for stalled threads
```

The reason hazard pointers give bounded memory usage is that the number of deferred nodes is bounded by the number of hazard slots (threads times slots per thread). EBR has no such bound - if one thread goes to sleep while holding an epoch guard, the entire system accumulates garbage indefinitely.

---

## Notes

- `std::hazard_pointer` is expected in C++26; currently available in Folly (`folly::hazard_pointer`) and libcds.
- Each thread typically needs 1-2 hazard pointer slots per concurrent operation.
- Hazard pointers were patent-encumbered (IBM patent expired approximately 2024); the standard sidesteps historical concerns.
- Combine with `std::atomic<std::shared_ptr<T>>` for simpler cases where performance is less critical.
- The standard also proposes `std::rcu` (Read-Copy-Update) as a complementary mechanism for read-heavy workloads.
