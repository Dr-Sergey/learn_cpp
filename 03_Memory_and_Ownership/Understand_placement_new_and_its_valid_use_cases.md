# Understand Placement `new` and Its Valid Use Cases

**Category:** Memory and Ownership  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/new>  

---

## Topic Overview

### What Is Placement `new`

Placement `new` constructs an object **at a specific memory address** without allocating memory. Regular `new` does two things: allocate then construct. Placement `new` does only the second part - you supply the memory yourself.

```cpp
void* buffer = /* some pre-allocated memory */;
T* obj = new (buffer) T(args...);  // construct T at buffer's address
```

This separation is what makes memory pools, custom containers, and type-erased buffers possible.

### When to Use It

| Use Case | Why |
| --- | --- |
| Memory pools / arenas | Pre-allocate, then construct objects in-place |
| Manual variant/optional | Construct different types in aligned storage |
| Embedded/real-time systems | Avoid heap allocation entirely |
| Custom containers | Separate allocation from construction |
| Shared memory / memory-mapped files | Construct objects at OS-provided addresses |

### Critical Rules

The main thing to internalize is the asymmetry: placement `new` borrows memory from you and hands it back unchanged. You own cleanup on both sides.

1. **Memory must be properly aligned** for the type
2. **Memory must be large enough** for the type
3. **You must manually call the destructor** - `delete` would try to free the memory
4. Placement `new` is **not replaceable** (unlike regular `operator new`)

```cpp
Regular new:              Placement new:
┌─────────────────┐       ┌─────────────────┐
│ 1. allocate     │       │ (buffer exists)  │
│ 2. construct T  │       │ 1. construct T   │
└─────────────────┘       └─────────────────┘

delete:                   Placement cleanup:
┌─────────────────┐       ┌─────────────────┐
│ 1. destroy ~T() │       │ 1. destroy ~T()  │
│ 2. deallocate   │       │ (don't free!)    │
└─────────────────┘       └─────────────────┘
```

---

## Self-Assessment

### Q1: Use placement `new` to construct an object in pre-allocated aligned storage

This example shows three progressively more sophisticated patterns. Method 2 using `std::construct_at` is the C++20-preferred spelling - it does the same thing as placement `new` but is constexpr-friendly and slightly harder to misuse.

```cpp
#include <iostream>
#include <cstddef>
#include <new>
#include <string>
#include <memory>

struct Widget {
    int id;
    std::string name;

    Widget(int i, std::string n) : id(i), name(std::move(n)) {
        std::cout << "  Widget(" << id << ", \"" << name << "\") constructed\n";
    }
    ~Widget() {
        std::cout << "  Widget(" << id << ", \"" << name << "\") destroyed\n";
    }
};

int main() {
    // Method 1: Using alignas + std::byte array
    std::cout << "=== Stack-allocated aligned buffer ===\n";
    {
        alignas(Widget) std::byte buffer[sizeof(Widget)];

        std::cout << "Buffer address: " << static_cast<void*>(buffer) << "\n";
        std::cout << "Buffer size: " << sizeof(buffer) << " bytes\n";
        std::cout << "Widget alignment: " << alignof(Widget) << "\n\n";

        // Construct Widget in the buffer
        Widget* w = new (buffer) Widget(1, "Alice");

        // Use the object
        std::cout << "Using: " << w->name << " (id=" << w->id << ")\n";

        // MUST manually destroy - never use delete!
        w->~Widget();   // explicit destructor call
    }

    // Method 2: Using std::construct_at (C++20) - preferred
    std::cout << "\n=== Using std::construct_at (C++20) ===\n";
    {
        alignas(Widget) std::byte buffer[sizeof(Widget)];

        Widget* w = std::construct_at(
            reinterpret_cast<Widget*>(buffer), 2, "Bob"
        );

        std::cout << "Using: " << w->name << " (id=" << w->id << ")\n";

        std::destroy_at(w);  // cleaner than w->~Widget()
    }

    // Method 3: Multiple objects in a flat buffer
    std::cout << "\n=== Array of objects in flat buffer ===\n";
    {
        constexpr int N = 3;
        alignas(Widget) std::byte buffer[sizeof(Widget) * N];

        Widget* arr[N];
        for (int i = 0; i < N; ++i) {
            void* slot = buffer + i * sizeof(Widget);
            arr[i] = new (slot) Widget(i + 10, "Item" + std::to_string(i));
        }

        std::cout << "\nAll constructed. Destroying in reverse:\n";
        for (int i = N - 1; i >= 0; --i) {
            arr[i]->~Widget();
        }
    }

    return 0;
}
```

**Output:**

```text
=== Stack-allocated aligned buffer ===
Buffer address: 0x7ffe...
Buffer size: 40 bytes
Widget alignment: 8

  Widget(1, "Alice") constructed
Using: Alice (id=1)
  Widget(1, "Alice") destroyed

=== Using std::construct_at (C++20) ===
  Widget(2, "Bob") constructed
Using: Bob (id=2)
  Widget(2, "Bob") destroyed

=== Array of objects in flat buffer ===
  Widget(10, "Item0") constructed
  Widget(11, "Item1") constructed
  Widget(12, "Item2") constructed

All constructed. Destroying in reverse:
  Widget(12, "Item2") destroyed
  Widget(11, "Item1") destroyed
  Widget(10, "Item0") destroyed
```

Reverse-order destruction in Method 3 mirrors what the compiler does for local arrays - a good habit to adopt whenever you control destruction order manually.

### Q2: Explain why you must call the destructor explicitly when using placement `new`

The reason is straightforward once you see what the destructor owns. The buffer is your memory and you handle its lifetime. But the object constructed inside that buffer may own its own resources - heap-allocated strings, arrays, file handles - and those need cleanup through the destructor. Skipping the destructor leaks those resources. Calling `delete` instead crashes because `delete` tries to free memory it did not allocate.

```cpp
#include <iostream>
#include <string>
#include <cstddef>
#include <new>

struct Resource {
    std::string data;
    int* heap_array;

    Resource(const std::string& d, int n)
        : data(d), heap_array(new int[n])
    {
        std::cout << "  Resource(\"" << data << "\") acquired " << n << " ints\n";
    }

    ~Resource() {
        std::cout << "  ~Resource(\"" << data << "\") releasing heap array\n";
        delete[] heap_array;
    }
};

int main() {
    std::cout << "=== Why explicit destructor is required ===\n\n";

    alignas(Resource) std::byte buffer[sizeof(Resource)];

    // Placement new: constructs in our buffer (no heap allocation for Resource itself)
    Resource* r = new (buffer) Resource("Important", 100);

    // The Resource object owns:
    // 1. A std::string (which internally heap-allocates)
    // 2. An int[] (heap-allocated)

    // If we DON'T call destructor:
    // - string's internal buffer LEAKS
    // - heap_array LEAKS
    // - The buffer memory itself is fine (it's on the stack)
    //   but the RESOURCES owned by the object leak!

    // If we call delete:
    // delete r;  // UNDEFINED BEHAVIOR!
    // delete would:
    // 1. Call destructor (OK)
    // 2. Try to free buffer as if it came from operator new (CRASH!)
    //    But buffer is on the stack!

    // CORRECT: call destructor explicitly
    r->~Resource();
    std::cout << "\nAll resources properly cleaned up.\n";
    std::cout << "Buffer (stack memory) naturally goes out of scope.\n";

    // Summary:
    std::cout << "\n=== Rules ===\n";
    std::cout << "Placement new -> explicit destructor call (p->~T())\n";
    std::cout << "Regular new   -> delete (calls dtor + frees memory)\n";
    std::cout << "NEVER use delete on placement-new objects!\n";
    std::cout << "NEVER skip destructor - owned resources will leak!\n";

    return 0;
}
```

### Q3: Show a use case in a memory pool where placement `new` avoids heap fragmentation

Here is the payoff of the whole technique. The pool makes one large allocation up front. All individual objects are constructed inside that block, so the heap sees one allocation and one deallocation, regardless of how many objects come and go. Cache locality is also improved because all objects are contiguous in memory.

```cpp
#include <iostream>
#include <cstddef>
#include <new>
#include <vector>
#include <string>
#include <memory>

// A simple object pool using placement new
template<typename T, size_t N>
class ObjectPool {
    // Pre-allocated contiguous block
    alignas(T) std::byte storage_[N * sizeof(T)];
    bool occupied_[N] = {};
    size_t count_ = 0;

    void* slot(size_t i) { return &storage_[i * sizeof(T)]; }

public:
    // Construct a T in the next available slot
    template<typename... Args>
    T* create(Args&&... args) {
        for (size_t i = 0; i < N; ++i) {
            if (!occupied_[i]) {
                occupied_[i] = true;
                ++count_;
                // Placement new - construct in pre-allocated memory
                return new (slot(i)) T(std::forward<Args>(args)...);
            }
        }
        return nullptr;  // pool full
    }

    // Destroy the object and free the slot
    void destroy(T* obj) {
        // Find which slot this object is in
        auto* base = storage_;
        auto* addr = reinterpret_cast<std::byte*>(obj);
        size_t index = (addr - base) / sizeof(T);

        if (index < N && occupied_[index]) {
            obj->~T();  // Explicit destructor - must not use delete!
            occupied_[index] = false;
            --count_;
        }
    }

    size_t size() const { return count_; }
    size_t capacity() const { return N; }

    // Destroy all active objects
    ~ObjectPool() {
        for (size_t i = 0; i < N; ++i) {
            if (occupied_[i]) {
                reinterpret_cast<T*>(slot(i))->~T();
            }
        }
    }
};

struct Particle {
    float x, y, vx, vy;
    std::string tag;

    Particle(float x, float y, std::string t)
        : x(x), y(y), vx(0), vy(0), tag(std::move(t))
    {
        std::cout << "  Particle(\"" << tag << "\") at (" << x << "," << y << ")\n";
    }
    ~Particle() {
        std::cout << "  ~Particle(\"" << tag << "\")\n";
    }
};

int main() {
    std::cout << "=== Object Pool with Placement New ===\n\n";

    ObjectPool<Particle, 5> pool;
    std::cout << "Pool capacity: " << pool.capacity() << "\n\n";

    // Create particles - no heap allocation! All in contiguous pre-allocated block
    Particle* p1 = pool.create(1.0f, 2.0f, "alpha");
    Particle* p2 = pool.create(3.0f, 4.0f, "beta");
    Particle* p3 = pool.create(5.0f, 6.0f, "gamma");

    std::cout << "\nPool size: " << pool.size() << "/" << pool.capacity() << "\n";

    // Recycle a slot
    std::cout << "\nDestroying beta:\n";
    pool.destroy(p2);
    std::cout << "Pool size: " << pool.size() << "/" << pool.capacity() << "\n";

    // Reuse the slot
    std::cout << "\nCreating delta (reuses beta's slot):\n";
    Particle* p4 = pool.create(7.0f, 8.0f, "delta");

    std::cout << "\nPool size: " << pool.size() << "/" << pool.capacity() << "\n";

    // Benefits of this approach:
    std::cout << "\n=== Benefits ===\n";
    std::cout << "1. Zero heap fragmentation (contiguous block)\n";
    std::cout << "2. Cache-friendly (objects are adjacent in memory)\n";
    std::cout << "3. O(1) alloc/dealloc (no heap walk)\n";
    std::cout << "4. No malloc/free overhead\n";

    std::cout << "\n=== Pool destructor cleans up remaining ===\n";
    return 0;
}
```

---

## Notes

- Placement `new` is the mechanism behind `std::construct_at`, `std::allocator::construct`, and all custom containers.
- In C++20, prefer `std::construct_at` and `std::destroy_at` over raw placement new - they're constexpr-friendly and clearer.
- Never use `delete` on a placement-new object - it will try to free memory that wasn't `operator new`-allocated.
- Always ensure the buffer outlives the constructed objects.
- Placement `new` is the only form of `operator new` that **cannot be replaced** (it's a no-op that just returns the pointer).
