# Design memory management architecture - arenas, pools, and PMR

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

Custom memory management replaces `new`/`delete` with allocators optimized for specific access patterns. **Arena allocators** batch-free all memory at once, **pool allocators** recycle fixed-size blocks, and **PMR** (polymorphic memory resources, C++17) provides a standard interface for pluggable allocators. These techniques reduce fragmentation, improve cache locality, and eliminate per-object allocation overhead.

The reason this matters in practice: `malloc` and `free` are general-purpose, which means they handle every imaginable allocation pattern - and that generality has a cost. If you know that all your allocations within a game frame will be thrown away together, or that your object pool always dispenses objects of the same size, you can beat `malloc` by a large margin with far simpler code.

### Allocator Comparison

| Allocator | Allocation | Deallocation | Fragmentation | Use Case |
| --- | --- | --- | --- | --- |
| **malloc/new** | General | Individual | Over time, yes | Default |
| **Arena (bump)** | O(1) bump pointer | All at once (reset) | None | Per-frame, per-request |
| **Pool (freelist)** | O(1) from freelist | O(1) return to list | None (fixed-size) | Particles, nodes |
| **PMR** | Pluggable | Pluggable | Depends on resource | Standard containers |
| **Stack allocator** | O(1) bump | LIFO only | None | Temporary buffers |

---

## Self-Assessment

### Q1: Implement an arena allocator

An arena allocator is beautifully simple: it owns a big block of memory and hands it out by bumping a pointer forward. Allocation is just a pointer increment and an alignment adjustment - there's no bookkeeping per object. The trade-off is that you can't free individual objects; you free everything at once by resetting the pointer back to the start. This is perfect for per-frame game data, per-request server allocations, or any situation where a group of objects all die together.

The alignment logic here is worth understanding: we round up the current pointer to the next multiple of `alignment` using the bitmask trick `(ptr + align - 1) & ~(align - 1)`. If the current block doesn't have enough room, we allocate a new block and retry.

**Answer:**

```cpp
#include <cstddef>
#include <cstdint>
#include <vector>
#include <cassert>
#include <memory>

// === Arena: fast bump allocation, bulk deallocation ===
class Arena {
public:
    explicit Arena(size_t block_size = 1024 * 1024)  // 1MB default
        : block_size_(block_size) {
        allocate_block();
    }

    // O(1) allocation: just bump a pointer
    void* allocate(size_t size, size_t alignment = alignof(std::max_align_t)) {
        // Align the current pointer
        uintptr_t current = reinterpret_cast<uintptr_t>(ptr_);
        uintptr_t aligned = (current + alignment - 1) & ~(alignment - 1);
        size_t padding = aligned - current;

        if (ptr_ + padding + size > end_) {
            allocate_block(std::max(block_size_, size + alignment));
            return allocate(size, alignment);  // Retry in new block
        }

        ptr_ += padding;
        void* result = ptr_;
        ptr_ += size;
        return result;
    }

    // Typed allocation
    template<typename T, typename... Args>
    T* create(Args&&... args) {
        void* mem = allocate(sizeof(T), alignof(T));
        return new (mem) T(std::forward<Args>(args)...);
    }

    // Reset: free ALL allocations at once (O(1))
    void reset() {
        for (size_t i = 1; i < blocks_.size(); ++i)
            operator delete(blocks_[i].data);
        blocks_.resize(1);
        ptr_ = static_cast<char*>(blocks_[0].data);
        end_ = ptr_ + blocks_[0].size;
    }

    ~Arena() {
        for (auto& b : blocks_)
            operator delete(b.data);
    }

    size_t bytes_used() const { return total_allocated_ - (end_ - ptr_); }

private:
    void allocate_block(size_t size = 0) {
        if (size == 0) size = block_size_;
        void* data = operator new(size);
        blocks_.push_back({data, size});
        ptr_ = static_cast<char*>(data);
        end_ = ptr_ + size;
        total_allocated_ += size;
    }

    struct Block { void* data; size_t size; };
    std::vector<Block> blocks_;
    char* ptr_ = nullptr;
    char* end_ = nullptr;
    size_t block_size_;
    size_t total_allocated_ = 0;
};

// === Usage: per-frame allocation ===
void game_frame(Arena& frame_arena) {
    frame_arena.reset();  // Free everything from last frame

    // All frame allocations are O(1) bump pointer
    auto* particles = frame_arena.create<ParticleSystem>(1000);
    auto* temp_buffer = static_cast<float*>(
        frame_arena.allocate(sizeof(float) * 4096));
    auto* debug_text = frame_arena.create<std::string>("FPS: 60");

    // ... process frame ...
    // No individual frees needed; reset() at start of next frame
}
```

Notice that `reset()` keeps the first block and releases any overflow blocks. That means in the steady state (where each frame's allocation fits in one block) you have zero heap traffic after the first frame. Also note: `create<T>()` uses placement new to construct the object in the arena memory, but the arena's `reset()` does *not* call destructors - if your objects have non-trivial destructors, you need to call them manually or use a different approach.

### Q2: Implement a pool allocator for fixed-size objects

A pool allocator is the right tool when you're constantly creating and destroying many objects of the same type - think particles, tree nodes, or network connections. It pre-allocates a contiguous slab of memory and builds a free list by reusing each free slot's storage to hold a pointer to the next free slot. Allocation pops from the free list, deallocation pushes back onto it - both O(1), no fragmentation, and the contiguous storage is cache-friendly.

The `static_assert` that `sizeof(T) >= sizeof(FreeNode)` is important: we're storing a `FreeNode*` in the same memory that will later hold a `T`. If `T` is smaller than a pointer, this scheme doesn't work.

**Answer:**

```cpp
// === Pool allocator: recycling fixed-size blocks ===
template<typename T>
class Pool {
public:
    explicit Pool(size_t capacity) : capacity_(capacity) {
        // Allocate contiguous memory for all objects
        storage_ = static_cast<char*>(
            operator new(sizeof(T) * capacity));

        // Build freelist
        for (size_t i = 0; i < capacity; ++i) {
            auto* node = reinterpret_cast<FreeNode*>(
                storage_ + i * sizeof(T));
            node->next = free_;
            free_ = node;
        }
    }

    ~Pool() {
        operator delete(storage_);
    }

    // O(1) allocation from freelist
    template<typename... Args>
    T* allocate(Args&&... args) {
        if (!free_) return nullptr;  // Pool exhausted

        FreeNode* node = free_;
        free_ = node->next;
        ++in_use_;

        // Construct in-place
        return new (node) T(std::forward<Args>(args)...);
    }

    // O(1) deallocation: return to freelist
    void deallocate(T* obj) {
        obj->~T();  // Destroy
        auto* node = reinterpret_cast<FreeNode*>(obj);
        node->next = free_;
        free_ = node;
        --in_use_;
    }

    size_t in_use() const { return in_use_; }
    size_t capacity() const { return capacity_; }
    bool full() const { return in_use_ >= capacity_; }

private:
    struct FreeNode { FreeNode* next; };
    static_assert(sizeof(T) >= sizeof(FreeNode),
        "T must be at least pointer-sized");

    char* storage_ = nullptr;
    FreeNode* free_ = nullptr;
    size_t capacity_;
    size_t in_use_ = 0;
};

// === Usage ===
Pool<Particle> particle_pool(10000);

void spawn_particle(float x, float y) {
    if (auto* p = particle_pool.allocate(x, y, /*velocity*/0.0f)) {
        active_particles.push_back(p);
    }
}

void kill_particle(Particle* p) {
    particle_pool.deallocate(p);  // Recycles memory
}
```

The key insight in `deallocate` is that we call `obj->~T()` explicitly (destroy the object) but then immediately reinterpret that memory as a `FreeNode` to thread it back into the free list. The memory is *reused*, not freed back to the OS. This is perfectly valid C++ as long as you don't access the object after destruction.

### Q3: Use PMR for pluggable allocators with standard containers

PMR (polymorphic memory resources) is the C++17 answer to a long-standing problem: how do you use `std::vector` and `std::string` with a custom allocator without templating everything in sight? The answer is a polymorphic base class `std::pmr::memory_resource` with virtual `do_allocate` and `do_deallocate`, and `std::pmr::vector<T>` is just `std::vector<T, std::pmr::polymorphic_allocator<T>>`. You pass the resource at construction time, and the container uses it for all its allocations.

The real payoff is `std::pmr::monotonic_buffer_resource`: back it with a stack array and you get zero heap allocations for your entire container, which is extremely useful for short-lived per-request processing.

**Answer:**

```cpp
#include <memory_resource>
#include <vector>
#include <string>

// === PMR: use standard containers with custom allocators ===
void pmr_example() {
    // Stack-based buffer: no heap allocation
    char buffer[4096];
    std::pmr::monotonic_buffer_resource arena(
        buffer, sizeof(buffer), std::pmr::null_memory_resource());

    // Containers use arena for allocation
    std::pmr::vector<int> numbers(&arena);
    numbers.reserve(100);
    for (int i = 0; i < 100; ++i)
        numbers.push_back(i);  // All in stack buffer

    std::pmr::string name("Hello World", &arena);

    // Nested containers: all use the same arena
    std::pmr::vector<std::pmr::string> names(&arena);
    names.emplace_back("Alice");
    names.emplace_back("Bob");
}

// === PMR-based pool for production ===
class RequestAllocator {
public:
    explicit RequestAllocator(size_t size = 64 * 1024)
        : buffer_(size),
          arena_(buffer_.data(), buffer_.size(),
                 std::pmr::new_delete_resource()) {}

    std::pmr::memory_resource* resource() { return &arena_; }

    // Reset between requests
    void reset() {
        arena_.release();
    }

private:
    std::vector<char> buffer_;
    std::pmr::monotonic_buffer_resource arena_;
};

void handle_request(RequestAllocator& alloc) {
    alloc.reset();  // Reuse buffer from previous request

    auto* res = alloc.resource();
    std::pmr::vector<std::pmr::string> headers(res);
    std::pmr::vector<uint8_t> body(res);

    // All allocations from pre-allocated buffer
    // No malloc() calls during request processing
    headers.emplace_back("Content-Type: text/html");
    body.assign(response_data.begin(), response_data.end());
}

// === Custom PMR resource wrapping our Arena ===
class ArenaPmrResource : public std::pmr::memory_resource {
public:
    explicit ArenaPmrResource(Arena& arena) : arena_(arena) {}

protected:
    void* do_allocate(size_t bytes, size_t alignment) override {
        return arena_.allocate(bytes, alignment);
    }
    void do_deallocate(void*, size_t, size_t) override {
        // Arena doesn't support individual deallocation
    }
    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }

private:
    Arena& arena_;
};
```

`ArenaPmrResource` is the bridge between the custom `Arena` from Q1 and the PMR ecosystem - wrap your arena in this adapter and suddenly `std::pmr::vector`, `std::pmr::string`, and `std::pmr::map` all work with it, no template surgery required. The `do_deallocate` no-op is intentional: the arena doesn't free individual slots, it frees everything on `reset()`.

---

## Notes

- **Arena allocator** is the most impactful optimization: O(1) allocate, O(1) reset, zero fragmentation.
- Use arenas for per-frame (games), per-request (servers), per-task (batch processing) lifetimes.
- **Pool allocator** is ideal for many same-sized objects: particles, tree nodes, connections.
- **PMR** (`std::pmr`) lets you use standard containers (`vector`, `string`, `map`) with custom allocators.
- `monotonic_buffer_resource` + stack buffer = zero heap allocations for small requests.
- Never use arenas for objects with complex destructors unless you explicitly call them.
- Profile first: custom allocators add complexity; only use them where `malloc` is a measured bottleneck.
