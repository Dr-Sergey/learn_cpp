# Design allocator-aware containers and types

**Category:** API and Library Design  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/named_req/AllocatorAwareContainer>  

---

## Topic Overview

Allocator-aware types let users control exactly where and how memory is allocated, without having to change the container code itself. This sounds like an advanced niche feature, but it matters quite a bit in practice: arena allocation for game engines, PMR (polymorphic memory resources) for controlling heap fragmentation, and embedded systems where you might not have a general-purpose heap at all.

The idea is that your container takes an allocator as a template parameter (or via PMR, at runtime). Instead of calling `new` and `delete` directly, it goes through `std::allocator_traits`, which dispatches to whatever allocator the user plugged in.

### Making a Type Allocator-Aware

Here is a minimal but complete allocator-aware buffer. The key parts are the `allocator_type` typedef, the allocator constructor, going through `std::allocator_traits` for every allocation/construction/destruction operation, and the `get_allocator()` accessor. The `[[no_unique_address]]` on the allocator member is a C++20 size optimization covered in the Q&A below.

```cpp
#include <memory>
#include <memory_resource>
#include <vector>
#include <string>
#include <iostream>
#include <array>
#include <cstddef>

template<typename T, typename Allocator = std::allocator<T>>
class SimpleBuffer {
public:
    using allocator_type = Allocator;
    using value_type = T;

    SimpleBuffer() = default;

    explicit SimpleBuffer(const Allocator& alloc)
        : alloc_(alloc) {}

    SimpleBuffer(size_t count, const T& value, const Allocator& alloc = Allocator())
        : alloc_(alloc) {
        data_ = std::allocator_traits<Allocator>::allocate(alloc_, count);
        size_ = count;
        for (size_t i = 0; i < count; ++i)
            std::allocator_traits<Allocator>::construct(alloc_, data_ + i, value);
    }

    ~SimpleBuffer() {
        for (size_t i = 0; i < size_; ++i)
            std::allocator_traits<Allocator>::destroy(alloc_, data_ + i);
        if (data_)
            std::allocator_traits<Allocator>::deallocate(alloc_, data_, size_);
    }

    allocator_type get_allocator() const { return alloc_; }
    size_t size() const { return size_; }
    T& operator[](size_t i) { return data_[i]; }

private:
    T* data_ = nullptr;
    size_t size_ = 0;
    [[no_unique_address]] Allocator alloc_{};
};

int main() {
    // With default allocator
    SimpleBuffer<int> buf1(5, 42);
    std::cout << "Default: " << buf1[0] << "\n";

    // With PMR allocator
    std::array<std::byte, 1024> storage;
    std::pmr::monotonic_buffer_resource resource(storage.data(), storage.size());
    SimpleBuffer<int, std::pmr::polymorphic_allocator<int>> buf2(
        5, 99, std::pmr::polymorphic_allocator<int>(&resource));
    std::cout << "PMR: " << buf2[0] << "\n";
}
```

Notice that `buf2` allocates entirely from the `storage` array on the stack - no heap involved. That is the payoff. The container code did not change at all; only the allocator type changed.

---

## Self-Assessment

### Q1: Why use `std::allocator_traits` instead of calling allocator methods directly

The reason is that `allocator_traits` provides default implementations for all the optional allocator methods. A custom allocator only needs to implement `allocate()` and `deallocate()`. The traits layer fills in `construct`, `destroy`, `max_size`, `select_on_container_copy_construction`, and other operations with sensible defaults. If you call allocator methods directly, your container code breaks as soon as someone passes a minimal custom allocator that only has the two required methods.

### Q2: What is `[[no_unique_address]]` and why use it for allocators

The standard allocator (`std::allocator`) is stateless - it has no data members, but C++ rules say every object must have a unique address, so `sizeof(std::allocator<int>)` is 1. Without `[[no_unique_address]]`, embedding one of these inside a container wastes padding bytes in every instance. With the attribute (C++20), the compiler is allowed to overlap the allocator's storage with another member, effectively making it zero-cost. This can save 8 bytes per container on 64-bit platforms, which adds up when you have millions of small containers.

### Q3: Show how PMR containers use a runtime-polymorphic allocator

The PMR approach is particularly elegant because all PMR containers share the same type (`std::pmr::vector<int>` is always the same type) even though they may use completely different memory resources at runtime. You do not get template proliferation. Here is the standard pattern:

```cpp
#include <memory_resource>
#include <vector>
#include <string>

void pmr_example() {
    std::byte buf[4096];
    std::pmr::monotonic_buffer_resource arena(buf, sizeof(buf));

    // All these use the arena - same allocator TYPE, different resource
    std::pmr::vector<int> nums(&arena);
    std::pmr::string name("hello", &arena);

    nums.push_back(42);  // Allocates from buf[], not heap
}
```

When `pmr_example` returns, the stack-allocated `buf` goes away and all the memory is freed instantly - no individual `delete` calls, no fragmentation. This is the arena allocation pattern that games and real-time systems rely on heavily.

---

## Notes

- Always use `std::allocator_traits` - never call allocator methods directly. This is the rule that keeps your container compatible with any conforming allocator.
- `[[no_unique_address]]` (C++20) enables empty base optimization for allocators stored as members rather than base classes.
- PMR (`std::pmr`) provides runtime allocator polymorphism without template proliferation - all `std::pmr::vector<T>` are the same type regardless of which resource backs them.
- Allocator-aware types should propagate the allocator to any sub-containers they own, so that memory stays in the same arena throughout the object graph.
