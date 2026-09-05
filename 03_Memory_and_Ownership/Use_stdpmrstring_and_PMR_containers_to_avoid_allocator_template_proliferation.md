# Use std::pmr::string and PMR Containers to Avoid Allocator Template Proliferation

**Category:** Memory and Ownership  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/memory/polymorphic_allocator>  

---

## Topic Overview

### The Problem: Allocator Template Proliferation

With traditional allocators, each allocator type creates a **different container type**. That sounds like a minor detail, but it means you cannot pass one to a function expecting the other:

```cpp
std::vector<int, MyAlloc<int>>   vec1;  // Type A
std::vector<int, PoolAlloc<int>> vec2;  // Type B - DIFFERENT type!
// vec1 = vec2;  // ERROR: different types!
```

This means functions that accept `std::vector<int>` can't accept `std::vector<int, CustomAlloc<int>>`.

### PMR Solution: Type-Erased Allocators

`std::pmr` containers use `std::pmr::polymorphic_allocator`, which type-erases the allocator. The allocator strategy becomes a runtime parameter instead of a compile-time type parameter, so all PMR containers of the same element type are the same type regardless of which memory resource backs them:

```cpp
std::pmr::vector<int> vec1(&arena);     // Uses arena
std::pmr::vector<int> vec2(&pool);      // Uses pool
// SAME TYPE! Can be assigned, passed to same function
```

If the table below feels like a lot, the one-line summary is: with PMR, the allocator is a runtime detail, not part of the type.

| Traditional | PMR |
| --- | --- |
| `std::vector<int, A>` != `std::vector<int, B>` | `std::pmr::vector<int>` - always same type |
| Allocator is part of TYPE | Allocator is runtime parameter |
| Every allocator = new template instantiation | One instantiation, any resource |

---

## Self-Assessment

### Q1: Allocate a `std::pmr::string` from a monotonic buffer and verify no heap allocation occurs

This example overrides `operator new` to count heap allocations, then shows that strings backed by a PMR arena never touch the heap. It also compares against regular `std::string` so the difference is concrete.

```cpp
#include <iostream>
#include <memory_resource>
#include <string>
#include <cstdlib>

// Track heap allocations
static int heap_allocs = 0;
void* operator new(size_t size) {
    ++heap_allocs;
    return std::malloc(size);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, size_t) noexcept { std::free(p); }

int main() {
    std::cout << "=== PMR string with monotonic buffer ===\n\n";

    // Stack buffer - no heap involved
    char buffer[256];
    std::pmr::monotonic_buffer_resource arena(buffer, sizeof(buffer));

    heap_allocs = 0;

    // PMR string uses the arena, not the heap
    std::pmr::string s1("Hello, PMR!", &arena);
    std::pmr::string s2("This is allocated from a stack buffer", &arena);
    std::pmr::string s3("No heap allocation whatsoever", &arena);

    std::cout << "s1: " << s1 << "\n";
    std::cout << "s2: " << s2 << "\n";
    std::cout << "s3: " << s3 << "\n";
    std::cout << "\nHeap allocations: " << heap_allocs << "\n";

    // Compare with regular std::string
    heap_allocs = 0;
    {
        std::string r1 = "Hello, regular string!";
        std::string r2 = "This uses the default allocator (heap)";
        std::string r3 = "Each long string triggers operator new";
    }
    std::cout << "Regular string heap allocs: " << heap_allocs << "\n";

    return 0;
}
// Expected output (varies by SSO threshold):
// s1: Hello, PMR!
// s2: This is allocated from a stack buffer
// s3: No heap allocation whatsoever
//
// Heap allocations: 0
// Regular string heap allocs: 2-3 (long strings that exceed SSO)
```

The PMR strings serve their allocations out of `buffer` on the stack - `operator new` is never called. The regular strings that exceed the small-string optimization threshold each trigger a heap allocation.

### Q2: Explain why `std::pmr::vector<T>` and `std::vector<T>` have the same template type for element type

The key insight here is that two `pmr::vector<int>` objects are the same type even when they use completely different memory resources at runtime. That is what makes it possible to pass them both to the same function - something traditional allocators make impossible.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <string>
#include <typeinfo>

// The key insight:
// std::pmr::vector<int> = std::vector<int, std::pmr::polymorphic_allocator<int>>
//
// ALL pmr::vectors of int are the SAME type, regardless of which
// memory_resource they use at runtime.

void process(const std::pmr::vector<int>& data) {
    int sum = 0;
    for (int x : data) sum += x;
    std::cout << "  Sum of " << data.size() << " elements: " << sum << "\n";
}

int main() {
    std::cout << "=== Type identity ===\n\n";

    // Different memory resources
    char buf1[1024], buf2[1024];
    std::pmr::monotonic_buffer_resource arena1(buf1, sizeof(buf1));
    std::pmr::monotonic_buffer_resource arena2(buf2, sizeof(buf2));

    // Same type despite different allocators!
    std::pmr::vector<int> v1({1, 2, 3}, &arena1);
    std::pmr::vector<int> v2({10, 20, 30, 40}, &arena2);

    std::cout << "v1 and v2 are same type: "
              << (typeid(v1) == typeid(v2) ? "YES" : "NO") << "\n\n";

    // Both can be passed to the same function
    process(v1);
    process(v2);

    // Traditional allocators: different types!
    // std::vector<int, AllocA<int>> a;
    // std::vector<int, AllocB<int>> b;
    // process(a);  // ERROR: can't convert AllocA vector to std::vector<int>

    std::cout << "\n=== PMR container aliases ===\n";
    std::cout << "pmr::vector<T>           = vector<T, pmr::polymorphic_allocator<T>>\n";
    std::cout << "pmr::string              = basic_string<char, ..., pmr::polymorphic_allocator<char>>\n";
    std::cout << "pmr::map<K,V>            = map<K, V, less<K>, pmr::polymorphic_allocator<pair<const K,V>>>\n";
    std::cout << "pmr::unordered_map<K,V>  = ...same pattern...\n\n";

    std::cout << "Note: pmr::vector<int> and std::vector<int> are DIFFERENT types.\n";
    std::cout << "The 'same type' benefit is between PMR containers using different resources.\n";

    return 0;
}
```

Notice that `v1` and `v2` backed by different arenas are still the same type and both pass into `process` without any overload or conversion. This is the whole value proposition of PMR.

### Q3: Show how polymorphic allocators allow runtime selection of allocation strategy

Here the same `build_data` function is called three times with three different memory resources. Because PMR type-erases the allocator, the function signature doesn't change - the strategy is a runtime decision.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <string>

// The allocation strategy is determined at RUNTIME, not compile time

std::pmr::vector<std::pmr::string> build_data(
    std::pmr::memory_resource* alloc, int n)
{
    std::pmr::vector<std::pmr::string> result(alloc);
    for (int i = 0; i < n; ++i) {
        result.emplace_back("item_" + std::to_string(i), alloc);
    }
    return result;
}

int main() {
    std::cout << "=== Runtime allocator selection ===\n\n";

    // Strategy 1: Arena for fast, temporary processing
    {
        char buf[4096];
        std::pmr::monotonic_buffer_resource arena(buf, sizeof(buf));
        std::cout << "Using monotonic arena:\n";
        auto data = build_data(&arena, 5);
        for (const auto& s : data) std::cout << "  " << s << "\n";
    }

    // Strategy 2: Pool for frequent alloc/dealloc
    {
        std::pmr::unsynchronized_pool_resource pool;
        std::cout << "\nUsing pool resource:\n";
        auto data = build_data(&pool, 5);
        for (const auto& s : data) std::cout << "  " << s << "\n";
    }

    // Strategy 3: Default (heap)
    {
        std::cout << "\nUsing default (heap):\n";
        auto data = build_data(std::pmr::new_delete_resource(), 5);
        for (const auto& s : data) std::cout << "  " << s << "\n";
    }

    // Strategy chosen at runtime based on conditions
    auto choose_allocator = [](bool is_hot_path) -> std::pmr::memory_resource* {
        static char fast_buf[8192];
        static std::pmr::monotonic_buffer_resource fast_arena(fast_buf, sizeof(fast_buf));

        if (is_hot_path) return &fast_arena;   // Fast path: arena
        return std::pmr::new_delete_resource(); // Normal: heap
    };

    std::cout << "\nRuntime strategy selection:\n";
    auto data = build_data(choose_allocator(true), 3);
    for (const auto& s : data) std::cout << "  " << s << "\n";

    return 0;
}
```

You can swap the memory resource based on a flag, a config value, or profiling data - `build_data` is completely unaware of which backing resource it gets.

---

## Notes

- `std::pmr::` aliases provide type-erased allocator containers - same type regardless of allocator strategy.
- `polymorphic_allocator` uses virtual dispatch (`memory_resource*`), not template parameters.
- PMR containers propagate their allocator to nested containers (e.g., `pmr::vector<pmr::string>`).
- Slight runtime cost vs static allocators (virtual function call per allocation), but eliminates template bloat.
- Use `pmr` when you need runtime allocator selection or want to avoid `N x M` template instantiations.
