# Know std::inplace_vector (C++26) for fixed-capacity, stack-allocated vectors

**Category:** Standard Library Containers  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/container/inplace_vector>  

---

## Topic Overview

`std::inplace_vector<T, N>` (C++26, P0843) is a dynamically-sized container with a **fixed maximum capacity `N`**, where all storage is allocated **inline** (on the stack or within the containing object). It combines the API of `std::vector` with the performance of `std::array` - no heap allocations, ever.

If you've ever worked with embedded systems, real-time audio, or any environment where you cannot afford an allocator call, this is the container you've been wanting. It gives you the push_back/pop_back/iterate API you're used to, backed entirely by stack storage.

### Key Properties

| Property | `std::vector<T>` | `std::inplace_vector<T,N>` | `std::array<T,N>` |
| --- | --- | --- | --- |
| Dynamic size | Yes (unbounded) | Yes (bounded by N) | No (fixed = N) |
| Heap allocation | Yes | **Never** | Never |
| Capacity | Dynamic (realloc) | Fixed at N | Fixed at N |
| `push_back()` | Amortized O(1) | O(1) if size < N | N/A |
| Can exceed capacity | Yes (realloc) | **No** (UB or exception) | N/A |
| `constexpr` support | C++20 (limited) | Full | Full |
| Trivially copyable | No | Yes (if T is) | Yes (if T is) |

### When to Use

- **Embedded / real-time systems:** No heap allocation allowed.
- **Small buffer optimization:** Known upper bound on collection size.
- **Performance-critical hot paths:** Avoid allocator overhead.
- **As a member of another class:** Inline storage avoids pointer indirection.

### Core API

The class below is a hand-rolled demonstration since C++26 isn't out yet. The real `std::inplace_vector` will have the same interface. Pay attention to how `alignas(T) std::byte storage_[N * sizeof(T)]` gives us properly aligned raw storage without constructing anything - that's placement new territory.

```cpp
// NOTE: std::inplace_vector is C++26. As of 2024, use boost::static_vector
// or implement a simple version.

// Conceptual API:
// std::inplace_vector<int, 16> v;
// v.push_back(42);              // OK if size < 16
// v.try_push_back(99);          // Returns pointer or nullptr (no exception)
// v.unchecked_push_back(99);    // UB if full (fastest, no check)
// v.size();                     // Current number of elements
// v.capacity();                 // Always N (16)
// v.max_size();                 // Always N (16)
// v[i];                         // Random access
// v.data();                     // Contiguous memory (stack-allocated)

#include <iostream>
#include <array>
#include <algorithm>
#include <cassert>
#include <stdexcept>

// Simple inplace_vector implementation for demonstration
template <typename T, std::size_t N>
class InplaceVector {
    alignas(T) std::byte storage_[N * sizeof(T)];
    std::size_t size_ = 0;

    T* data_ptr() { return reinterpret_cast<T*>(storage_); }
    const T* data_ptr() const { return reinterpret_cast<const T*>(storage_); }

public:
    InplaceVector() = default;

    ~InplaceVector() {
        for (std::size_t i = 0; i < size_; ++i)
            data_ptr()[i].~T();
    }

    void push_back(const T& val) {
        if (size_ >= N) throw std::bad_alloc();
        new (&data_ptr()[size_]) T(val);
        ++size_;
    }

    T* try_push_back(const T& val) {
        if (size_ >= N) return nullptr;
        new (&data_ptr()[size_]) T(val);
        return &data_ptr()[size_++];
    }

    T& operator[](std::size_t i) { return data_ptr()[i]; }
    const T& operator[](std::size_t i) const { return data_ptr()[i]; }

    std::size_t size() const { return size_; }
    static constexpr std::size_t capacity() { return N; }
    bool empty() const { return size_ == 0; }
    bool full() const { return size_ == N; }

    T* data() { return data_ptr(); }
    T* begin() { return data_ptr(); }
    T* end() { return data_ptr() + size_; }
    const T* begin() const { return data_ptr(); }
    const T* end() const { return data_ptr() + size_; }
};

int main() {
    InplaceVector<int, 8> v;

    v.push_back(10);
    v.push_back(20);
    v.push_back(30);

    std::cout << "Size: " << v.size() << "/" << v.capacity() << "\n";
    // Output: Size: 3/8

    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Output: 10 20 30

    // Fill to capacity
    for (int i = 4; i <= 8; ++i) v.push_back(i * 10);

    std::cout << "Full: " << std::boolalpha << v.full() << "\n";
    // Output: Full: true

    // try_push_back returns nullptr when full
    auto* result = v.try_push_back(999);
    std::cout << "try_push_back on full: " << (result ? "success" : "failed") << "\n";
    // Output: try_push_back on full: failed

    // Sort works (contiguous memory + random access)
    std::sort(v.begin(), v.end());
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Output: 10 20 30 40 50 60 70 80

    return 0;
}
```

The `try_push_back` variant is worth using whenever you're not sure whether the buffer is full - it returns a pointer to the new element on success, or `nullptr` on failure, with no exception overhead.

### Important Notes

- `std::inplace_vector` **never allocates** heap memory - all storage is within the object itself.
- Exceeding capacity with `push_back()` throws `std::bad_alloc`. Use `try_push_back()` for non-throwing insertion.
- `unchecked_push_back()` has **undefined behavior** if full - use only when you've verified capacity.
- The entire object is trivially copyable if `T` is, making it suitable for `memcpy`, serialization, and shared memory.
- `constexpr`-friendly: can be used in compile-time computations.

---

## Self-Assessment

### Q1: Use std::inplace_vector<int,16> as a fixed-capacity alternative to std::vector

Here the simplified version uses `std::array` under the hood for brevity. Notice that all standard algorithms work because the storage is contiguous - `std::sort`, `std::accumulate`, and everything else in `<algorithm>` and `<numeric>` just work.

```cpp
#include <iostream>
#include <vector>
#include <array>
#include <algorithm>
#include <numeric>
#include <stdexcept>

// Simplified inplace_vector for demonstration (C++26 not yet available)
template <typename T, std::size_t Cap>
class inplace_vector {
    std::array<T, Cap> buf_{};
    std::size_t sz_ = 0;
public:
    void push_back(const T& v) {
        if (sz_ >= Cap) throw std::bad_alloc();
        buf_[sz_++] = v;
    }
    T& operator[](std::size_t i) { return buf_[i]; }
    const T& operator[](std::size_t i) const { return buf_[i]; }
    std::size_t size() const { return sz_; }
    static constexpr std::size_t capacity() { return Cap; }
    T* begin() { return buf_.data(); }
    T* end() { return buf_.data() + sz_; }
    const T* begin() const { return buf_.data(); }
    const T* end() const { return buf_.data() + sz_; }
    T* data() { return buf_.data(); }
};

int main() {
    // Fixed-capacity vector: max 16 ints, zero heap allocations
    inplace_vector<int, 16> v;

    for (int i = 1; i <= 10; ++i) {
        v.push_back(i * 10);
    }

    std::cout << "Size: " << v.size() << " / " << v.capacity() << "\n";
    // Output: Size: 10 / 16

    // All std algorithms work (contiguous memory)
    std::sort(v.begin(), v.end(), std::greater<>{});
    std::cout << "Sorted desc: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Output: Sorted desc: 100 90 80 70 60 50 40 30 20 10

    int sum = std::accumulate(v.begin(), v.end(), 0);
    std::cout << "Sum: " << sum << "\n";
    // Output: Sum: 550

    // Compare with std::vector:
    // std::vector<int> heap_v;  // Allocates on HEAP
    // heap_v.push_back(42);     // May trigger realloc

    // inplace_vector<int, 16>:  // All storage on STACK (or inline)
    // No heap allocation, no reallocation, no iterator invalidation from capacity growth

    // Address check: data() is the object's own memory
    std::cout << "Object at: " << (void*)&v << "\n";
    std::cout << "Data at:   " << (void*)v.data() << "\n";
    // These are very close (within the same object) - stack allocated!

    return 0;
}
```

**How it works:**

- `inplace_vector<int, 16>` stores up to 16 ints directly inside the object - on the stack if local, or inline if a member.
- No `new`/`delete` calls ever happen. The `std::array<T, Cap>` member provides the storage.
- Standard algorithms work directly because the data is contiguous.
- This is the C++26 standard equivalent of `boost::static_vector` and LLVM's `SmallVector` (without the fallback to heap).

### Q2: Show the compile error when attempting to exceed the static capacity

The C++26 standard defines three push_back variants with clearly different contracts. Understanding which one to use in which situation is the main API decision point for this container.

```cpp
#include <iostream>
#include <array>
#include <stdexcept>

template <typename T, std::size_t Cap>
class inplace_vector {
    std::array<T, Cap> buf_{};
    std::size_t sz_ = 0;
public:
    // push_back: throws if full
    void push_back(const T& v) {
        if (sz_ >= Cap) throw std::bad_alloc();
        buf_[sz_++] = v;
    }

    // try_push_back: returns false if full (non-throwing)
    bool try_push_back(const T& v) {
        if (sz_ >= Cap) return false;
        buf_[sz_++] = v;
        return true;
    }

    // unchecked_push_back: UB if full! (for maximum performance)
    void unchecked_push_back(const T& v) {
        buf_[sz_++] = v;  // No check - caller must guarantee space
    }

    std::size_t size() const { return sz_; }
    static constexpr std::size_t capacity() { return Cap; }
};

int main() {
    inplace_vector<int, 4> v;

    // Fill to capacity
    v.push_back(1);
    v.push_back(2);
    v.push_back(3);
    v.push_back(4);
    std::cout << "Size: " << v.size() << "/" << v.capacity() << "\n";
    // Output: Size: 4/4

    // --- Method 1: push_back throws ---
    try {
        v.push_back(5);  // Capacity exceeded!
    } catch (const std::bad_alloc& e) {
        std::cout << "push_back threw std::bad_alloc: " << e.what() << "\n";
    }
    // Output: push_back threw std::bad_alloc: std::bad_alloc

    // --- Method 2: try_push_back returns false ---
    bool ok = v.try_push_back(5);
    std::cout << "try_push_back: " << std::boolalpha << ok << "\n";
    // Output: try_push_back: false

    // --- Method 3: unchecked_push_back - UNDEFINED BEHAVIOR ---
    // v.unchecked_push_back(5);  // DON'T DO THIS - writes past buffer!

    // The C++26 standard specifies:
    // - push_back(val) -> throws std::bad_alloc if size() == capacity()
    // - try_push_back(val) -> returns T* (nullptr if full)
    // - unchecked_push_back(val) -> precondition: size() < capacity()
    //
    // Note: There is no "compile error" for exceeding capacity at runtime.
    // The capacity is a compile-time constant, but insertions are runtime operations.
    // Static analysis or constexpr evaluation might catch it at compile time.

    return 0;
}
```

**How it works:**

- The capacity `N` is a compile-time constant, but `push_back()` is a runtime operation - overflow is detected at **runtime**, not compile time.
- `push_back()` throws `std::bad_alloc` when full (in the C++26 standard).
- `try_push_back()` is the non-throwing alternative - returns `nullptr` on failure.
- `unchecked_push_back()` is the fastest path but has a precondition (UB if violated).
- A **compile-time error** only occurs if you attempt to overflow during a `constexpr` evaluation - the compiler rejects UB in constant expressions.

### Q3: Compare inplace_vector with boost::static_vector for pre-C++26 codebases

If you need this feature today, `boost::static_vector` is the nearest drop-in. The migration path to C++26 is intentionally smooth because the API was designed with that compatibility in mind.

```cpp
#include <iostream>
#include <vector>
#include <array>

// Feature comparison: std::inplace_vector (C++26) vs boost::static_vector

int main() {
    std::cout << "=== inplace_vector vs boost::static_vector ===\n\n";

    // +-----------------------+--------------------------+---------------------------+
    // | Feature               | std::inplace_vector<T,N> | boost::static_vector<T,N> |
    // +-----------------------+--------------------------+---------------------------+
    // | Standard              | C++26                    | Boost 1.71+ (2019)        |
    // | Header                | <inplace_vector>         | <boost/container/...>     |
    // | Heap allocation       | Never                    | Never                     |
    // | Overflow push_back    | throws std::bad_alloc    | throws std::bad_alloc     |
    // | try_push_back         | Yes (returns T*)         | No (Boost 1.84 adds it)   |
    // | unchecked_push_back   | Yes (UB if full)         | No                        |
    // | constexpr             | Full                     | Partial                   |
    // | trivially copyable    | Yes (if T is)            | Implementation-dependent  |
    // | Ranges support        | Yes (C++26)              | Limited                   |
    // | Compiler support      | Not yet (2024)           | GCC, Clang, MSVC          |
    // | No Boost dependency   | Yes                      | No                        |
    // +-----------------------+--------------------------+---------------------------+

    // Pre-C++26 alternatives:
    // 1. boost::container::static_vector<T, N>  - closest match
    // 2. LLVM SmallVector<T, N>                 - falls back to heap if exceeded
    // 3. absl::InlinedVector<T, N>              - similar to SmallVector
    // 4. etl::vector<T, N>                      - embedded template library
    // 5. Manual: std::array<T, N> + size counter - simplest

    // Manual polyfill using std::array:
    struct FixedBuf {
        std::array<int, 8> data{};
        std::size_t sz = 0;

        void push_back(int v) {
            if (sz >= 8) throw std::bad_alloc();
            data[sz++] = v;
        }
        int& operator[](std::size_t i) { return data[i]; }
        std::size_t size() const { return sz; }
    };

    FixedBuf buf;
    buf.push_back(10);
    buf.push_back(20);
    buf.push_back(30);
    std::cout << "Manual polyfill: [";
    for (std::size_t i = 0; i < buf.size(); ++i) {
        if (i > 0) std::cout << ", ";
        std::cout << buf[i];
    }
    std::cout << "]\n";
    // Output: Manual polyfill: [10, 20, 30]

    // Key difference: SmallVector/InlinedVector FALL BACK to heap when N is exceeded
    // inplace_vector/static_vector NEVER use heap - they throw or return failure

    std::cout << "\nMigration path:\n";
    std::cout << "  Pre-C++26: boost::container::static_vector<T, N>\n";
    std::cout << "  C++26:     std::inplace_vector<T, N>\n";
    std::cout << "  The API is intentionally similar for easy migration.\n";

    return 0;
}
```

**How it works:**

- `std::inplace_vector` (C++26) was designed based on experience with `boost::static_vector`, LLVM's `SmallVector`, and game industry needs.
- **Key API additions in C++26:** `try_push_back()` (non-throwing), `unchecked_push_back()` (no-check), full `constexpr` support.
- **Migration:** `boost::static_vector` shares the same concept and most of the API. Switching to `std::inplace_vector` when C++26 is available should be a near-mechanical replacement.
- **SmallVector vs inplace_vector:** SmallVector has a heap fallback when capacity is exceeded (small buffer optimization pattern), while `inplace_vector` strictly refuses to allocate heap memory.

---

## Notes

- **Stack size awareness:** `inplace_vector<LargeStruct, 1000>` allocates `1000 * sizeof(LargeStruct)` bytes on the stack. Watch out for stack overflow with large N or large T.
- **Trivially copyable:** If T is trivially copyable, so is `inplace_vector<T, N>` - enabling `memcpy`, `memmove`, and use in shared memory / IPC.
- **Zero-cost when empty:** An empty `inplace_vector<int, 16>` still occupies `16 * sizeof(int)` + size overhead on the stack.
- **Embedded systems:** This is the go-to container for embedded C++ where `malloc`/`free` are unavailable or forbidden.
- **Real-time systems:** Deterministic timing - no allocation jitter, no worst-case realloc stalls.
