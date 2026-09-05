# Know How to Use `std::allocator_traits` to Write Allocator-Aware Code

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/allocator_traits>  

---

## Topic Overview

### What Is `std::allocator_traits`

`std::allocator_traits<Alloc>` is a traits class that wraps any allocator, providing a uniform interface and sensible defaults for operations the allocator doesn't define. You should **never call allocator member functions directly** - always go through `allocator_traits`. That is the whole point of the abstraction: your container or algorithm does not need to care whether the allocator defines a custom `construct()` or not, because `allocator_traits` fills in the right default if it is missing.

### Why It Matters

| Without `allocator_traits` | With `allocator_traits` |
| --- | --- |
| Must call `alloc.allocate(n)` directly | `traits::allocate(alloc, n)` - uniform |
| Must check if `construct()` exists | `traits::construct()` - defaults to placement new |
| Must check if `propagate_on_*` exists | Defaults provided automatically |
| Writing allocator-aware containers is hard | Straightforward interface |

### Key Members of `allocator_traits<Alloc>`

| Member | Default if not in Alloc |
| --- | --- |
| `allocate(a, n)` | Required - must be defined |
| `deallocate(a, p, n)` | Required - must be defined |
| `construct(a, p, args...)` | `::new((void*)p) T(args...)` |
| `destroy(a, p)` | `p->~T()` |
| `max_size(a)` | `numeric_limits<size_type>::max() / sizeof(T)` |
| `rebind_alloc<U>` | `Alloc<U>` if template |
| `propagate_on_container_copy_assignment` | `false_type` |
| `propagate_on_container_move_assignment` | `false_type` |
| `propagate_on_container_swap` | `false_type` |
| `is_always_equal` | `is_empty<Alloc>::type` |

### Minimal Allocator Requirements (C++11+)

The good news is that since C++11, you only need to define two things and the rest is handled by `allocator_traits`. Here is the minimum conforming allocator:

```cpp
template<typename T>
struct MyAlloc {
    using value_type = T;

    MyAlloc() = default;
    template<typename U> MyAlloc(const MyAlloc<U>&) {}

    T* allocate(std::size_t n);
    void deallocate(T* p, std::size_t n);
};
template<typename T, typename U>
bool operator==(const MyAlloc<T>&, const MyAlloc<U>&) { return true; }
template<typename T, typename U>
bool operator!=(const MyAlloc<T>&, const MyAlloc<U>&) { return false; }
```

Everything else is filled in by `allocator_traits`.

---

## Self-Assessment

### Q1: Write a custom allocator that logs every allocation, and plug it into `std::vector`

This example shows two things at once: how little you need to write to get a working allocator, and how `allocator_traits` is the right way to call into it explicitly when you need to.

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <cstdlib>

template<typename T>
struct LoggingAllocator {
    using value_type = T;

    LoggingAllocator() = default;

    // Converting constructor - required for rebind
    template<typename U>
    LoggingAllocator(const LoggingAllocator<U>&) noexcept {}

    T* allocate(std::size_t n) {
        std::cout << "[alloc] " << n << " x " << sizeof(T)
                  << " bytes = " << n * sizeof(T) << " bytes\n";
        T* p = static_cast<T*>(std::malloc(n * sizeof(T)));
        if (!p) throw std::bad_alloc();
        return p;
    }

    void deallocate(T* p, std::size_t n) noexcept {
        std::cout << "[dealloc] " << n << " x " << sizeof(T)
                  << " bytes = " << n * sizeof(T) << " bytes\n";
        std::free(p);
    }
};

template<typename T, typename U>
bool operator==(const LoggingAllocator<T>&, const LoggingAllocator<U>&) { return true; }

template<typename T, typename U>
bool operator!=(const LoggingAllocator<T>&, const LoggingAllocator<U>&) { return false; }

int main() {
    std::cout << "=== vector with LoggingAllocator ===\n";
    std::vector<int, LoggingAllocator<int>> v;

    // allocator_traits::construct is used internally
    v.push_back(10);
    v.push_back(20);
    v.push_back(30);
    v.push_back(40);
    v.push_back(50);

    std::cout << "\nContents: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n\n";

    // Show allocator_traits explicit usage
    using Alloc = LoggingAllocator<int>;
    using Traits = std::allocator_traits<Alloc>;

    Alloc a;
    int* p = Traits::allocate(a, 1);
    Traits::construct(a, p, 42);           // placement new via traits
    std::cout << "Constructed value: " << *p << "\n";
    Traits::destroy(a, p);                 // destructor via traits
    Traits::deallocate(a, p, 1);

    std::cout << "\nmax_size: " << Traits::max_size(a) << "\n";

    return 0;
}
```

**Output (typical):**

```text
=== vector with LoggingAllocator ===
[alloc] 1 x 4 bytes = 4 bytes
[alloc] 2 x 4 bytes = 8 bytes
[dealloc] 1 x 4 bytes = 4 bytes
[alloc] 4 x 4 bytes = 16 bytes
[dealloc] 2 x 4 bytes = 8 bytes
[alloc] 8 x 4 bytes = 32 bytes
[dealloc] 4 x 4 bytes = 16 bytes

Contents: 10 20 30 40 50

[alloc] 1 x 4 bytes = 4 bytes
Constructed value: 42
[dealloc] 1 x 4 bytes = 4 bytes

max_size: 4611686018427387903
[dealloc] 8 x 4 bytes = 32 bytes
```

The growing alloc/dealloc pattern is `vector`'s geometric reallocation at work. Each time capacity doubles, the old buffer is freed and a new one allocated.

### Q2: Explain how `std::scoped_allocator_adaptor` propagates allocators to nested containers

Without `scoped_allocator_adaptor`, an outer container using your custom allocator will pass that allocator to its own storage, but inner containers (like a `vector<vector<int>>`) will just use `std::allocator` for the inner elements. `scoped_allocator_adaptor` fixes this by automatically forwarding the allocator to nested container constructors.

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <scoped_allocator>
#include <string>
#include <cstdlib>

template<typename T>
struct TrackedAlloc {
    using value_type = T;
    std::string tag;

    TrackedAlloc(const std::string& t = "default") : tag(t) {}

    template<typename U>
    TrackedAlloc(const TrackedAlloc<U>& o) noexcept : tag(o.tag) {}

    T* allocate(std::size_t n) {
        std::cout << "[" << tag << "] allocate " << n
                  << " x sizeof(" << typeid(T).name() << ")\n";
        T* p = static_cast<T*>(std::malloc(n * sizeof(T)));
        if (!p) throw std::bad_alloc();
        return p;
    }

    void deallocate(T* p, std::size_t n) noexcept {
        std::cout << "[" << tag << "] deallocate " << n << "\n";
        std::free(p);
    }
};

template<typename T, typename U>
bool operator==(const TrackedAlloc<T>& a, const TrackedAlloc<U>& b) {
    return a.tag == b.tag;
}
template<typename T, typename U>
bool operator!=(const TrackedAlloc<T>& a, const TrackedAlloc<U>& b) {
    return !(a == b);
}

int main() {
    using InnerAlloc = TrackedAlloc<int>;
    using InnerVec = std::vector<int, InnerAlloc>;
    using OuterAlloc = std::scoped_allocator_adaptor<TrackedAlloc<InnerVec>>;
    using OuterVec = std::vector<InnerVec, OuterAlloc>;

    // The scoped_allocator_adaptor propagates the allocator to inner containers
    OuterVec outer(OuterAlloc(TrackedAlloc<InnerVec>("POOL")));

    std::cout << "--- Adding inner vector ---\n";
    // emplace_back creates InnerVec. scoped_allocator_adaptor passes
    // the rebound allocator to the InnerVec constructor automatically
    outer.emplace_back();
    outer[0].push_back(1);
    outer[0].push_back(2);

    std::cout << "\n--- Adding second inner vector ---\n";
    outer.emplace_back();
    outer[1].push_back(3);

    std::cout << "\nOuter size: " << outer.size() << "\n";
    for (size_t i = 0; i < outer.size(); ++i) {
        std::cout << "  inner[" << i << "]: ";
        for (int v : outer[i]) std::cout << v << " ";
        std::cout << "\n";
    }

    std::cout << "\n--- Cleanup ---\n";
    return 0;
}
```

**Key concept:** Without `scoped_allocator_adaptor`, the outer vector would use the custom allocator but inner vectors would use `std::allocator<int>`. With it, the allocator propagates to all nested uses.

### Q3: Show how PMR (`std::pmr`) containers use polymorphic allocators without template bloat

The main downside of classic custom allocators is that they become part of the container's type: `vector<int, MyAlloc>` and `vector<int>` are different types and cannot be passed to the same function. PMR solves this by moving the allocator out of the type and into a runtime pointer. All PMR containers share the same type (`pmr::vector<int>`) regardless of which memory resource they use.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <string>
#include <array>

// A custom memory_resource that tracks allocations
class TrackingResource : public std::pmr::memory_resource {
    std::pmr::memory_resource* upstream_;
    size_t total_allocated_ = 0;
    size_t num_allocations_ = 0;
public:
    explicit TrackingResource(std::pmr::memory_resource* up = std::pmr::get_default_resource())
        : upstream_(up) {}

    size_t total_allocated() const { return total_allocated_; }
    size_t num_allocations() const { return num_allocations_; }

protected:
    void* do_allocate(size_t bytes, size_t alignment) override {
        std::cout << "  [track] allocate " << bytes << " bytes (align " << alignment << ")\n";
        total_allocated_ += bytes;
        ++num_allocations_;
        return upstream_->allocate(bytes, alignment);
    }
    void do_deallocate(void* p, size_t bytes, size_t alignment) override {
        std::cout << "  [track] deallocate " << bytes << " bytes\n";
        upstream_->deallocate(p, bytes, alignment);
    }
    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }
};

int main() {
    // Stack buffer for monotonic_buffer_resource
    std::array<std::byte, 4096> buffer;
    std::pmr::monotonic_buffer_resource mono(buffer.data(), buffer.size(),
                                              std::pmr::null_memory_resource());

    TrackingResource tracker(&mono);

    std::cout << "=== PMR vector ===\n";
    {
        std::pmr::vector<int> v(&tracker);
        v.push_back(10);
        v.push_back(20);
        v.push_back(30);

        std::cout << "Contents: ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n";
    }
    std::cout << "Stats: " << tracker.num_allocations() << " allocations, "
              << tracker.total_allocated() << " bytes total\n";

    std::cout << "\n=== PMR string ===\n";
    {
        std::pmr::string s("Hello, PMR world! This is a long string to force allocation.", &tracker);
        std::cout << "String: " << s << "\n";
    }

    std::cout << "\n=== Nested PMR containers ===\n";
    {
        // PMR containers automatically propagate their allocator to nested containers
        std::pmr::vector<std::pmr::string> vs(&tracker);
        vs.emplace_back("first");
        vs.emplace_back("second string that is long enough to allocate");

        for (const auto& str : vs) std::cout << "  '" << str << "'\n";
    }

    std::cout << "\nTotal: " << tracker.num_allocations() << " allocations, "
              << tracker.total_allocated() << " bytes\n";

    return 0;
}
```

**PMR vs classic allocators:**

| Feature | Classic Allocator | PMR (`polymorphic_allocator`) |
| --- | --- | --- |
| Part of the type? | Yes (`vector<int, MyAlloc>`) | **No** (`pmr::vector<int>`) |
| Runtime swappable? | No | **Yes** |
| Can mix in one container? | No | **Yes** |
| Overhead | Zero (stateless) | One pointer + virtual call |
| Header | `<memory>` | `<memory_resource>` |

---

## Notes

- Always use `allocator_traits` instead of calling allocator methods directly - your code will work with any conforming allocator.
- `scoped_allocator_adaptor` is essential when nested containers should share the same allocator.
- PMR (`std::pmr`) is the modern approach - it decouples allocator type from container type, enabling runtime allocator selection.
- Custom `memory_resource` classes are easier to write than classic allocators - just override 3 virtual functions.
