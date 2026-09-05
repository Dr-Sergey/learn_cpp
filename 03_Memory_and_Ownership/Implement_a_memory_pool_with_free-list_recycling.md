# Implement a Memory Pool with Free-List Recycling

**Category:** Memory and Ownership  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/memory>  

---

## Topic Overview

### What Is a Memory Pool

A memory pool (or object pool) pre-allocates a fixed block of memory and carves it into fixed-size slots. When you "allocate" an object, you get a slot from the pool. When you "deallocate," the slot returns to a **free list** for reuse. The key benefit is that you pay the cost of talking to the OS allocator once at startup, and every individual alloc/free after that is just a pointer manipulation.

### Why Use a Pool

| Feature | `new`/`delete` | Memory Pool |
| --- | --- | --- |
| Allocation speed | Varies (heap walk) | **O(1)** - pop from free list |
| Deallocation speed | Varies | **O(1)** - push to free list |
| Fragmentation | Can fragment heap | **None** - fixed-size slots |
| Cache locality | Scattered | **Excellent** - contiguous block |
| Thread overhead | Global lock (often) | Can be per-thread |

### Free-List Recycling

The key idea: when a slot is free, repurpose its own memory to store a pointer to the next free slot. This is an **intrusive linked list** - no extra memory needed. The same bytes that will eventually hold your object currently hold a pointer. Here is what it looks like as slots are allocated and freed:

```cpp
Pool memory: [slot0][slot1][slot2][slot3][slot4]

Free list (all free):
  free_head -> slot0 -> slot1 -> slot2 -> slot3 -> slot4 -> nullptr

After allocating slot0 and slot1:
  free_head -> slot2 -> slot3 -> slot4 -> nullptr

After freeing slot0:
  free_head -> slot0 -> slot2 -> slot3 -> slot4 -> nullptr
```

Allocation is just "pop the head pointer and return it." Deallocation is just "push this pointer back to the head." Both are O(1) with no heap interaction.

### Alignment Requirement

Each slot must be at least `sizeof(void*)` bytes and aligned for `void*`, since we store a pointer in freed slots. For the actual objects, we also need proper alignment: `alignof(T)`. The slot size and alignment must satisfy both constraints simultaneously.

---

## Self-Assessment

### Q1: Build a typed `MemoryPool<T,N>` that pre-allocates N objects and recycles freed slots via an intrusive list

This is the full implementation. Pay attention to how `allocate` pops the free-list head and how `deallocate` pushes back onto it - those two operations are the entire mechanism.

```cpp
#include <iostream>
#include <cstddef>
#include <cassert>
#include <new>
#include <array>
#include <vector>

template<typename T, size_t N>
class MemoryPool {
    // Each slot is at least large enough for T or a pointer
    static constexpr size_t SLOT_SIZE = sizeof(T) > sizeof(void*)
                                        ? sizeof(T) : sizeof(void*);

    // Aligned storage for N slots
    alignas(alignof(T)) std::byte storage_[N * SLOT_SIZE];

    // Free list head - points to first free slot
    void* free_head_ = nullptr;
    size_t allocated_ = 0;

    // Get pointer to slot i
    void* slot(size_t i) {
        return &storage_[i * SLOT_SIZE];
    }

public:
    MemoryPool() {
        // Build the free list: each slot points to the next
        for (size_t i = 0; i < N; ++i) {
            void* s = slot(i);
            // Store pointer to next free slot inside this slot
            void* next = (i + 1 < N) ? slot(i + 1) : nullptr;
            *static_cast<void**>(s) = next;
        }
        free_head_ = slot(0);
    }

    // Allocate one T (constructs in-place)
    template<typename... Args>
    T* allocate(Args&&... args) {
        if (!free_head_) {
            throw std::bad_alloc();
        }

        // Pop from free list
        void* ptr = free_head_;
        free_head_ = *static_cast<void**>(free_head_);
        ++allocated_;

        // Construct T in the slot
        return ::new (ptr) T(std::forward<Args>(args)...);
    }

    // Deallocate one T (destroys and returns to free list)
    void deallocate(T* obj) {
        assert(obj != nullptr);
        // Destroy the object
        obj->~T();

        // Push slot back onto free list
        void* ptr = static_cast<void*>(obj);
        *static_cast<void**>(ptr) = free_head_;
        free_head_ = ptr;
        --allocated_;
    }

    size_t capacity() const { return N; }
    size_t allocated() const { return allocated_; }
    size_t available() const { return N - allocated_; }
};

struct Widget {
    int id;
    double value;
    Widget(int i, double v) : id(i), value(v) {
        std::cout << "  Widget(" << id << ", " << value << ") constructed\n";
    }
    ~Widget() {
        std::cout << "  Widget(" << id << ") destroyed\n";
    }
};

int main() {
    MemoryPool<Widget, 4> pool;

    std::cout << "Pool capacity: " << pool.capacity() << "\n";
    std::cout << "Available: " << pool.available() << "\n\n";

    // Allocate some widgets
    Widget* w1 = pool.allocate(1, 3.14);
    Widget* w2 = pool.allocate(2, 2.72);
    Widget* w3 = pool.allocate(3, 1.41);

    std::cout << "\nAllocated: " << pool.allocated()
              << ", Available: " << pool.available() << "\n\n";

    // Free w2 - slot goes back to free list
    pool.deallocate(w2);
    std::cout << "\nAfter freeing w2 - Available: " << pool.available() << "\n\n";

    // Allocate a new widget - reuses w2's slot
    Widget* w4 = pool.allocate(4, 9.99);
    std::cout << "\nw4 reused slot: " << (w4 == w2 ? "yes" : "no") << "\n";

    // Cleanup
    pool.deallocate(w1);
    pool.deallocate(w3);
    pool.deallocate(w4);

    std::cout << "\nFinal - Available: " << pool.available() << "\n";

    return 0;
}
```

**Output:**

```text
Pool capacity: 4
Available: 4

  Widget(1, 3.14) constructed
  Widget(2, 2.72) constructed
  Widget(3, 1.41) constructed

Allocated: 3, Available: 1

  Widget(2) destroyed

After freeing w2 - Available: 2

  Widget(4, 9.99) constructed

w4 reused slot: yes
  Widget(1) destroyed
  Widget(3) destroyed
  Widget(4) destroyed

Final - Available: 4
```

Notice the last line: `w4 reused slot: yes`. The freed slot from w2 was immediately handed back to the next allocation. No heap involvement, no fragmentation.

### Q2: Show that pool allocation has O(1) amortized alloc/free with no fragmentation

Here we run a microbenchmark comparing the pool against `new`/`delete` for 100,000 operations. We also check the slot spacing to confirm there is no fragmentation.

```cpp
#include <iostream>
#include <chrono>
#include <vector>
#include <cstddef>
#include <new>

// Simple pool for benchmarking
template<typename T, size_t N>
class FastPool {
    alignas(alignof(T)) std::byte storage_[N * sizeof(T)];
    void* free_head_ = nullptr;
public:
    FastPool() {
        for (size_t i = 0; i < N; ++i) {
            void* s = &storage_[i * sizeof(T)];
            *static_cast<void**>(s) = (i + 1 < N) ? &storage_[(i+1) * sizeof(T)] : nullptr;
        }
        free_head_ = &storage_[0];
    }

    void* alloc() {
        // O(1): just pop the head
        void* ptr = free_head_;
        if (ptr) free_head_ = *static_cast<void**>(ptr);
        return ptr;
    }

    void dealloc(void* ptr) {
        // O(1): just push to head
        *static_cast<void**>(ptr) = free_head_;
        free_head_ = ptr;
    }
};

struct Small { int data[4]; };  // 16 bytes

int main() {
    constexpr size_t COUNT = 100'000;
    constexpr size_t POOL_SIZE = COUNT;

    // Benchmark pool allocation
    FastPool<Small, POOL_SIZE> pool;
    std::vector<void*> ptrs(COUNT);

    auto t1 = std::chrono::high_resolution_clock::now();
    for (size_t i = 0; i < COUNT; ++i) {
        ptrs[i] = pool.alloc();
    }
    for (size_t i = 0; i < COUNT; ++i) {
        pool.dealloc(ptrs[i]);
    }
    auto t2 = std::chrono::high_resolution_clock::now();
    auto pool_ns = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    // Benchmark new/delete
    t1 = std::chrono::high_resolution_clock::now();
    for (size_t i = 0; i < COUNT; ++i) {
        ptrs[i] = new Small;
    }
    for (size_t i = 0; i < COUNT; ++i) {
        delete static_cast<Small*>(ptrs[i]);
    }
    t2 = std::chrono::high_resolution_clock::now();
    auto heap_ns = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    std::cout << "=== " << COUNT << " alloc+free operations ===\n";
    std::cout << "Pool:       " << pool_ns << " us\n";
    std::cout << "new/delete: " << heap_ns << " us\n";
    std::cout << "Speedup:    ~" << (heap_ns > 0 ? heap_ns / (pool_ns > 0 ? pool_ns : 1) : 0) << "x\n";

    // No fragmentation proof: all slots are contiguous
    void* first = pool.alloc();
    void* second = pool.alloc();
    ptrdiff_t gap = static_cast<std::byte*>(second) - static_cast<std::byte*>(first);
    std::cout << "\nSlot spacing: " << gap << " bytes (== sizeof(Small)="
              << sizeof(Small) << "? " << (gap == sizeof(Small) ? "YES" : "varies") << ")\n";
    pool.dealloc(second);
    pool.dealloc(first);

    return 0;
}
```

**Output (typical):**

```text
=== 100000 alloc+free operations ===
Pool:       245 us
new/delete: 3821 us
Speedup:    ~15x

Slot spacing: 16 bytes (== sizeof(Small)=16? YES)
```

The slot spacing being exactly `sizeof(Small)` confirms there is no fragmentation - all objects sit in a perfectly packed contiguous block.

### Q3: Explain the alignment requirements for storing a free-list pointer inside freed object storage

The intrusive free list stores a `void*` pointer inside each free slot. This imposes two constraints:

| Requirement | Why |
| --- | --- |
| `sizeof(slot) >= sizeof(void*)` | The pointer must fit in the slot |
| `alignof(slot) >= alignof(void*)` | The pointer must be properly aligned |

The slot dimensions must satisfy both the object's requirements and the pointer's requirements. For tiny types like `char`, the slot has to be enlarged just to hold the free-list pointer.

```cpp
#include <iostream>
#include <cstddef>
#include <type_traits>

template<typename T>
void check_pool_compatibility() {
    constexpr size_t obj_size = sizeof(T);
    constexpr size_t obj_align = alignof(T);
    constexpr size_t ptr_size = sizeof(void*);
    constexpr size_t ptr_align = alignof(void*);

    // The slot must fit a T AND a void*
    constexpr size_t slot_size = obj_size > ptr_size ? obj_size : ptr_size;
    // The slot must satisfy alignment of T AND void*
    constexpr size_t slot_align = obj_align > ptr_align ? obj_align : ptr_align;

    std::cout << "Type requirements:\n";
    std::cout << "  sizeof(T)  = " << obj_size << ", alignof(T)  = " << obj_align << "\n";
    std::cout << "  sizeof(void*) = " << ptr_size << ", alignof(void*) = " << ptr_align << "\n";
    std::cout << "  -> slot_size = " << slot_size << ", slot_align = " << slot_align << "\n";

    // If T is smaller than void*, we waste some bytes per slot
    if (obj_size < ptr_size) {
        std::cout << "  WARNING: " << (ptr_size - obj_size)
                  << " bytes wasted per slot (T smaller than pointer)\n";
    }
}

// Test types
struct Tiny { char c; };           // 1 byte - smaller than void*!
struct Normal { int x; double y; };  // 16 bytes
struct alignas(64) CacheLine { int data[16]; };  // 64-byte aligned

int main() {
    std::cout << "=== Tiny (1 byte) ===\n";
    check_pool_compatibility<Tiny>();

    std::cout << "\n=== Normal (16 bytes) ===\n";
    check_pool_compatibility<Normal>();

    std::cout << "\n=== CacheLine (64-byte aligned) ===\n";
    check_pool_compatibility<CacheLine>();

    // Key insight: for types smaller than sizeof(void*),
    // the pool slot must be enlarged. The formula is:
    // slot_size = max(sizeof(T), sizeof(void*))
    // slot_align = max(alignof(T), alignof(void*))

    return 0;
}
```

**Output (64-bit system):**

```text
=== Tiny (1 byte) ===
Type requirements:
  sizeof(T)  = 1, alignof(T)  = 1
  sizeof(void*) = 8, alignof(void*) = 8
  -> slot_size = 8, slot_align = 8
  WARNING: 7 bytes wasted per slot (T smaller than pointer)

=== Normal (16 bytes) ===
Type requirements:
  sizeof(T)  = 16, alignof(T)  = 8
  sizeof(void*) = 8, alignof(void*) = 8
  -> slot_size = 16, slot_align = 8

=== CacheLine (64-byte aligned) ===
Type requirements:
  sizeof(T)  = 64, alignof(T)  = 64
  sizeof(void*) = 8, alignof(void*) = 8
  -> slot_size = 64, slot_align = 64
```

For `Tiny`, a 1-byte type wastes 7 bytes per slot just to accommodate the free-list pointer. For types at or above pointer size, there is no overhead.

---

## Notes

- Memory pools shine for **same-size, high-frequency allocations** (game entities, network packets, AST nodes).
- For production use, consider `std::pmr::monotonic_buffer_resource` (no recycling) or `std::pmr::unsynchronized_pool_resource` (pool with recycling) - they implement these patterns and integrate with standard containers.
- The intrusive free list technique means a freed slot's data is overwritten by the next pointer. Never access an object after returning it to the pool.
- For multithreaded pools, each thread can have its own pool (thread-local) or use lock-free free lists.
