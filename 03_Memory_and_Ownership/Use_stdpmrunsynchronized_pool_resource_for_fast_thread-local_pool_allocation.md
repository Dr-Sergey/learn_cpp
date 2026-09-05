# Use std::pmr::unsynchronized_pool_resource for Fast Thread-Local Pool Allocation

**Category:** Memory and Ownership  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/memory/unsynchronized_pool_resource>  

---

## Topic Overview

### What Is `unsynchronized_pool_resource`

A PMR memory resource that organizes memory into pools of different block sizes. It is **not thread-safe** (no synchronization overhead), making it the fastest PMR pool option for single-threaded or thread-local use. The speed comes precisely from skipping the locking - that is a deliberate trade-off, not an oversight.

### PMR Pool Comparison

| Resource | Thread-safe | Deallocation | Best for |
| --- | --- | --- | --- |
| `monotonic_buffer_resource` | No | No-op (freed at destruction) | Allocate-only workloads |
| `unsynchronized_pool_resource` | **No** | Yes (recycles blocks) | Single-thread alloc/dealloc |
| `synchronized_pool_resource` | Yes | Yes (recycles blocks) | Multi-threaded shared alloc |

### How Pool Resources Work

The pool resource maintains a set of fixed-size free lists, one per block size. An allocation request goes to the smallest pool that fits the request, and a deallocation returns the block to that same pool for reuse. Large allocations that do not fit any pool fall through to the upstream resource.

```cpp
Pool layout (simplified):
  Pool 1:  [8-byte blocks]   -> used used used free
  Pool 2:  [16-byte blocks]  -> used used free free
  Pool 3:  [32-byte blocks]  -> used free free free
  ...

Allocate(12 bytes) -> Pool 2 (smallest pool >= 12)
Deallocate -> block returned to its pool for reuse
```

---

## Self-Assessment

### Q1: Build a parser allocating many small AST nodes using an `unsynchronized_pool_resource`

The parser here allocates every AST node from a single pool resource. Notice that the pool is a member of `Parser` - each parser instance owns its own pool, which is the right pattern for thread-local or per-task use.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <string>
#include <variant>

// Simple AST node types using PMR allocation
struct ASTNode;

using NodePtr = ASTNode*;

struct NumberLit { double value; };
struct StringLit { std::pmr::string value; };
struct BinaryOp {
    char op;
    NodePtr left;
    NodePtr right;
};

struct ASTNode {
    std::variant<NumberLit, StringLit, BinaryOp> data;

    void print(int indent = 0) const {
        std::string pad(indent, ' ');
        std::visit([&](const auto& d) {
            using T = std::decay_t<decltype(d)>;
            if constexpr (std::is_same_v<T, NumberLit>) {
                std::cout << pad << "Number(" << d.value << ")\n";
            } else if constexpr (std::is_same_v<T, StringLit>) {
                std::cout << pad << "String(\"" << d.value << "\")\n";
            } else if constexpr (std::is_same_v<T, BinaryOp>) {
                std::cout << pad << "BinaryOp '" << d.op << "'\n";
                if (d.left) d.left->print(indent + 2);
                if (d.right) d.right->print(indent + 2);
            }
        }, data);
    }
};

class Parser {
    std::pmr::unsynchronized_pool_resource pool_;
    std::pmr::vector<ASTNode*> nodes_;  // Track for cleanup

public:
    Parser() : nodes_(&pool_) {}

    ASTNode* make_number(double v) {
        void* mem = pool_.allocate(sizeof(ASTNode), alignof(ASTNode));
        auto* node = new (mem) ASTNode{NumberLit{v}};
        nodes_.push_back(node);
        return node;
    }

    ASTNode* make_string(const char* s) {
        void* mem = pool_.allocate(sizeof(ASTNode), alignof(ASTNode));
        auto* node = new (mem) ASTNode{StringLit{std::pmr::string(s, &pool_)}};
        nodes_.push_back(node);
        return node;
    }

    ASTNode* make_binop(char op, ASTNode* l, ASTNode* r) {
        void* mem = pool_.allocate(sizeof(ASTNode), alignof(ASTNode));
        auto* node = new (mem) ASTNode{BinaryOp{op, l, r}};
        nodes_.push_back(node);
        return node;
    }

    ~Parser() {
        for (auto* n : nodes_) {
            n->~ASTNode();
            pool_.deallocate(n, sizeof(ASTNode), alignof(ASTNode));
        }
    }
};

int main() {
    std::cout << "=== AST Parser with unsynchronized_pool_resource ===\n\n";

    Parser parser;

    // Build AST: (3.14 + 2.0) * 10.0
    auto* left = parser.make_binop('+',
        parser.make_number(3.14),
        parser.make_number(2.0));
    auto* root = parser.make_binop('*', left, parser.make_number(10.0));

    root->print();
    // All nodes allocated from the pool - fast, no fragmentation

    return 0;
}
// Expected output:
// BinaryOp '*'
//   BinaryOp '+'
//     Number(3.14)
//     Number(2)
//   Number(10)
```

All node allocations go to the same pool, so they are co-located in memory and the allocator avoids the per-allocation overhead of `malloc`. The destructor manually calls each node's destructor then deallocates back to the pool so the blocks can be reused.

### Q2: Measure allocation throughput vs the default allocator for thousands of small objects

This benchmark runs three strategies side by side: default `new`/`delete`, pool resource with deallocation, and monotonic arena (no deallocation). Run it and you will see pool allocation is significantly faster than the default heap for many small objects of the same size.

```cpp
#include <iostream>
#include <memory_resource>
#include <vector>
#include <chrono>

struct SmallObj {
    int data[4];  // 16 bytes
};

template<typename Func>
long long bench(const char* label, Func f, int iterations) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) f();
    auto end = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    std::cout << label << ": " << us << " us\n";
    return us;
}

int main() {
    constexpr int ROUNDS = 100;
    constexpr int OBJECTS = 10000;

    std::cout << "=== Allocation benchmark: " << OBJECTS << " objects x "
              << ROUNDS << " rounds ===\n\n";

    // Default allocator (malloc/free per object)
    auto t_default = bench("Default (new/delete)", [&] {
        std::vector<SmallObj*> ptrs;
        ptrs.reserve(OBJECTS);
        for (int i = 0; i < OBJECTS; ++i)
            ptrs.push_back(new SmallObj{{i, i, i, i}});
        for (auto* p : ptrs)
            delete p;
    }, ROUNDS);

    // unsynchronized_pool_resource
    auto t_pool = bench("Unsync pool        ", [&] {
        std::pmr::unsynchronized_pool_resource pool;
        std::pmr::vector<SmallObj*> ptrs(&pool);
        ptrs.reserve(OBJECTS);
        for (int i = 0; i < OBJECTS; ++i) {
            void* mem = pool.allocate(sizeof(SmallObj), alignof(SmallObj));
            ptrs.push_back(new (mem) SmallObj{{i, i, i, i}});
        }
        for (auto* p : ptrs)
            pool.deallocate(p, sizeof(SmallObj), alignof(SmallObj));
    }, ROUNDS);

    // monotonic (no deallocation - fastest possible)
    auto t_mono = bench("Monotonic arena    ", [&] {
        std::pmr::monotonic_buffer_resource arena;
        std::pmr::vector<SmallObj*> ptrs(&arena);
        ptrs.reserve(OBJECTS);
        for (int i = 0; i < OBJECTS; ++i) {
            void* mem = arena.allocate(sizeof(SmallObj), alignof(SmallObj));
            ptrs.push_back(new (mem) SmallObj{{i, i, i, i}});
        }
        // No deallocation - arena frees everything at destruction
    }, ROUNDS);

    std::cout << "\nSpeedup vs default:\n";
    std::cout << "  Pool:      " << (double)t_default / (t_pool > 0 ? t_pool : 1) << "x\n";
    std::cout << "  Monotonic: " << (double)t_default / (t_mono > 0 ? t_mono : 1) << "x\n";

    return 0;
}
```

The monotonic arena wins the benchmark because it never needs to track individual deallocations - it just bumps a pointer. The pool comes second because it does support deallocation but avoids the per-object `malloc` overhead.

### Q3: Explain why `unsynchronized_pool_resource` must not be shared between threads

The reason this trips people up is that "no synchronization" sounds like a minor limitation. It is not - it means the internal free-list pointers can be corrupted mid-update if two threads touch the pool simultaneously. That is a data race, which is undefined behavior.

```cpp
#include <iostream>
#include <memory_resource>
#include <thread>
#include <vector>

int main() {
    std::cout << "=== Thread safety of pool resources ===\n\n";

    // unsynchronized_pool_resource has NO internal locking:
    // - Allocation bumps internal pointers without atomics
    // - Free-list manipulation is not guarded by mutexes
    // - Sharing between threads = DATA RACE = UB
    //
    // This is BY DESIGN - the lack of synchronization is what makes it fast.

    // WRONG: sharing unsynchronized pool between threads
    // std::pmr::unsynchronized_pool_resource shared_pool;  // DATA RACE!
    // std::thread t1([&] { shared_pool.allocate(100); });
    // std::thread t2([&] { shared_pool.allocate(100); });

    // CORRECT: one pool per thread
    std::cout << "Strategy 1: Thread-local pools\n";
    auto worker = [](int id) {
        std::pmr::unsynchronized_pool_resource local_pool;
        std::pmr::vector<int> data(&local_pool);
        for (int i = 0; i < 1000; ++i) data.push_back(id * 1000 + i);
        std::cout << "  Thread " << id << ": " << data.size() << " items\n";
    };

    std::thread t1(worker, 1);
    std::thread t2(worker, 2);
    t1.join();
    t2.join();

    // CORRECT: synchronized_pool_resource for shared access
    std::cout << "\nStrategy 2: synchronized_pool_resource\n";
    std::pmr::synchronized_pool_resource shared_pool;
    auto shared_worker = [&shared_pool](int id) {
        std::pmr::vector<int> data(&shared_pool);
        for (int i = 0; i < 1000; ++i) data.push_back(id * 1000 + i);
        std::cout << "  Thread " << id << ": " << data.size() << " items\n";
    };

    std::thread t3(shared_worker, 3);
    std::thread t4(shared_worker, 4);
    t3.join();
    t4.join();

    std::cout << "\n=== Guidelines ===\n";
    std::cout << "unsynchronized: Fastest, but single-thread ONLY\n";
    std::cout << "synchronized:   Thread-safe, slight overhead from locking\n";
    std::cout << "thread_local:   Use unsynchronized with thread_local for best perf\n";

    return 0;
}
```

The safe patterns are: one `unsynchronized_pool_resource` per thread (each thread owns its own), or one `synchronized_pool_resource` shared across threads. The `thread_local` storage class is a natural fit for the first approach.

---

## Notes

- `unsynchronized_pool_resource` is the fastest PMR pool - no synchronization overhead.
- Use it for thread-local allocation or when you can guarantee single-threaded access.
- It organizes memory into pools of increasing block sizes - good for many small objects of varying sizes.
- Unlike `monotonic_buffer_resource`, it does support deallocation (blocks returned to pool for reuse).
- For multi-threaded access, use `synchronized_pool_resource` or use one `unsynchronized_pool_resource` per thread.
