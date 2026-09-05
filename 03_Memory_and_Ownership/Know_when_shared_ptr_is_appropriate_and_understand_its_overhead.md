# Know When `shared_ptr` Is Appropriate and Understand Its Overhead

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/shared_ptr>  

---

## Topic Overview

### When to Use `shared_ptr`

`shared_ptr` provides **shared ownership**: multiple owners keep the object alive, and the last one to release it triggers destruction. The key word is "last" - the object's lifetime is not tied to any single scope. If you know exactly who should own an object, `unique_ptr` is almost always the better choice. Reach for `shared_ptr` when you genuinely cannot say who the last owner will be.

| Use `shared_ptr` when... | Use `unique_ptr` instead when... |
| --- | --- |
| Multiple owners have independent lifetimes | There's a single clear owner |
| Object lifetime isn't tied to one scope | Object lifetime matches one scope |
| Observer pattern (with `weak_ptr`) | Simple ownership transfer |
| Caches, registries, shared configs | Most cases - `unique_ptr` is the default |

### The Overhead of `shared_ptr`

`shared_ptr` is not free. It pays for its flexibility with real runtime costs, and it is worth knowing exactly what they are before you decide to use it everywhere.

| Cost | Details |
| --- | --- |
| **Size** | 2 pointers (object ptr + control block ptr) vs 1 for `unique_ptr` |
| **Control block** | Heap allocation for ref count, weak count, deleter, allocator |
| **Atomic ref counting** | Every copy/destruction does `atomic_increment`/`atomic_decrement` |
| **Cache misses** | Object and control block are at different addresses (unless `make_shared`) |

### `make_shared` Optimization

`make_shared` collapses the two allocations into one, which improves cache locality and halves the allocator calls. Here is the layout difference:

```cpp
new T + shared_ptr(p):          make_shared<T>():
┌────────────┐                  ┌──────────────────────┐
│ Control Block │ <- alloc 1    │ Control Block + T     │ <- single alloc
├────────────┤                  │  ref_count: 1         │
│ ref_count: 1 │               │  weak_count: 1        │
│ weak_count: 1│               │  T object data        │
└────────────┘                  └──────────────────────┘
┌────────────┐ <- alloc 2
│ T object    │
└────────────┘

make_shared: 1 allocation, better cache locality
```

---

## Self-Assessment

### Q1: Demonstrate the reference counting overhead of `shared_ptr` vs `unique_ptr` in a microbenchmark

Each copy of a `shared_ptr` does an atomic increment. Each destruction does an atomic decrement and a comparison. On modern CPUs, atomic operations on a contended cache line can be significantly slower than a plain pointer copy. This benchmark makes that cost visible.

```cpp
#include <iostream>
#include <memory>
#include <chrono>
#include <vector>

struct Widget { int data[4]; };

template<typename Func>
long long benchmark(Func f, int iterations) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) f();
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
}

int main() {
    constexpr int N = 1'000'000;

    // Benchmark: create + copy + destroy shared_ptr
    auto shared_time = benchmark([]{
        auto sp = std::make_shared<Widget>();
        auto sp2 = sp;    // atomic increment
        auto sp3 = sp;    // atomic increment
        // 3 atomic decrements on destruction
    }, N);

    // Benchmark: create + move unique_ptr
    auto unique_time = benchmark([]{
        auto up = std::make_unique<Widget>();
        auto up2 = std::move(up);    // just pointer copy
        auto up3 = std::move(up2);   // just pointer copy
        // single delete on destruction
    }, N);

    std::cout << "=== " << N << " iterations ===\n";
    std::cout << "shared_ptr (copy): " << shared_time << " us\n";
    std::cout << "unique_ptr (move): " << unique_time << " us\n";
    std::cout << "Ratio: " << (double)shared_time / unique_time << "x\n";

    // Size comparison
    std::cout << "\n=== Size ===\n";
    std::cout << "sizeof(shared_ptr<Widget>): " << sizeof(std::shared_ptr<Widget>) << " bytes\n";
    std::cout << "sizeof(unique_ptr<Widget>): " << sizeof(std::unique_ptr<Widget>) << " bytes\n";
    std::cout << "sizeof(Widget*):            " << sizeof(Widget*) << " bytes\n";

    return 0;
}
```

**Typical output (64-bit system):**

```text
=== 1000000 iterations ===
shared_ptr (copy): 48000 us
unique_ptr (move): 12000 us
Ratio: 4x

=== Size ===
sizeof(shared_ptr<Widget>): 16 bytes
sizeof(unique_ptr<Widget>): 8 bytes
sizeof(Widget*):            8 bytes
```

The 4x slowdown and double size are not bugs - they are the price of shared ownership. In most code this is fine. In a hot loop creating millions of smart pointers, it is not.

### Q2: Explain why `shared_ptr<T>` stores a control block separately and what `make_shared<T>` optimizes

The control block exists because `shared_ptr` needs to store metadata (counts, deleter, allocator) that must outlive the `T` object itself - specifically, it must survive as long as any `weak_ptr` holds a reference. That constraint forces the control block to be a separate allocation.

```cpp
#include <iostream>
#include <memory>

struct Large {
    char data[1024];
};

int main() {
    // Method 1: Two allocations
    // Allocation 1: new Large
    // Allocation 2: control block (inside shared_ptr constructor)
    {
        Large* raw = new Large;
        std::shared_ptr<Large> sp(raw);
        // Control block and object are at different addresses
        std::cout << "=== shared_ptr(new T) - two allocations ===\n";
        std::cout << "Object address:  " << sp.get() << "\n";
        std::cout << "shared_ptr size: " << sizeof(sp) << " bytes\n";
        std::cout << "use_count: " << sp.use_count() << "\n";
    }

    // Method 2: Single allocation (make_shared)
    // Control block and object are allocated together
    {
        auto sp = std::make_shared<Large>();
        std::cout << "\n=== make_shared<T>() - single allocation ===\n";
        std::cout << "Object address:  " << sp.get() << "\n";
        std::cout << "use_count: " << sp.use_count() << "\n";
    }

    // Why separate control block?
    // 1. shared_ptr can manage objects not allocated with new (custom deleters)
    // 2. Control block can outlive the object (weak_ptr keeps it alive)
    // 3. Multiple shared_ptrs from different raw pointers would need separate counts

    std::cout << "\n=== Control block contents ===\n";
    std::cout << "  - Strong reference count (atomic)\n";
    std::cout << "  - Weak reference count (atomic)\n";
    std::cout << "  - Deleter (type-erased)\n";
    std::cout << "  - Allocator (type-erased)\n";

    // make_shared advantage: object + control block in one allocation
    std::cout << "\n=== make_shared advantages ===\n";
    std::cout << "  1. Single heap allocation (faster)\n";
    std::cout << "  2. Better cache locality\n";
    std::cout << "  3. Exception-safe (no leak if ctor throws)\n";

    // make_shared disadvantage:
    std::cout << "\n=== make_shared disadvantage ===\n";
    std::cout << "  Object memory not freed until ALL weak_ptrs expire\n";
    std::cout << "  (control block and object share the allocation)\n";

    // Demonstration of the weak_ptr issue
    {
        std::weak_ptr<Large> wp;
        {
            auto sp = std::make_shared<Large>();
            wp = sp;
            std::cout << "\nInside scope: use_count=" << sp.use_count() << "\n";
        }
        // sp destroyed, but memory not freed because wp holds the control block
        // which is in the same allocation as the Large object
        std::cout << "After scope: expired=" << wp.expired()
                  << ", but allocation may still be held\n";
    }

    return 0;
}
```

The `make_shared` disadvantage is subtle but real for large objects: since the object and control block share one allocation, the 1024-byte `Large` cannot be freed until the very last `weak_ptr` expires. With a separate allocation, the `Large` is freed as soon as the strong count hits zero, even if `weak_ptr`s still exist.

### Q3: Show a cyclic ownership bug with two `shared_ptr`s and fix it with `weak_ptr`

This is the most common real-world `shared_ptr` bug. Two objects own each other through `shared_ptr`, so neither reference count ever reaches zero. The memory leaks silently.

```cpp
#include <iostream>
#include <memory>
#include <string>

// === THE BUG: Cyclic shared_ptr -> memory leak ===

struct NodeBad {
    std::string name;
    std::shared_ptr<NodeBad> partner;  // Strong reference -> cycle!

    NodeBad(std::string n) : name(std::move(n)) {
        std::cout << "  " << name << " constructed\n";
    }
    ~NodeBad() {
        std::cout << "  " << name << " destroyed\n";
    }
};

void demonstrate_leak() {
    std::cout << "=== Cyclic shared_ptr (LEAKS!) ===\n";
    {
        auto alice = std::make_shared<NodeBad>("Alice");
        auto bob   = std::make_shared<NodeBad>("Bob");

        alice->partner = bob;    // alice -> bob (ref count: bob=2)
        bob->partner = alice;    // bob -> alice (ref count: alice=2)

        std::cout << "  alice use_count: " << alice.use_count() << "\n";
        std::cout << "  bob use_count:   " << bob.use_count() << "\n";
    }
    // alice goes out of scope: ref count drops to 1 (bob still holds it)
    // bob goes out of scope: ref count drops to 1 (alice still holds it)
    // Neither reaches 0 -> MEMORY LEAK!
    std::cout << "  (No destructors called - leaked!)\n\n";
}

// === THE FIX: Use weak_ptr to break the cycle ===

struct NodeGood {
    std::string name;
    std::weak_ptr<NodeGood> partner;  // Weak reference -> no cycle!

    NodeGood(std::string n) : name(std::move(n)) {
        std::cout << "  " << name << " constructed\n";
    }
    ~NodeGood() {
        std::cout << "  " << name << " destroyed\n";
    }

    void greet() const {
        if (auto p = partner.lock()) {  // Safely check if partner alive
            std::cout << "  " << name << "'s partner is " << p->name << "\n";
        } else {
            std::cout << "  " << name << "'s partner is gone\n";
        }
    }
};

void demonstrate_fix() {
    std::cout << "=== weak_ptr breaks the cycle (CORRECT) ===\n";
    {
        auto alice = std::make_shared<NodeGood>("Alice");
        auto bob   = std::make_shared<NodeGood>("Bob");

        alice->partner = bob;    // weak_ptr: doesn't affect bob's ref count
        bob->partner = alice;

        std::cout << "  alice use_count: " << alice.use_count() << "\n";  // 1
        std::cout << "  bob use_count:   " << bob.use_count() << "\n";    // 1

        alice->greet();
        bob->greet();
    }
    // alice goes out of scope: ref count 1->0 -> destroyed
    // bob goes out of scope: ref count 1->0 -> destroyed
    std::cout << "  (Both properly destroyed!)\n";
}

int main() {
    demonstrate_leak();
    demonstrate_fix();
    return 0;
}
```

**Output:**

```text
=== Cyclic shared_ptr (LEAKS!) ===
  Alice constructed
  Bob constructed
  alice use_count: 2
  bob use_count:   2
  (No destructors called - leaked!)

=== weak_ptr breaks the cycle (CORRECT) ===
  Alice constructed
  Bob constructed
  alice use_count: 1
  bob use_count:   1
  Alice's partner is Bob
  Bob's partner is Alice
  Bob destroyed
  Alice destroyed
  (Both properly destroyed!)
```

The `weak_ptr` version keeps the use counts at 1, so both objects are destroyed when the local variables go out of scope. The rule of thumb: in any bidirectional relationship, one direction should be a `weak_ptr`.

---

## Notes

- **Default to `unique_ptr`** - only use `shared_ptr` when ownership is genuinely shared.
- Always use `make_shared` to avoid double allocation and ensure exception safety.
- `weak_ptr::lock()` returns `shared_ptr` - always check for `nullptr` (object may be dead).
- Atomic ref counting is ~4x slower than non-atomic; in single-threaded hot paths, consider alternatives.
- `shared_ptr` aliasing constructor allows a `shared_ptr<Member>` that shares ownership with the parent.
