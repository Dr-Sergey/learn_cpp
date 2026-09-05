# Understand the ABA problem in lock-free programming

**Category:** Concurrency and Parallelism  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/ABA_problem>  

---

## Topic Overview

The **ABA problem** is a subtle bug in lock-free algorithms that use compare-and-swap (CAS). CAS succeeds when the pointer value matches the expected value - but it can't detect if the value was changed to something else and then changed *back*. The reason this trips people up is that CAS only checks bits in memory. It has absolutely no knowledge of history - it can't tell the difference between "this address was never touched" and "this address was freed, reallocated, and reused at the exact same location."

### The ABA Scenario (Lock-Free Stack)

Walk through this scenario carefully - the timeline is the key to understanding why CAS fails here:

```cpp
Initial state:  head -> [A] -> [B] -> [C] -> null

Thread 1:  pop() begins
           reads head=A, reads A.next=B
           (wants to CAS head from A to B)
           ... gets preempted ...

Thread 2:  pop() -> removes A (head=B)
           pop() -> removes B (head=C)
           push(A) -> reuses A's memory, head=[A]->[C]->null

Thread 1:  resumes
           CAS(head, A, B) -> SUCCEEDS! (head still points to A)
           But B was already freed!
           head -> [B(freed)] -> undefined behavior
```

Thread 1 saw `A` when it started and still sees `A` when it resumes - but the world has changed completely in between. `B` is gone, yet Thread 1 happily sets `head` to point at it. The stack is now corrupted.

### Why CAS Can't Detect ABA

The fundamental limitation is that CAS is a bitwise comparison. It knows nothing about the history of a memory location:

```cpp
CAS only checks: "is the bits at this address still the same?"
CAS does NOT check: "has this address been freed and reused?"

Value at &head:
  Time 0: 0x1000 (A)  <- Thread 1 reads this
  Time 1: 0x2000 (B)  <- Thread 2 pops A
  Time 2: 0x3000 (C)  <- Thread 2 pops B
  Time 3: 0x1000 (A') <- Thread 2 pushes new A at SAME address
  Time 4: CAS(&head, 0x1000, 0x2000) -> SUCCESS (A'==A)
           But the stack is now corrupted
```

The address `0x1000` came back, and CAS has no way to know it was ever gone.

### Solutions Overview

Each solution attacks the problem from a different angle. The table below gives you a quick comparison:

| Solution | Mechanism | Overhead | Availability |
| --- | --- | --- | --- |
| Tagged pointer | Version counter in unused bits | Low | Manual |
| Double-width CAS | 128-bit CAS with counter | Medium | x86 (`cmpxchg16b`) |
| Hazard pointers | Prevent reclamation of in-use nodes | Medium | C++26 |
| Epoch-based reclamation | Defer deletion to safe epoch | Low | Libraries |
| Garbage collection | Never reuse addresses | High | Not native C++ |

---

## Self-Assessment

### Q1: Construct a concrete ABA scenario with a compare_exchange-based stack

**Answer:**

This code demonstrates the vulnerable stack and then manually walks through the ABA timeline so you can see exactly how the corruption happens:

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <chrono>
#include <cassert>

// Lock-free stack vulnerable to ABA
template<typename T>
struct Node {
    T data;
    Node* next;
};

template<typename T>
class VulnerableStack {
    std::atomic<Node<T>*> head_{nullptr};

public:
    void push(Node<T>* node) {
        node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(
            node->next, node,
            std::memory_order_release, std::memory_order_relaxed))
            ;
    }

    Node<T>* pop() {
        Node<T>* old_head = head_.load(std::memory_order_acquire);
        while (old_head) {
            // DANGER: between loading old_head and CAS,
            // old_head might be freed and its memory reused!
            Node<T>* next = old_head->next; // <- may read freed memory
            if (head_.compare_exchange_weak(
                old_head, next,
                std::memory_order_release, std::memory_order_relaxed))
                return old_head;
        }
        return nullptr;
    }
};

int main() {
    // === Demonstrate ABA step by step ===
    // We'll simulate the scenario manually

    VulnerableStack<int> stack;

    auto* A = new Node<int>{1, nullptr};
    auto* B = new Node<int>{2, nullptr};
    auto* C = new Node<int>{3, nullptr};

    // Build stack: head -> A -> B -> C
    stack.push(C);
    stack.push(B);
    stack.push(A);
    // State: head -> A -> B -> C -> null

    std::cout << "Initial: A=" << A << " B=" << B << " C=" << C << "\n";

    // Thread 1 simulation: reads head=A, A.next=B, then pauses
    Node<int>* t1_head = A;         // Thread 1 read head
    Node<int>* t1_next = A->next;   // Thread 1 read A.next = B
    std::cout << "Thread1 reads: head=" << t1_head
              << " next=" << t1_next << "\n";

    // Thread 2 simulation: pop A, pop B, push new node at A's old address
    auto* popped_A = stack.pop(); // removes A
    auto* popped_B = stack.pop(); // removes B
    // State: head -> C -> null

    // Simulate memory reuse: reuse A's memory for a new node
    popped_A->data = 999;
    popped_A->next = nullptr;
    stack.push(popped_A); // push "new" A back
    // State: head -> A(999) -> C -> null

    std::cout << "After Thread2: head points to same address as A: "
              << (popped_A == t1_head ? "YES (ABA!)" : "no") << "\n";

    // Thread 1 resumes: CAS(head, A, B) - would succeed!
    // Because head still == A (same address), but:
    // - B is no longer in the stack (it was popped and freed)
    // - A.next is now C, not B
    // - Setting head=B would point to freed memory

    std::cout << "Thread1 would set head=B=" << t1_next
              << " (already popped!) -> CORRUPTION\n";

    // Cleanup
    delete C;
    while (auto* n = stack.pop()) delete n;
    delete popped_B;
}
```

**Explanation:** The CAS in Thread 1's pop() would succeed because `head` still points to the same address as `A`. But `A.next` (which is `B`) was already removed and freed. Setting `head = B` corrupts the stack, pointing to freed memory.

### Q2: Explain how tagged pointers (version counter) prevent the ABA problem

**Answer:**

The insight here is simple: instead of storing just a pointer, store a pointer paired with a monotonically increasing version counter. Now every modification to `head` increments the counter. Even if the address returns to `A`, the tag will be different - so CAS correctly rejects the stale snapshot.

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <vector>
#include <cstdint>

// Tagged pointer: combines a pointer and a version counter
// into a single value that can be atomically CAS'd

template<typename T>
struct TaggedPtr {
    T* ptr;
    uintptr_t tag; // version counter (incremented on every update)

    bool operator==(const TaggedPtr& other) const {
        return ptr == other.ptr && tag == other.tag;
    }
};

template<typename T>
struct ABASafeNode {
    T data;
    ABASafeNode* next;
};

template<typename T>
class ABASafeStack {
    // Uses double-width atomic for tagged pointer
    // On x86-64: 128-bit CAS via cmpxchg16b
    std::atomic<TaggedPtr<ABASafeNode<T>>> head_{{nullptr, 0}};

public:
    void push(ABASafeNode<T>* node) {
        TaggedPtr<ABASafeNode<T>> old_head = head_.load(std::memory_order_relaxed);
        TaggedPtr<ABASafeNode<T>> new_head;
        do {
            node->next = old_head.ptr;
            new_head = {node, old_head.tag + 1}; // increment version
        } while (!head_.compare_exchange_weak(
            old_head, new_head,
            std::memory_order_release, std::memory_order_relaxed));
    }

    ABASafeNode<T>* pop() {
        TaggedPtr<ABASafeNode<T>> old_head = head_.load(std::memory_order_acquire);
        TaggedPtr<ABASafeNode<T>> new_head;
        while (old_head.ptr) {
            new_head = {old_head.ptr->next, old_head.tag + 1};
            if (head_.compare_exchange_weak(
                old_head, new_head,
                std::memory_order_release, std::memory_order_relaxed))
                return old_head.ptr;
        }
        return nullptr;
    }
};

int main() {
    // === Why tagged pointers prevent ABA ===
    //
    // Standard CAS:
    //   CAS(head, A, B) succeeds if head == A (address only)
    //   ABA: head goes A -> B -> A, CAS sees A -> succeeds (BAD!)
    //
    // Tagged CAS:
    //   CAS(head, {A,5}, {B,6}) succeeds if head == {A,5}
    //   ABA: head goes {A,5} -> {B,6} -> {A,7}
    //   CAS sees {A,7} != {A,5} -> FAILS (tag mismatch!)
    //
    //   Even though the pointer is the same (A), the version
    //   counter has changed (5->7), so CAS correctly fails.

    ABASafeStack<int> stack;

    auto* nodes = new ABASafeNode<int>[3]{{1,nullptr},{2,nullptr},{3,nullptr}};
    stack.push(&nodes[2]);
    stack.push(&nodes[1]);
    stack.push(&nodes[0]);

    // Pop all
    while (auto* n = stack.pop()) {
        std::cout << "Popped: " << n->data << "\n";
    }
    // Output: Popped: 1, 2, 3

    delete[] nodes;

    // NOTE: On 64-bit systems, you can also pack the tag into
    // unused high bits of the pointer (x86-64 uses only 48 bits):
    //
    //   uintptr_t tagged = (uintptr_t)ptr | (tag << 48);
    //   T* real_ptr = (T*)(tagged & 0x0000FFFFFFFFFFFF);
    //   uint16_t real_tag = (uint16_t)(tagged >> 48);
    //
    // This allows using regular 64-bit CAS instead of 128-bit.
}
```

**Explanation:** A tagged pointer pairs the pointer with a monotonically increasing version counter. Even if the same pointer value reappears (ABA), the tag will have changed. CAS compares *both* the pointer and the tag, so it correctly detects the intermediate modification and fails.

### Q3: Show how hazard pointers solve ABA by deferring reclamation

**Answer:**

Hazard pointers take a completely different approach. Instead of detecting that a pointer was reused, they *prevent* reuse from happening in the first place. Before a thread reads a pointer and dereferences it, it publishes that pointer in a globally-visible "hazard pointer" slot. No other thread is allowed to free a node while it appears in any hazard pointer slot. So the address can never be recycled while anyone might still be using it - and without address recycling, ABA can't happen.

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <algorithm>
#include <functional>

// Hazard pointers solve ABA by PREVENTING address reuse

// === Simplified Hazard Pointer System ===
template<typename T>
class HazardPtrSystem {
    static constexpr int MAX_THREADS = 8;
    std::atomic<T*> hp_array_[MAX_THREADS]{}; // one HP per thread
    static std::atomic<int> tid_counter_;

    struct Retired { T* ptr; };
    // Per-thread retired lists (simplified - not thread_local for demo)
    std::vector<Retired> retired_[MAX_THREADS];

    static int get_tid() {
        thread_local int id = tid_counter_.fetch_add(1);
        return id;
    }

public:
    // PROTECT: announce "I'm using this pointer"
    T* protect(const std::atomic<T*>& source) {
        int tid = get_tid();
        T* ptr;
        do {
            ptr = source.load(std::memory_order_acquire);
            hp_array_[tid].store(ptr, std::memory_order_release);
        } while (source.load(std::memory_order_acquire) != ptr);
        return ptr;
    }

    void unprotect() {
        hp_array_[get_tid()].store(nullptr, std::memory_order_release);
    }

    // RETIRE: defer deletion until safe
    void retire(T* ptr) {
        int tid = get_tid();
        retired_[tid].push_back({ptr});

        // Try to reclaim when we have enough retired nodes
        if (retired_[tid].size() >= MAX_THREADS * 2) {
            // Collect all protected pointers
            std::vector<T*> protected_ptrs;
            for (int i = 0; i < MAX_THREADS; ++i) {
                T* hp = hp_array_[i].load(std::memory_order_acquire);
                if (hp) protected_ptrs.push_back(hp);
            }

            // Delete unprotected retired nodes
            auto& list = retired_[tid];
            list.erase(std::remove_if(list.begin(), list.end(),
                [&](const Retired& r) {
                    bool safe = std::find(protected_ptrs.begin(),
                        protected_ptrs.end(), r.ptr) == protected_ptrs.end();
                    if (safe) delete r.ptr;
                    return safe;
                }), list.end());
        }
    }
};

template<typename T>
std::atomic<int> HazardPtrSystem<T>::tid_counter_{0};

// === Lock-free stack protected by hazard pointers (ABA-safe) ===
struct Node {
    int data;
    Node* next;
};

HazardPtrSystem<Node> hp;
std::atomic<Node*> stack_head{nullptr};

void safe_push(int val) {
    auto* node = new Node{val, nullptr};
    node->next = stack_head.load(std::memory_order_relaxed);
    while (!stack_head.compare_exchange_weak(
        node->next, node,
        std::memory_order_release, std::memory_order_relaxed))
        ;
}

int safe_pop() {
    Node* old_head;
    while (true) {
        old_head = hp.protect(stack_head);  // <- PROTECT before reading
        if (!old_head) { hp.unprotect(); return -1; }

        Node* next = old_head->next;  // safe: old_head can't be freed
        if (stack_head.compare_exchange_weak(
            old_head, next,
            std::memory_order_release, std::memory_order_relaxed)) {
            int val = old_head->data;
            hp.unprotect();         // <- done reading
            hp.retire(old_head);    // <- defer deletion (NOT immediate delete!)
            return val;
            // old_head won't be freed until no thread's HP points to it
            // -> it can NEVER be reallocated while another thread reads it
            // -> ABA is IMPOSSIBLE
        }
    }
}

int main() {
    constexpr int N = 5;
    for (int i = 0; i < N; ++i) safe_push(i);

    std::vector<std::thread> threads;
    for (int t = 0; t < 4; ++t) {
        threads.emplace_back([&] {
            for (int i = 0; i < 2; ++i) {
                int val = safe_pop();
                if (val >= 0)
                    std::cout << "Thread " << std::this_thread::get_id()
                              << " popped " << val << "\n";
            }
        });
    }
    for (auto& t : threads) t.join();

    // Each value popped exactly once, no ABA corruption
    // The hazard pointer ensures nodes are never freed while
    // any thread might still be dereferencing them.
}
```

**Explanation:** Hazard pointers solve ABA by a fundamentally different mechanism than tagged pointers: instead of detecting that a pointer was reused, they *prevent* reuse entirely. A retired node stays allocated until no thread is reading it. Since the address is never recycled, the ABA scenario (same address, different node) cannot occur.

---

## Notes

- **ABA only matters with CAS + manual memory management.** Languages with garbage collection (Java, Go) never have ABA because freed objects aren't reused at the same address.
- **Tagged pointers** are the simplest fix and work well when the tag counter doesn't wrap around. A 16-bit counter wraps after 65,536 operations - enough for most practical scenarios.
- **Double-width CAS (`cmpxchg16b`)** is available on x86-64 but requires 16-byte alignment. Check with `std::atomic<TaggedPtr>::is_lock_free()`.
- **ABA in practice:** Surprisingly rare with modern allocators (jemalloc, tcmalloc) because freed memory isn't immediately reused. But it's still a *correctness* bug - it will happen eventually under stress.
- **TSan cannot detect ABA** - it's a logical bug, not a data race.
- Compile with `-std=c++17 -O2 -pthread -mcx16` (for 128-bit CAS on x86).
