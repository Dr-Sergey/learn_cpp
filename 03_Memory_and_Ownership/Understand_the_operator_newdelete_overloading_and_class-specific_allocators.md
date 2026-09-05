# Understand the operator new/delete Overloading and Class-Specific Allocators

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/new/operator_new>  

---

## Topic Overview

### Levels of `new`/`delete` Customization

You can intercept allocation at three different levels. Knowing which level to choose is most of the battle - the global override is a sledgehammer, the class-specific version is a scalpel.

| Level | Scope | How |
| --- | --- | --- |
| **Global** | All allocations in program | `void* operator new(size_t)` at global scope |
| **Class-specific** | Only that class | `static void* operator new(size_t)` inside class |
| **Placement** | Custom parameters | `void* operator new(size_t, Arena&)` |

### Lookup Rules

When you write `new MyClass`, the compiler searches for an allocator in a specific order. Class-specific always wins over global.

```cpp
new MyClass;
// 1. Look for MyClass::operator new (class-specific)
// 2. If not found, use ::operator new (global)
// 3. Global can be replaced by user code
```

### All Overloadable Forms

| Operator | Purpose |
| --- | --- |
| `operator new(size_t)` | Single object |
| `operator new[](size_t)` | Array |
| `operator delete(void*)` | Single dealloc |
| `operator delete[](void*)` | Array dealloc |
| `operator delete(void*, size_t)` | Sized dealloc (C++14) |
| `operator new(size_t, align_val_t)` | Aligned alloc (C++17) |

---

## Self-Assessment

### Q1: Override `operator new` and `operator delete` for a class to use a pool allocator

A pool allocator pre-allocates a fixed number of slots and hands them out in O(1) time without touching the heap. The `PoolAllocated` base class below does the heavy lifting using a bitset to track which slots are free. Notice the size check in `operator new` - that guards against accidental use when a derived class is larger than `T`, in which case it falls back to the global allocator.

```cpp
#include <iostream>
#include <cstddef>
#include <cstdlib>
#include <array>
#include <bitset>

// Simple fixed-size pool for a specific class
template<typename T, size_t PoolSize = 64>
class PoolAllocated {
    struct Pool {
        alignas(T) unsigned char storage[PoolSize][sizeof(T)];
        std::bitset<PoolSize> used;
        size_t count = 0;

        void* allocate() {
            for (size_t i = 0; i < PoolSize; ++i) {
                if (!used[i]) {
                    used.set(i);
                    ++count;
                    return storage[i];
                }
            }
            throw std::bad_alloc();
        }

        void deallocate(void* p) {
            auto base = reinterpret_cast<unsigned char*>(storage);
            auto addr = reinterpret_cast<unsigned char*>(p);
            size_t index = static_cast<size_t>(addr - base) / sizeof(T);
            if (index < PoolSize && used[index]) {
                used.reset(index);
                --count;
            }
        }
    };

    static Pool& pool() {
        static Pool p;
        return p;
    }

public:
    // Class-specific operator new — uses pool instead of heap
    static void* operator new(size_t size) {
        if (size != sizeof(T)) {
            return ::operator new(size);  // Derived class? Fallback
        }
        std::cout << "  [pool] allocating slot\n";
        return pool().allocate();
    }

    static void operator delete(void* p, size_t size) noexcept {
        if (size != sizeof(T)) {
            ::operator delete(p, size);
            return;
        }
        std::cout << "  [pool] freeing slot\n";
        pool().deallocate(p);
    }

    static size_t pool_usage() { return pool().count; }
};

struct Bullet : PoolAllocated<Bullet, 100> {
    float x, y;
    int damage;
    Bullet(float x, float y, int d) : x(x), y(y), damage(d) {}
};

int main() {
    std::cout << "=== Class-specific pool allocator ===\n\n";

    Bullet* b1 = new Bullet(1.0f, 2.0f, 10);  // Uses pool
    Bullet* b2 = new Bullet(3.0f, 4.0f, 20);
    Bullet* b3 = new Bullet(5.0f, 6.0f, 30);

    std::cout << "Pool usage: " << Bullet::pool_usage() << "\n\n";

    delete b2;  // Returns to pool
    std::cout << "After delete b2, pool usage: " << Bullet::pool_usage() << "\n\n";

    // Slot reused!
    Bullet* b4 = new Bullet(7.0f, 8.0f, 40);
    std::cout << "Pool usage after reuse: " << Bullet::pool_usage() << "\n\n";

    delete b1;
    delete b3;
    delete b4;

    std::cout << "Final pool usage: " << Bullet::pool_usage() << "\n";

    return 0;
}
// Expected output:
// === Class-specific pool allocator ===
//
//   [pool] allocating slot
//   [pool] allocating slot
//   [pool] allocating slot
// Pool usage: 3
//
//   [pool] freeing slot
// After delete b2, pool usage: 2
//
//   [pool] allocating slot
// Pool usage after reuse: 3
//
//   [pool] freeing slot
//   [pool] freeing slot
//   [pool] freeing slot
// Final pool usage: 0
```

### Q2: Explain the difference between overloading the global `operator new` vs class-specific `operator new`

The global override is shown commented out deliberately - uncommenting it would intercept every allocation in the program, including those inside the standard library. The `Tracked` class shows the surgical version: only `Tracked` uses the custom allocator, everything else is unaffected.

```cpp
#include <iostream>
#include <cstddef>
#include <cstdlib>
#include <new>

// ============================================================
// GLOBAL operator new: affects ALL allocations in the program
// ============================================================
// Uncomment to activate (affects everything including std::string, std::vector, etc.)
/*
void* operator new(size_t size) {
    std::cout << "[GLOBAL new] " << size << " bytes\n";
    void* p = std::malloc(size);
    if (!p) throw std::bad_alloc();
    return p;
}

void operator delete(void* p) noexcept {
    std::cout << "[GLOBAL delete]\n";
    std::free(p);
}
*/

// ============================================================
// CLASS-SPECIFIC: affects only this class
// ============================================================
struct Tracked {
    int data;

    static void* operator new(size_t size) {
        std::cout << "[Tracked::new] " << size << " bytes\n";
        void* p = std::malloc(size);
        if (!p) throw std::bad_alloc();
        return p;
    }

    static void operator delete(void* p) noexcept {
        std::cout << "[Tracked::delete]\n";
        std::free(p);
    }

    // Array versions too
    static void* operator new[](size_t size) {
        std::cout << "[Tracked::new[]] " << size << " bytes\n";
        void* p = std::malloc(size);
        if (!p) throw std::bad_alloc();
        return p;
    }

    static void operator delete[](void* p) noexcept {
        std::cout << "[Tracked::delete[]]\n";
        std::free(p);
    }
};

struct Untracked {
    int data;
    // No custom operator new — uses global ::operator new
};

int main() {
    std::cout << "=== Global vs Class-Specific ===\n\n";

    std::cout << "--- Class-specific (only Tracked) ---\n";
    Tracked* t = new Tracked{42};          // Tracked::operator new
    delete t;                               // Tracked::operator delete

    Tracked* arr = new Tracked[3];         // Tracked::operator new[]
    delete[] arr;                           // Tracked::operator delete[]

    std::cout << "\n--- Default (Untracked uses global) ---\n";
    Untracked* u = new Untracked{10};      // ::operator new (default)
    delete u;                               // ::operator delete (default)

    std::cout << "\n=== Comparison ===\n";
    std::cout << "Global new:  Affects ALL types (dangerous, hard to debug)\n";
    std::cout << "Class new:   Affects ONE class (safe, targeted)\n";
    std::cout << "Global:      Can't use different strategies per class\n";
    std::cout << "Class:       Each class can have its own allocator\n";
    std::cout << "Inheritance: Derived classes inherit class-specific new\n";
    std::cout << "             (check size in implementation!)\n";

    return 0;
}
```

### Q3: Show how `nothrow new` (`new(std::nothrow)`) avoids exceptions and returns `nullptr` on failure

Regular `new` throws `std::bad_alloc` on failure. `nothrow new` returns `nullptr` instead - but be careful: the "no throw" part only covers the allocation itself. If the constructor throws, that exception still propagates regardless of the `nothrow` tag.

```cpp
#include <iostream>
#include <new>       // std::nothrow
#include <cstddef>
#include <memory>

int main() {
    std::cout << "=== nothrow new ===\n\n";

    // Regular new: throws std::bad_alloc on failure
    // int* p1 = new int[huge_size];  // throws if OOM

    // nothrow new: returns nullptr on failure
    int* p1 = new(std::nothrow) int(42);
    if (p1) {
        std::cout << "Allocated: " << *p1 << "\n";
        delete p1;
    } else {
        std::cout << "Allocation failed (nullptr)\n";
    }

    // Attempt a huge allocation — will return nullptr instead of throwing
    size_t huge = static_cast<size_t>(-1) / 2;  // ~9 exabytes
    int* p2 = new(std::nothrow) int[huge];
    if (!p2) {
        std::cout << "Huge allocation: nullptr (as expected)\n";
    } else {
        delete[] p2;
    }

    // nothrow new with arrays
    double* arr = new(std::nothrow) double[1000];
    if (arr) {
        arr[0] = 3.14;
        std::cout << "Array allocated: " << arr[0] << "\n";
        delete[] arr;
    }

    // IMPORTANT CAVEAT: nothrow new only prevents exception from allocation.
    // If the CONSTRUCTOR throws, the exception still propagates!
    struct MayThrow {
        MayThrow(bool shouldThrow) {
            if (shouldThrow) throw std::runtime_error("ctor failed");
        }
    };

    try {
        // new(nothrow) prevents bad_alloc, NOT constructor exceptions
        MayThrow* m = new(std::nothrow) MayThrow(true);
        delete m;
    } catch (const std::runtime_error& e) {
        std::cout << "Constructor threw despite nothrow: " << e.what() << "\n";
    }

    std::cout << "\n=== When to use nothrow new ===\n";
    std::cout << "1. C++ code interfacing with C (no exceptions)\n";
    std::cout << "2. Embedded/real-time systems (exceptions disabled)\n";
    std::cout << "3. Graceful degradation instead of crash\n";
    std::cout << "4. Prefer: try/catch with regular new in most C++ code\n";
    std::cout << "5. Best: use containers/smart pointers that handle this\n";

    return 0;
}
// Expected output:
// === nothrow new ===
//
// Allocated: 42
// Huge allocation: nullptr (as expected)
// Array allocated: 3.14
// Constructor threw despite nothrow: ctor failed
//
// === When to use nothrow new ===
// 1. C++ code interfacing with C (no exceptions)
// 2. Embedded/real-time systems (exceptions disabled)
// 3. Graceful degradation instead of crash
// 4. Prefer: try/catch with regular new in most C++ code
// 5. Best: use containers/smart pointers that handle this
```

---

## Notes

- Class-specific `operator new` is `static` even if not declared so - it must be, since the object doesn't exist yet.
- Always override `operator delete` when overriding `operator new` - they must be paired.
- Check `size` parameter in class-specific `new` to handle derived classes correctly.
- Sized deallocation (`operator delete(void*, size_t)`, C++14) helps allocators avoid looking up block size.
- `new(std::nothrow)` doesn't protect against constructor exceptions - only allocation failure.
