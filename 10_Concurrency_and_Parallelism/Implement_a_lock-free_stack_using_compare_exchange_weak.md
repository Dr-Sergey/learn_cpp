# Implement a lock-free stack using compare_exchange_weak

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange>  

---

## Topic Overview

A **lock-free stack** (also called a Treiber stack, after its inventor) uses `std::atomic<Node*>` and `compare_exchange_weak` (CAS) instead of mutexes. Multiple threads can push and pop concurrently without ever blocking - if one thread is delayed or preempted, the others still make forward progress.

The reason this trips people up is that "lock-free" does not mean "one instruction." It means the data structure's progress guarantee is provided by hardware atomics rather than OS blocking. The CAS retry loop is the mechanism that makes it work: you read the current state, prepare your change, and then atomically ask "is the state still what I read? If so, apply my change. If not, retry." Only one thread's CAS can win at a time, but no thread is ever blocked - only momentarily retrying.

### Treiber Stack Algorithm

```cpp
Push(value):                          Pop():
  new_node->next = head.load()         old_head = head.load()
  while (!head.CAS(new_node->next,     while (old_head &&
              new_node))                    !head.CAS(old_head,
    ; // retry (another thread won)              old_head->next))
                                            ; // retry
                                        return old_head->data

CAS = compare_exchange_weak:
  "If head still equals expected, set head = desired and return true.
   Otherwise, update expected to current head and return false."
```

### Visual: Concurrent Push

Here is a concrete example of two threads pushing simultaneously. Thread 2's first CAS fails because Thread 1 got there first, but Thread 2 retries and succeeds:

```cpp
Thread 1: push(A)           Thread 2: push(B)

head -> [C] -> [D] -> null    head -> [C] -> [D] -> null
A.next = C                  B.next = C
CAS(head, C->A) success     CAS(head, C->B) fail! (head is now A)
head -> [A] -> [C] -> [D]   B.next = A  (updated by CAS)
                            CAS(head, A->B) success
                            head -> [B] -> [A] -> [C] -> [D]
```

---

## Self-Assessment

### Q1: Build a Treiber stack using atomic<Node*> and compare_exchange_weak for push and pop

**Answer:**

Read through the push and pop implementations carefully - the key detail is that `compare_exchange_weak` takes `new_node->next` as the "expected" parameter, and on failure it automatically updates that value to the current head. That means the retry loop does not need to reload the head manually; the CAS does it for you:

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>
#include <optional>

template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T val) : data(std::move(val)), next(nullptr) {}
    };

    std::atomic<Node*> head_{nullptr};

public:
    void push(T value) {
        Node* new_node = new Node(std::move(value));
        new_node->next = head_.load(std::memory_order_relaxed);

        // CAS loop: try to set head to new_node
        // If head changed since we read it, CAS updates new_node->next
        // to the current head and we retry
        while (!head_.compare_exchange_weak(
            new_node->next,  // expected (updated on failure)
            new_node,        // desired
            std::memory_order_release,
            std::memory_order_relaxed)) {
            // Retry - another thread pushed/popped between load and CAS
        }
    }

    std::optional<T> pop() {
        Node* old_head = head_.load(std::memory_order_relaxed);

        while (old_head) {
            // Try to advance head to old_head->next
            if (head_.compare_exchange_weak(
                old_head,           // expected
                old_head->next,     // desired
                std::memory_order_acquire,
                std::memory_order_relaxed)) {

                T value = std::move(old_head->data);
                delete old_head; // NOTE: simplified - has ABA issues (see Q3)
                return value;
            }
            // CAS failed - old_head updated to current head, retry
        }
        return std::nullopt; // stack is empty
    }

    ~LockFreeStack() {
        while (pop()) {} // drain remaining nodes
    }
};

int main() {
    LockFreeStack<int> stack;

    // Push from multiple threads
    std::vector<std::thread> pushers;
    for (int t = 0; t < 4; ++t) {
        pushers.emplace_back([&, t] {
            for (int i = 0; i < 1000; ++i)
                stack.push(t * 1000 + i);
        });
    }
    for (auto& t : pushers) t.join();

    // Pop all - should get exactly 4000 items
    int count = 0;
    while (stack.pop()) ++count;
    std::cout << "Popped: " << count << " items\n";
    // Output: Popped: 4000 items
}
```

The push operation creates a new node, sets its `next` to the current head, then attempts a CAS to swing the head pointer. If another thread modified head between our load and CAS, the CAS fails, updates `new_node->next` to the actual current head, and we retry. Pop works similarly - we try to advance head to `head->next`.

### Q2: Explain why compare_exchange_weak is preferred over strong in a retry loop

**Answer:**

This is one of those "why does this even exist" questions that has a genuinely satisfying answer once you understand the hardware. On load-linked/store-conditional architectures (ARM, POWER), there is no single atomic read-modify-write instruction for CAS. Instead, the hardware provides a pair of instructions: `LL` loads a value and sets a "reservation," and `SC` stores a new value only if the reservation is still valid. If anything disturbs the reservation - another core writing to that cache line, an interrupt, even just cache pressure - the `SC` fails, even if the value has not actually changed. That is a spurious failure.

`compare_exchange_weak` exposes this spurious-failure possibility directly. `compare_exchange_strong` hides it by internally looping until the failure is genuine (value mismatch) rather than spurious:

```cpp
#include <atomic>
#include <iostream>

int main() {
    std::atomic<int> x{42};
    int expected = 42;

    // compare_exchange_WEAK:
    // - May SPURIOUSLY fail even if x == expected
    // - Returns false without performing the exchange
    // - On some architectures (ARM, POWER), this maps to LL/SC instructions
    //   which can fail due to cache contention, not actual value change
    //
    // compare_exchange_STRONG:
    // - Only fails if x != expected (no spurious failures)
    // - Internally, it's a LOOP around the weak version:
    //   while (!weak(expected, desired)) {
    //       if (expected != original_expected) return false; // genuine fail
    //   }
    //   return true;

    // IN A RETRY LOOP (like Treiber stack push/pop):
    //
    //   while (!x.compare_exchange_weak(expected, desired)) { }
    //
    // We're ALREADY in a loop, so spurious failures just cause an extra iteration.
    // Using "strong" would add an INNER loop (to eliminate spurious failures)
    // inside our OUTER loop - redundant work!

    // WEAK in a loop (optimal):
    while (!x.compare_exchange_weak(expected, 99)) {
        // Retry - expected is auto-updated
    }
    std::cout << "x = " << x.load() << "\n"; // 99

    // STRONG without a loop (when you need exactly one attempt):
    x.store(42);
    expected = 42;
    bool success = x.compare_exchange_strong(expected, 100);
    std::cout << "Strong success: " << std::boolalpha << success << "\n"; // true

    // SUMMARY:
    // Use WEAK when: you have a retry loop (CAS loops, lock-free algorithms)
    // Use STRONG when: you need a single attempt with a definitive result
    //                  (e.g., conditional update without retry)

    // PERFORMANCE on ARM/POWER:
    //   weak  -> single LL/SC pair -> fast
    //   strong -> LL/SC loop until success or genuine failure -> slower per call
    // On x86: both compile to the same CMPXCHG instruction (no difference)
}
```

The rule of thumb is simple: if you already have a retry loop (which every lock-free algorithm does), use `compare_exchange_weak`. If you need a single definitive "did this exchange happen or not" without a loop, use `compare_exchange_strong`.

### Q3: Show the ABA risk in the Treiber stack and how to mitigate it

**Answer:**

The ABA problem is subtle enough that it has tripped up experienced programmers. The reason it trips people up is that CAS only checks the *value* of the pointer, not whether the *meaning* of that pointer has changed. Here is the scenario:

Thread 1 reads the head pointer (value A), then gets preempted. While it is sleeping, Thread 2 pops A, pops B, and then pushes a new node back to address A (perhaps through a memory allocator that reused the same address). Thread 1 wakes up, its CAS sees "head is still A" - true! - and succeeds. But the A that Thread 1 linked against is not the same A it read. The stack is now corrupted.

```cpp
#include <atomic>
#include <iostream>
#include <cstdint>
#include <optional>
#include <utility>

// === THE ABA PROBLEM ===
//
// Thread 1 (pop):                Thread 2 (concurrent):
// 1. Read head -> A
// 2. Read A->next -> B
//    (About to CAS head from A to B)
//    ...gets preempted...
//                                 3. Pop A (head = B)
//                                 4. Pop B (head = C)
//                                 5. Push A back (recycle) -> head = A -> C
//                                    (A is now reused but A->next != B!)
// 6. CAS(head, A -> B) succeeds!
//    Because head IS A again - but A->next is now C, not B!
//    head = B (DANGLING!) - stack is corrupted
//
// The value went A -> B -> A. CAS sees A and thinks nothing changed.
// But the MEANING of A has changed - this is the "ABA" problem.

// === MITIGATION: Tagged pointer (version counter) ===
template<typename T>
class ABAFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T val) : data(std::move(val)), next(nullptr) {}
    };

    // Pack a pointer + version counter into a single atomic
    struct TaggedPtr {
        Node* ptr;
        uintptr_t tag; // version counter

        bool operator==(const TaggedPtr& o) const {
            return ptr == o.ptr && tag == o.tag;
        }
    };

    // NOTE: requires std::atomic<TaggedPtr> to be lock-free
    // On x86-64, this works with 128-bit CAS (CMPXCHG16B)
    std::atomic<TaggedPtr> head_{{nullptr, 0}};

public:
    void push(T value) {
        Node* new_node = new Node(std::move(value));
        TaggedPtr old_head = head_.load(std::memory_order_relaxed);

        TaggedPtr new_head;
        new_head.ptr = new_node;

        do {
            new_node->next = old_head.ptr;
            new_head.tag = old_head.tag + 1; // increment version!
        } while (!head_.compare_exchange_weak(
            old_head, new_head,
            std::memory_order_release,
            std::memory_order_relaxed));
    }

    std::optional<T> pop() {
        TaggedPtr old_head = head_.load(std::memory_order_relaxed);

        while (old_head.ptr) {
            TaggedPtr new_head{old_head.ptr->next, old_head.tag + 1};

            if (head_.compare_exchange_weak(
                old_head, new_head,
                std::memory_order_acquire,
                std::memory_order_relaxed)) {

                T value = std::move(old_head.ptr->data);
                // Don't delete immediately - use hazard pointers or
                // epoch-based reclamation for safe memory reclamation
                delete old_head.ptr; // simplified
                return value;
            }
        }
        return std::nullopt;
    }
};

int main() {
    // The tag (version counter) prevents ABA:
    // Even if the pointer returns to the same address A,
    // the tag has changed from 0 to 2, so CAS fails correctly.
    //
    // pop reads: {A, tag=0}
    // Meanwhile: pop A (tag->1), pop B (tag->2), push A back (tag->3)
    // CAS compares {A, 0} vs {A, 3} -> NOT EQUAL -> fails -> retry

    std::cout << "ABA-safe stack operational\n";

    // OTHER ABA MITIGATIONS:
    // 1. Hazard pointers (C++26: std::hazard_pointer)
    //    - Each thread publishes pointers it's reading
    //    - Deletions deferred until no hazard pointer references the node
    //
    // 2. Epoch-based reclamation (RCU-like)
    //    - Readers enter/exit epochs
    //    - Deleted nodes reclaimed when no reader is in the old epoch
    //
    // 3. Don't reuse node memory (allocate fresh, use GC)
    //    - Wasteful but eliminates ABA by design
}
```

The tagged-pointer approach adds a monotonically incrementing version counter alongside the pointer. Now CAS checks both the pointer value and the version. Even if the pointer comes back to the same address, the version has moved forward, so CAS correctly detects the intervening changes and retries.

---

## Notes

- Memory ordering: Push uses `release` (publishes the new node's contents to other threads), pop uses `acquire` (reads the node's data after observing the pointer). This ensures the node's contents are visible after a successful pop.
- Memory reclamation: The simplified `delete` in pop has a use-after-free risk if another thread is still reading the same node. Use hazard pointers, epoch-based reclamation, or `std::hazard_pointer` (C++26) for production code.
- 128-bit CAS: The tagged-pointer approach requires double-width CAS (`CMPXCHG16B` on x86-64). Compile with `-mcx16` on GCC/Clang.
- `compare_exchange_weak` loop pattern: The first parameter is in/out - on failure it is updated to the current value automatically. No manual reload is needed.
- Test with `-fsanitize=thread` and stress testing to verify lock-free correctness. These algorithms are very hard to get right by reasoning alone.
