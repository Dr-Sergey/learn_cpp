# Know When to Use Intrusive Reference Counting vs `std::shared_ptr`

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://www.boost.org/doc/libs/release/libs/smart_ptr/doc/html/smart_ptr.html#intrusive_ptr>  

---

## Topic Overview

### The Two-Allocation Problem of `shared_ptr`

When you create a `shared_ptr` from a raw pointer, **two heap allocations** occur: one for the object and one for the control block. `make_shared` merges them into one allocation, but the control block is still a distinct structure sitting alongside the object.

```cpp
shared_ptr<T> sp(new T);

Allocation 1: T object          Allocation 2: Control block
┌──────────┐                    ┌─────────────────┐
│ T data   │                    │ strong_count: 1  │
└──────────┘                    │ weak_count: 1    │
                                │ deleter          │
                                │ allocator        │
                                └─────────────────┘
```

(`make_shared` merges these into one allocation, but the control block is still separate from T's viewpoint.)

### Intrusive Reference Counting

With intrusive counting, the **reference count lives inside the object itself**. There is no separate allocation, no second pointer, and no extra indirection.

```cpp
┌──────────────────┐
│ ref_count: 1     │  <- Part of the object
│ T data           │
└──────────────────┘
```

**No separate control block, no extra allocation.**

### Comparison

| Feature | `shared_ptr` | Intrusive Ref Count |
| --- | --- | --- |
| Allocations | 2 (or 1 with `make_shared`) | **1** (always) |
| Overhead per pointer | 2 pointers (16 bytes) | **1 pointer** (8 bytes) |
| Control block | Separate heap object | **Embedded in object** |
| Can create from raw `T*` | Yes (but dangerous - double ownership) | **Yes, safely** - ref count is in the object |
| Works with any `T`? | Yes | **No** - T must embed the count |
| Standard library? | Yes (`std::shared_ptr`) | No (Boost `intrusive_ptr`) |
| Custom deleter? | Yes (type-erased) | Must implement manually |

### When to Prefer Intrusive Counting

1. **Performance-critical paths** - one fewer allocation, one fewer cache miss
2. **Interop with C APIs** that pass raw pointers - you can safely recreate the smart pointer
3. **COM objects** / **OS reference-counted objects** - they already have embedded ref counts
4. **Memory-constrained systems** - less overhead per pointer

---

## Self-Assessment

### Q1: Explain the two-allocation problem of `shared_ptr` and how intrusive counts solve it

This example makes the memory layout differences concrete and shows why intrusive counting is safer when you need to reconstruct a smart pointer from a raw pointer.

```cpp
#include <iostream>
#include <memory>
#include <cstddef>

struct Widget {
    int data[4];
};

int main() {
    // === Two-allocation problem ===
    std::cout << "=== shared_ptr from raw pointer: 2 allocations ===\n";
    {
        // Allocation 1: new Widget
        Widget* raw = new Widget{{1, 2, 3, 4}};
        // Allocation 2: control block (inside shared_ptr ctor)
        std::shared_ptr<Widget> sp(raw);

        std::cout << "Object at:      " << sp.get() << "\n";
        std::cout << "shared_ptr size: " << sizeof(sp) << " bytes (2 pointers)\n";
        std::cout << "use_count: " << sp.use_count() << "\n";
    }

    // === make_shared: 1 allocation, but control block still exists ===
    std::cout << "\n=== make_shared: 1 allocation ===\n";
    {
        auto sp = std::make_shared<Widget>(Widget{{5, 6, 7, 8}});
        std::cout << "Object at:      " << sp.get() << "\n";
        std::cout << "shared_ptr size: " << sizeof(sp) << " bytes (still 2 pointers)\n";
        // Control block and Widget are co-allocated but the shared_ptr
        // still stores 2 pointers: one to object, one to control block
    }

    // === Intrusive solution: 0 extra allocations ===
    std::cout << "\n=== Intrusive: ref count inside the object ===\n";
    std::cout << "  - Ref count is a member of the object\n";
    std::cout << "  - Smart pointer only stores 1 raw pointer\n";
    std::cout << "  - No separate control block allocation\n";
    std::cout << "  - Can safely create smart ptr from raw T* at any time\n";

    return 0;
}
```

The critical difference is the "can safely create smart ptr from raw T* at any time" point. With `shared_ptr`, wrapping a raw pointer in a second `shared_ptr` creates a second independent control block, which causes a double-free. With intrusive counting, the count is in the object, so wrapping the same raw pointer twice is always safe.

### Q2: Implement a simple `intrusive_ptr` using an atomic ref count embedded in the object

This is a full, working implementation of intrusive reference counting. The `RefCounted` base class provides the embedded counter, and `intrusive_ptr` manages the add/release calls automatically.

```cpp
#include <iostream>
#include <atomic>
#include <utility>

// Base class that provides the embedded reference count
class RefCounted {
    mutable std::atomic<int> ref_count_{0};

    template<typename T> friend class intrusive_ptr;
    template<typename T> friend void intrusive_ptr_add_ref(const T*);
    template<typename T> friend void intrusive_ptr_release(const T*);

public:
    RefCounted() = default;
    virtual ~RefCounted() = default;

    int use_count() const { return ref_count_.load(std::memory_order_relaxed); }

    // Non-copyable ref count (copies start at 0)
    RefCounted(const RefCounted&) : ref_count_(0) {}
    RefCounted& operator=(const RefCounted&) { return *this; }
};

// Free functions for ref count management
template<typename T>
void intrusive_ptr_add_ref(const T* p) {
    p->ref_count_.fetch_add(1, std::memory_order_relaxed);
}

template<typename T>
void intrusive_ptr_release(const T* p) {
    if (p->ref_count_.fetch_sub(1, std::memory_order_acq_rel) == 1) {
        delete p;
    }
}

// The intrusive_ptr smart pointer - stores only 1 pointer
template<typename T>
class intrusive_ptr {
    T* ptr_ = nullptr;

    void add_ref() { if (ptr_) intrusive_ptr_add_ref(ptr_); }
    void release() { if (ptr_) intrusive_ptr_release(ptr_); }

public:
    // Constructors
    intrusive_ptr() = default;
    explicit intrusive_ptr(T* p, bool add_ref_flag = true) : ptr_(p) {
        if (add_ref_flag) add_ref();
    }

    // Copy
    intrusive_ptr(const intrusive_ptr& other) : ptr_(other.ptr_) { add_ref(); }
    intrusive_ptr& operator=(const intrusive_ptr& other) {
        if (this != &other) {
            release();
            ptr_ = other.ptr_;
            add_ref();
        }
        return *this;
    }

    // Move
    intrusive_ptr(intrusive_ptr&& other) noexcept : ptr_(other.ptr_) {
        other.ptr_ = nullptr;
    }
    intrusive_ptr& operator=(intrusive_ptr&& other) noexcept {
        if (this != &other) {
            release();
            ptr_ = other.ptr_;
            other.ptr_ = nullptr;
        }
        return *this;
    }

    // Destructor
    ~intrusive_ptr() { release(); }

    // Access
    T* get() const { return ptr_; }
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    explicit operator bool() const { return ptr_ != nullptr; }

    void reset() { release(); ptr_ = nullptr; }
};

// Example class with embedded ref count
class Document : public RefCounted {
    std::string title_;
public:
    explicit Document(std::string title) : title_(std::move(title)) {
        std::cout << "  Document(\"" << title_ << "\") created\n";
    }
    ~Document() override {
        std::cout << "  Document(\"" << title_ << "\") destroyed\n";
    }
    const std::string& title() const { return title_; }
};

int main() {
    std::cout << "=== intrusive_ptr demo ===\n\n";

    std::cout << "sizeof(intrusive_ptr<Document>): "
              << sizeof(intrusive_ptr<Document>) << " bytes (1 pointer)\n";
    std::cout << "sizeof(std::shared_ptr<Document>): "
              << sizeof(std::shared_ptr<Document>) << " bytes (2 pointers)\n\n";

    // Create
    intrusive_ptr<Document> p1(new Document("Report"));
    std::cout << "After create: use_count=" << p1->use_count() << "\n";

    // Copy - atomic increment
    {
        intrusive_ptr<Document> p2 = p1;
        std::cout << "After copy:   use_count=" << p1->use_count() << "\n";

        intrusive_ptr<Document> p3 = p2;
        std::cout << "After copy2:  use_count=" << p1->use_count() << "\n";
    }
    std::cout << "After scope:  use_count=" << p1->use_count() << "\n";

    // Safe to create intrusive_ptr from raw pointer!
    Document* raw = p1.get();
    {
        intrusive_ptr<Document> p4(raw);  // Adds ref - safe!
        std::cout << "From raw ptr: use_count=" << p1->use_count() << "\n";
    }
    std::cout << "After p4:     use_count=" << p1->use_count() << "\n";

    // Move
    intrusive_ptr<Document> p5 = std::move(p1);
    std::cout << "After move:   p1=" << (p1 ? "valid" : "null")
              << ", p5 use_count=" << p5->use_count() << "\n";

    std::cout << "\n--- Cleanup ---\n";
    return 0;
}
```

**Output:**

```text
=== intrusive_ptr demo ===

sizeof(intrusive_ptr<Document>): 8 bytes (1 pointer)
sizeof(std::shared_ptr<Document>): 16 bytes (2 pointers)

  Document("Report") created
After create: use_count=1
After copy:   use_count=2
After copy2:  use_count=3
After scope:  use_count=1
From raw ptr: use_count=2
After p4:     use_count=1
After move:   p1=null, p5 use_count=1

--- Cleanup ---
  Document("Report") destroyed
```

Notice that creating `p4` from a raw pointer safely incremented the count. That is the key advantage over `shared_ptr` for C API interop scenarios.

### Q3: List use cases where `intrusive_ptr` is preferred over `shared_ptr`

| Use Case | Why Intrusive is Better |
| --- | --- |
| **COM objects** (`IUnknown::AddRef/Release`) | Objects already have embedded ref counts |
| **Kernel / OS handles** | Reference counts are part of the object in the kernel |
| **Game engines** (entity systems) | Millions of objects - save 1 allocation + 8 bytes per pointer |
| **C API interop** | Can pass raw pointers and reconstruct smart pointer safely |
| **Lock-free data structures** | Simpler atomic operations on a single counter |
| **Embedded systems** | Minimal memory overhead |
| **Serialization/networking** | Object identity is preserved across raw pointer roundtrips |

The C API interop case is the most compelling. Here is why `shared_ptr` fails in that scenario and `intrusive_ptr` succeeds:

```cpp
#include <iostream>
#include <memory>
#include <atomic>

// Example: C API interop (the killer use case for intrusive_ptr)
// With shared_ptr, passing a raw pointer to C loses the control block
// With intrusive_ptr, the ref count travels with the object

struct CApiObject : public RefCounted {  // (using RefCounted from Q2)
    int handle;
    explicit CApiObject(int h) : handle(h) {}
};

// Simulated C callback
// void c_callback(void* user_data) {
//     CApiObject* obj = static_cast<CApiObject*>(user_data);
//     // With intrusive_ptr: safe to wrap back into smart pointer
//     intrusive_ptr<CApiObject> safe(obj);  // increments ref count
//     // Use safely...
// }  // decrements ref count

// With shared_ptr this would be UNSAFE:
// void c_callback(void* user_data) {
//     auto* obj = static_cast<Widget*>(user_data);
//     std::shared_ptr<Widget> sp(obj);  // creates NEW control block!
//     // Double-free when both shared_ptrs try to delete
// }

int main() {
    std::cout << "=== Why intrusive is safer with C interop ===\n";
    std::cout << "shared_ptr: raw pointer loses control block -> double-free risk\n";
    std::cout << "intrusive:  ref count is IN the object -> always safe to rewrap\n";

    std::cout << "\n=== Memory comparison (1M objects) ===\n";
    constexpr size_t N = 1'000'000;
    size_t shared_overhead = N * (sizeof(std::shared_ptr<int>) + 32);  // ~32 byte control block
    size_t intrusive_overhead = N * (sizeof(void*) + sizeof(std::atomic<int>));
    std::cout << "shared_ptr total:   ~" << shared_overhead / 1024 / 1024 << " MB\n";
    std::cout << "intrusive_ptr total: ~" << intrusive_overhead / 1024 / 1024 << " MB\n";

    return 0;
}
```

---

## Notes

- `std::shared_ptr` is the safe default - use it unless profiling shows it's a bottleneck.
- Intrusive ref counting requires **cooperative objects** (they must inherit a base class or embed a counter).
- Boost provides `boost::intrusive_ptr` - a well-tested production implementation.
- C++26 may introduce `std::retain_ptr` or similar to standardize intrusive counting.
- Never mix `shared_ptr` and intrusive counting for the same object - pick one ownership strategy.
