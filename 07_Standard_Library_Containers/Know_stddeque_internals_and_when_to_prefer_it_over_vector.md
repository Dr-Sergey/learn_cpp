# Know std::deque internals and when to prefer it over vector

**Category:** Standard Library Containers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/container/deque>  

---

## Topic Overview

`std::deque` (double-ended queue) provides O(1) insertion and removal at **both** ends, unlike `std::vector` which only supports O(1) at the back. The trade-off is non-contiguous memory and worse cache performance. Understanding the internal structure explains both the performance characteristics and the iterator invalidation rules, which are subtly different from `vector`.

### Internal Structure

The key insight is that `std::deque` doesn't store elements in one big flat array. Instead it maintains a "map" - an array of pointers to fixed-size blocks. Elements live inside those blocks, and new blocks get attached to either end of the map as needed. When you add to the front, you're placing the element in a pre-existing block's spare space or allocating a new block - either way, no existing element moves.

```cpp
Map (array of pointers to fixed-size blocks):
  [Block*] [Block*] [Block*] [Block*] [Block*]
     |        |        |        |        |
     v        v        v        v        v
  [a][b]  [c][d]   [e][f]   [g][h]   [i][ ]
                                         ^
                                     end (spare space)
```

- **Block array (map):** A dynamically allocated array of pointers to fixed-size memory blocks.
- **Blocks (chunks):** Each block holds a fixed number of elements (typically 512 bytes / sizeof(T)).
- **Front/back growth:** New blocks are added to the beginning or end of the map.

### std::deque vs std::vector Comparison

If the table feels like a lot, the main takeaway is: `deque` gives you O(1) front operations that `vector` can never match, but you pay for it with non-contiguous memory and no `data()` or `reserve()`.

| Feature | `std::vector` | `std::deque` |
| --- | --- | --- |
| `push_back()` | Amortized O(1) | Amortized O(1) |
| `push_front()` | **O(n)** (shift all) | **O(1)** |
| `pop_front()` | **O(n)** (shift all) | **O(1)** |
| Random access `[i]` | O(1) | O(1) (slightly slower) |
| Memory layout | **Contiguous** | Non-contiguous (chunked) |
| Cache performance | **Excellent** | Good but worse than vector |
| Iterator invalidation | push_back may invalidate all | push_back/push_front invalidate all iterators |
| Pointer/ref invalidation | push_back may invalidate all | push_back/push_front do **NOT** invalidate refs/ptrs |
| `data()` pointer | Yes | **No** (not contiguous) |
| `reserve()` | Yes | **No** |
| `shrink_to_fit()` | Yes | Yes |

The pointer/reference stability row is particularly useful: if you take the address of a deque element, `push_back` and `push_front` won't invalidate it. With `vector`, any operation that triggers reallocation invalidates all pointers. This is why `std::deque` is the default backing container for `std::queue` and `std::stack`.

### How push_front Works in O(1)

1. If the first block has spare space at the beginning -> place element there.
2. If no space -> allocate a new block, add its pointer to the front of the map.
3. The map itself may need to be reallocated (like vector), but elements are never moved - only pointers are copied.

### Core Example

Here's the basic API. The comments about `data()` and `reserve()` not compiling are the most common surprises for people coming from `vector`.

```cpp
#include <iostream>
#include <deque>
#include <vector>
#include <chrono>

int main() {
    std::deque<int> dq;

    // O(1) push_front and push_back
    dq.push_front(1);   // [1]
    dq.push_front(0);   // [0, 1]
    dq.push_back(2);    // [0, 1, 2]
    dq.push_back(3);    // [0, 1, 2, 3]

    // Random access
    std::cout << "dq[0] = " << dq[0] << "\n";  // Output: dq[0] = 0
    std::cout << "dq[3] = " << dq[3] << "\n";  // Output: dq[3] = 3

    // O(1) pop from both ends
    dq.pop_front();  // [1, 2, 3]
    dq.pop_back();   // [1, 2]

    // Iteration
    for (int x : dq) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 2

    // No data() - deque memory is NOT contiguous
    // dq.data();  // ERROR: no member named 'data'

    // No reserve() - deque doesn't pre-allocate in blocks
    // dq.reserve(100);  // ERROR: no member named 'reserve'

    return 0;
}
```

### Important Notes

- **Prefer `std::vector`** in most cases - contiguous memory gives better cache behavior.
- Use `std::deque` when you need **efficient front insertion/removal** or when you need **pointer/reference stability** during push_back/push_front.
- `std::deque` cannot be passed to APIs expecting a contiguous buffer (`data()` doesn't exist).
- Iterators are invalidated by any insertion or erasure (even push_back), but **pointers and references to elements remain valid** for push_back/push_front (not for middle insertions).

---

## Self-Assessment

### Q1: Explain why deque has O(1) push_front but vector does not

The numbers here speak for themselves. Vector's push_front scales with O(n^2) because every single push_front shifts all existing elements. Deque's only ever adds a block pointer to the map, which stays tiny.

```cpp
#include <iostream>
#include <deque>
#include <vector>
#include <chrono>

void benchmark_push_front(int n) {
    // --- Vector push_front (O(n) each -> O(n^2) total) ---
    {
        std::vector<int> v;
        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < n; ++i) {
            v.insert(v.begin(), i);  // O(n) - shifts all elements
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "vector push_front x" << n << ": " << dur.count() << " us\n";
    }

    // --- Deque push_front (O(1) each -> O(n) total) ---
    {
        std::deque<int> dq;
        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < n; ++i) {
            dq.push_front(i);  // O(1) - adds to front of first block
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "deque  push_front x" << n << ": " << dur.count() << " us\n";
    }
}

int main() {
    benchmark_push_front(10'000);
    benchmark_push_front(50'000);
    // Expected output (approximate):
    // vector push_front x10000: ~15000 us (O(n^2))
    // deque  push_front x10000: ~100 us   (O(n))
    // vector push_front x50000: ~350000 us
    // deque  push_front x50000: ~500 us

    return 0;
}
```

**Explanation:**

- **`std::vector` push_front (via `insert(begin(), x)`):** Vector stores elements contiguously. To insert at position 0, **every existing element must be shifted right by one position** -> O(n) per insertion.
- **`std::deque` push_front:** Deque stores elements in fixed-size blocks with a map of pointers.
  1. If the first block has space at the front -> construct element there. O(1).
  2. If the first block is full -> allocate a new block, prepend its pointer to the map. O(1) amortized (map reallocation is rare and only copies pointers, not elements).
- **Key insight:** Deque never needs to move existing elements during push_front/push_back. Only pointers in the map may be relocated.

### Q2: Show why deque's memory is not contiguous and how this affects cache performance

The address printout makes the chunked layout concrete. Within a block, addresses are consecutive. Between blocks, there's a jump - and that jump costs a cache miss every time iteration crosses a block boundary.

```cpp
#include <iostream>
#include <deque>
#include <vector>
#include <chrono>
#include <numeric>

int main() {
    constexpr int N = 1'000'000;

    // --- Setup ---
    std::vector<int> vec(N);
    std::deque<int> deq(N);
    std::iota(vec.begin(), vec.end(), 0);
    std::iota(deq.begin(), deq.end(), 0);

    // --- Demonstrate non-contiguous memory ---
    std::cout << "Vector element addresses (first 5):\n";
    for (int i = 0; i < 5; ++i) {
        std::cout << "  [" << i << "] @ " << &vec[i]
                  << " (diff from [0]: " << (&vec[i] - &vec[0]) * sizeof(int) << " bytes)\n";
    }
    // Output: consecutive addresses, diff = 0, 4, 8, 12, 16

    std::cout << "\nDeque element addresses (first 5):\n";
    for (int i = 0; i < 5; ++i) {
        std::cout << "  [" << i << "] @ " << &deq[i]
                  << " (diff from [0]: "
                  << reinterpret_cast<const char*>(&deq[i]) -
                     reinterpret_cast<const char*>(&deq[0])
                  << " bytes)\n";
    }
    // Elements within same block are contiguous, but blocks may be far apart

    // --- Cache performance benchmark: sequential sum ---
    auto time_sum = [](auto& container, const char* name) {
        volatile long long sum = 0;
        auto start = std::chrono::high_resolution_clock::now();
        for (int rep = 0; rep < 100; ++rep) {
            sum = 0;
            for (const auto& x : container) {
                sum += x;
            }
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << name << " sum (100 reps): " << dur.count() << " us\n";
    };

    std::cout << "\nCache performance:\n";
    time_sum(vec, "vector");
    time_sum(deq, "deque ");
    // Expected: vector is 1.5-3x faster due to contiguous memory -> better prefetching

    return 0;
}
```

**How it works:**

- **Vector:** All elements live in one contiguous memory block. CPU prefetchers predict sequential access and load cache lines ahead of time. Minimal cache misses.
- **Deque:** Elements are split across multiple blocks (chunks). When iteration crosses a block boundary, the CPU must fetch from a potentially distant memory location -> cache miss.
- **Within a block**, deque elements are contiguous (good cache behavior). But **between blocks**, there's a pointer indirection through the map.
- Typical sequential iteration speedup for vector over deque: 1.5x to 3x depending on element size and hardware.
- **Rule of thumb:** If you iterate frequently, prefer `vector`. If you need O(1) push_front, use `deque`.

### Q3: Demonstrate a use case where deque is better than vector+reverse for a FIFO queue

The sliding window example is the clearest demonstration of why `deque` exists. You need O(1) at both ends, and no standard data structure gives you that except `deque`. A vector-based FIFO is O(n^2) because every `erase(begin())` shifts the whole array.

```cpp
#include <iostream>
#include <deque>
#include <vector>
#include <queue>
#include <chrono>
#include <string>

// FIFO queue: elements enter at back, leave from front

// Approach 1: std::queue (uses std::deque internally by default)
void fifo_with_queue() {
    std::queue<std::string> q;  // Backed by std::deque<std::string>

    q.push("Task A");
    q.push("Task B");
    q.push("Task C");

    std::cout << "Queue-based FIFO:\n";
    while (!q.empty()) {
        std::cout << "  Processing: " << q.front() << "\n";
        q.pop();  // O(1) - deque's pop_front
    }
    // Output:
    //   Processing: Task A
    //   Processing: Task B
    //   Processing: Task C
}

// Approach 2: Raw deque as a sliding window
void sliding_window_deque() {
    std::deque<int> window;
    constexpr int WINDOW_SIZE = 3;

    std::cout << "\nSliding window (size " << WINDOW_SIZE << "):\n";
    for (int val : {10, 20, 30, 40, 50, 60}) {
        window.push_back(val);               // O(1)
        if (window.size() > WINDOW_SIZE) {
            window.pop_front();               // O(1) - impossible with vector!
        }
        std::cout << "  Window: [";
        for (size_t i = 0; i < window.size(); ++i) {
            if (i > 0) std::cout << ", ";
            std::cout << window[i];
        }
        std::cout << "]\n";
    }
    // Output:
    //   Window: [10]
    //   Window: [10, 20]
    //   Window: [10, 20, 30]
    //   Window: [20, 30, 40]
    //   Window: [30, 40, 50]
    //   Window: [40, 50, 60]
}

// Approach 3: Performance comparison - deque vs vector for FIFO
void benchmark_fifo(int n) {
    // Vector FIFO: push_back + erase(begin()) - O(n) per dequeue
    {
        std::vector<int> v;
        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < n; ++i) v.push_back(i);
        for (int i = 0; i < n; ++i) v.erase(v.begin());  // O(n) each!
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "\nvector FIFO x" << n << ": " << dur.count() << " us\n";
    }

    // Deque FIFO: push_back + pop_front - O(1) per dequeue
    {
        std::deque<int> dq;
        auto start = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < n; ++i) dq.push_back(i);
        for (int i = 0; i < n; ++i) dq.pop_front();  // O(1) each!
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "deque  FIFO x" << n << ": " << dur.count() << " us\n";
    }
}

int main() {
    fifo_with_queue();
    sliding_window_deque();
    benchmark_fifo(100'000);
    // Expected:
    //   vector FIFO x100000: ~1,000,000+ us (O(n^2))
    //   deque  FIFO x100000: ~1,000 us      (O(n))

    return 0;
}
```

**How it works:**

- **FIFO pattern:** Elements enter at the back (`push_back`) and exit from the front (`pop_front`).
- **Vector FIFO:** `erase(begin())` shifts every remaining element left -> O(n) per dequeue -> O(n^2) total.
- **Deque FIFO:** `pop_front()` removes the first element in O(1) - no elements are moved. Only the internal pointer to the first element advances.
- **`std::queue`** uses `std::deque` as its default backing container precisely because of this O(1) push_back + pop_front performance.
- **Sliding window:** A classic use case for deque - maintain a fixed-size window over a stream by pushing to the back and popping from the front, both in O(1).

---

## Notes

- **When to use `std::deque`:**
  - FIFO queues / sliding windows (O(1) push_back + pop_front)
  - When you need pointer/reference stability during push_back/push_front
  - When elements are very large (deque doesn't need to reallocate/copy all elements)
- **When NOT to use `std::deque`:**
  - Sequential iteration performance is critical (use vector)
  - Need contiguous memory for C APIs or SIMD (use vector)
  - Need `reserve()` (deque doesn't support it)
- **`std::deque` as default for `std::stack` and `std::queue`:** Both container adaptors use deque by default. `std::stack` uses `push_back`/`pop_back`, `std::queue` uses `push_back`/`pop_front`.
- **Memory overhead:** Deque uses more memory per element due to the block map + partially filled blocks.
- **C++20 `std::erase`/`std::erase_if`** work with deque: `std::erase(dq, value);`
