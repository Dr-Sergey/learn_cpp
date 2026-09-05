# Understand `memory_resource` Interface and How to Chain Fallback Allocators

**Category:** Memory and Ownership  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/memory/memory_resource>  

---

## Topic Overview

### What Is `std::pmr::memory_resource`

`memory_resource` is an abstract base class (polymorphic interface) for custom allocators. The whole idea is that instead of baking the allocator into the container's type, you pass it as a runtime parameter. Here is the interface you implement when you write a custom resource:

```cpp
class memory_resource {
protected:
    virtual void* do_allocate(size_t bytes, size_t alignment) = 0;
    virtual void do_deallocate(void* p, size_t bytes, size_t alignment) = 0;
    virtual bool do_is_equal(const memory_resource& other) const noexcept = 0;
public:
    void* allocate(size_t bytes, size_t alignment = alignof(max_align_t));
    void deallocate(void* p, size_t bytes, size_t alignment = alignof(max_align_t));
    bool is_equal(const memory_resource& other) const noexcept;
};
```

Three virtual functions. That is the whole contract. If you can implement those three, you have a fully usable PMR allocator.

### Chaining with Upstream (Fallback) Allocators

Many `memory_resource` implementations accept an **upstream** resource as fallback. When the primary resource runs out, it forwards to the upstream. You can read this as a priority chain - try the fast resource first, fall back to the general-purpose one only when necessary:

```cpp
monotonic_buffer_resource(buffer, size, upstream)
         | (overflow)
         v
    upstream resource (e.g., new_delete_resource)
```

This creates a **chain**: try the primary allocator first, fall back to the upstream on overflow. You can make this chain as long as you want, and you can put `null_memory_resource()` at the end if you want to guarantee that overflow is an error rather than a silent heap allocation.

### Standard Memory Resources

| Resource | Behavior |
| --- | --- |
| `new_delete_resource()` | Uses `::operator new` / `::operator delete` |
| `null_memory_resource()` | Always throws `bad_alloc` (good for testing) |
| `monotonic_buffer_resource` | Fast bump-pointer, no individual dealloc |
| `synchronized_pool_resource` | Thread-safe pool allocator |
| `unsynchronized_pool_resource` | Single-threaded pool allocator |

### Why Virtual Dispatch Instead of Templates

The template allocator model (the classic `std::allocator<T>`) bakes the allocator into the container type. Two `vector<int>` with different allocators are literally different types and cannot be assigned to each other. PMR avoids this by using virtual dispatch - all `pmr::vector<int>` are the same type, regardless of which `memory_resource` backs them.

| Feature | Template Allocator (`std::allocator<T>`) | PMR (`memory_resource`) |
| --- | --- | --- |
| Part of container type | Yes | No |
| Runtime swappable | No | Yes |
| Can store in same vector | No (`vector<int, A>` != `vector<int, B>`) | Yes |
| Overhead | Zero (inlined) | Virtual call per alloc |
| ABI stable | No (different types) | Yes |

---

## Self-Assessment

### Q1: Build a fallback chain: try stack buffer first, fall back to `new_delete_resource` on overflow

Watch how all the containers in the second half of this example use the same `memory_resource*` parameter but the backing memory differs. That is the whole point - same container type, runtime-selectable storage.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <array>
#include <string>

int main() {
    // Stack buffer - small, fast, no heap allocation
    std::array<std::byte, 256> stack_buffer;

    // Chain: monotonic uses stack_buffer first,
    //        falls back to new_delete_resource on overflow
    std::pmr::monotonic_buffer_resource pool(
        stack_buffer.data(), stack_buffer.size(),
        std::pmr::new_delete_resource()  // upstream fallback
    );

    std::cout << "=== Fallback chain demo ===\n";
    std::cout << "Stack buffer: " << stack_buffer.size() << " bytes\n\n";

    // Use the pool with a PMR vector
    std::pmr::vector<int> v(&pool);

    // These fit in the stack buffer
    std::cout << "Adding 10 ints (40 bytes)...\n";
    for (int i = 0; i < 10; ++i) v.push_back(i);
    std::cout << "Size: " << v.size() << ", capacity: " << v.capacity() << "\n";

    // Keep adding until we overflow the stack buffer
    std::cout << "\nAdding 100 more ints (overflows stack buffer)...\n";
    for (int i = 10; i < 110; ++i) v.push_back(i);
    std::cout << "Size: " << v.size() << ", capacity: " << v.capacity() << "\n";
    // Overflow - fallback to new_delete_resource (heap allocation)

    // Multiple containers sharing the same chain
    std::cout << "\n=== Multiple containers, one chain ===\n";

    // New buffer for this demo
    std::array<std::byte, 1024> bigger_buffer;
    std::pmr::monotonic_buffer_resource pool2(
        bigger_buffer.data(), bigger_buffer.size(),
        std::pmr::new_delete_resource()
    );

    std::pmr::vector<int> ints(&pool2);
    std::pmr::vector<double> doubles(&pool2);
    std::pmr::string str(&pool2);

    ints.push_back(42);
    doubles.push_back(3.14);
    str = "Hello from stack buffer!";

    std::cout << "int: " << ints[0] << "\n";
    std::cout << "double: " << doubles[0] << "\n";
    std::cout << "string: " << str << "\n";

    // With null_memory_resource: CRASH if stack buffer overflows
    std::cout << "\n=== null_memory_resource (no fallback) ===\n";
    std::array<std::byte, 32> tiny_buffer;
    std::pmr::monotonic_buffer_resource strict_pool(
        tiny_buffer.data(), tiny_buffer.size(),
        std::pmr::null_memory_resource()  // throws on overflow!
    );

    try {
        std::pmr::vector<int> small_vec(&strict_pool);
        for (int i = 0; i < 100; ++i) small_vec.push_back(i);  // will overflow
    } catch (const std::bad_alloc& e) {
        std::cout << "Caught overflow: " << e.what() << "\n";
        std::cout << "(null_memory_resource threw - no fallback allowed)\n";
    }

    return 0;
}
```

The `null_memory_resource` pattern at the end is a useful testing technique: put it as the final fallback and your tests will immediately tell you if any allocation slipped past your intended budget.

### Q2: Implement a custom `memory_resource` that counts total bytes allocated and deallocated

Implementing `memory_resource` is simpler than most people expect - three virtual functions, and you delegate the real work to an upstream. This makes it easy to add instrumentation as a layer in the chain rather than replacing the allocator entirely.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <string>

class CountingResource : public std::pmr::memory_resource {
    std::pmr::memory_resource* upstream_;
    size_t bytes_allocated_ = 0;
    size_t bytes_deallocated_ = 0;
    size_t num_allocs_ = 0;
    size_t num_deallocs_ = 0;
    size_t peak_usage_ = 0;
    size_t current_usage_ = 0;

protected:
    void* do_allocate(size_t bytes, size_t alignment) override {
        bytes_allocated_ += bytes;
        ++num_allocs_;
        current_usage_ += bytes;
        if (current_usage_ > peak_usage_) peak_usage_ = current_usage_;

        std::cout << "  [+alloc] " << bytes << " bytes (align " << alignment
                  << ") - current: " << current_usage_ << "\n";

        return upstream_->allocate(bytes, alignment);
    }

    void do_deallocate(void* p, size_t bytes, size_t alignment) override {
        bytes_deallocated_ += bytes;
        ++num_deallocs_;
        current_usage_ -= bytes;

        std::cout << "  [-dealloc] " << bytes << " bytes - current: " << current_usage_ << "\n";

        upstream_->deallocate(p, bytes, alignment);
    }

    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }

public:
    explicit CountingResource(
        std::pmr::memory_resource* upstream = std::pmr::get_default_resource())
        : upstream_(upstream) {}

    void print_stats() const {
        std::cout << "\n=== Allocation Statistics ===\n";
        std::cout << "  Total allocated:   " << bytes_allocated_ << " bytes (" << num_allocs_ << " calls)\n";
        std::cout << "  Total deallocated: " << bytes_deallocated_ << " bytes (" << num_deallocs_ << " calls)\n";
        std::cout << "  Currently in use:  " << current_usage_ << " bytes\n";
        std::cout << "  Peak usage:        " << peak_usage_ << " bytes\n";
        std::cout << "  Leaked:            " << (bytes_allocated_ - bytes_deallocated_) << " bytes\n";
    }
};

int main() {
    CountingResource counter;

    std::cout << "--- Vector operations ---\n";
    {
        std::pmr::vector<int> v(&counter);
        v.push_back(1);
        v.push_back(2);
        v.push_back(3);
        v.push_back(4);
        v.push_back(5);
        std::cout << "Vector size: " << v.size() << "\n";
    }   // vector destroyed - deallocates

    counter.print_stats();

    std::cout << "\n--- String operations ---\n";
    {
        std::pmr::string s("A long string that definitely allocates on the heap", &counter);
        s += " - and grows even more!";
        std::cout << "String length: " << s.length() << "\n";
    }

    counter.print_stats();

    return 0;
}
```

The stats printout shows reallocation clearly: a vector that grows by push-back will allocate, then allocate a larger block, then deallocate the old one. You can see the doubling strategy in the numbers.

### Q3: Explain why `memory_resource` uses virtual dispatch while `std::allocator` uses templates

This example shows the concrete consequence of the design difference: with template allocators, two containers using different allocators cannot go into the same collection. With PMR, they can because `pmr::vector<int>` is always the same type.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <array>
#include <cstddef>
#include <memory>

// The fundamental difference:

// Template approach: allocator is part of the TYPE
// std::vector<int, std::allocator<int>>  !=  std::vector<int, MyAllocator<int>>
// These are different types! Can't store in same container, can't pass to same function.

// Virtual dispatch approach: allocator is a RUNTIME parameter
// std::pmr::vector<int> is always the same type
// Different memory_resources can be swapped at runtime

void process_vector(const std::pmr::vector<int>& v) {
    // This function works with ANY memory_resource
    // No template needed, no recompilation needed
    std::cout << "Processing " << v.size() << " ints\n";
}

int main() {
    std::cout << "=== Template allocator (different TYPES) ===\n";
    // These are different types:
    // std::vector<int, std::allocator<int>> v1;
    // std::vector<int, MyAlloc<int>> v2;
    // v1 = v2;  // ERROR: different types

    std::cout << "=== PMR (same TYPE, different allocators) ===\n";

    // All are std::pmr::vector<int> - same type!
    std::array<std::byte, 1024> buf;
    std::pmr::monotonic_buffer_resource mono(buf.data(), buf.size());

    std::pmr::vector<int> v1(std::pmr::get_default_resource());  // default heap
    std::pmr::vector<int> v2(&mono);  // stack buffer

    v1.push_back(1);
    v2.push_back(2);

    // Both can be passed to the SAME function
    process_vector(v1);  // using default (heap)
    process_vector(v2);  // using monotonic (stack)

    // Can be stored in the same container
    std::vector<std::pmr::vector<int>*> all = {&v1, &v2};
    std::cout << "Stored " << all.size() << " pmr::vectors in one container\n";

    std::cout << "\n=== Trade-offs ===\n";
    std::cout << "Template allocator:\n";
    std::cout << "  + Zero overhead (inlined, no virtual calls)\n";
    std::cout << "  + Full optimization by compiler\n";
    std::cout << "  - Different type per allocator (template bloat)\n";
    std::cout << "  - Can't swap at runtime\n";
    std::cout << "  - Hard to use across ABI boundaries\n";

    std::cout << "\nPMR memory_resource:\n";
    std::cout << "  + Same type regardless of resource\n";
    std::cout << "  + Runtime swappable\n";
    std::cout << "  + ABI stable\n";
    std::cout << "  + Easy to chain/compose resources\n";
    std::cout << "  - Virtual call overhead per allocation\n";
    std::cout << "  - Slightly harder to optimize\n";

    return 0;
}
```

The virtual call overhead is typically negligible compared to the cost of the allocation itself, so in most real code the PMR trade-off is a clear win when you need runtime flexibility.

---

## Notes

- The `memory_resource` interface has only 3 virtual functions - simple to implement.
- Chain resources with upstream parameters: fast primary - slower fallback.
- Use `null_memory_resource()` as the final fallback to guarantee no silent heap allocation.
- `monotonic_buffer_resource` is the fastest - good for short-lived, batch-style allocation patterns.
- PMR containers propagate their allocator to nested containers automatically (unlike classic allocator).
