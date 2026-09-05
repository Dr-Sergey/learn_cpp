# Implement persistent and immutable data structures in C++

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://github.com/arximboldi/immer>  

---

## Topic Overview

Persistent data structures preserve previous versions when modified. Instead of mutating in place, operations return a new version that shares most of the old structure via **structural sharing**. This is how functional languages like Clojure and Scala handle data - and you can use the same techniques in C++.

The reason this matters is that immutable data is inherently safe to share between threads, trivially supports undo/redo, and eliminates a whole class of bugs caused by unexpected mutation. The trade-off is a small overhead per operation, which is usually negligible in practice.

### Why Persistent Data Structures

The contrast with mutable structures is stark. With a mutable container, the old version is gone the moment you modify it. With a persistent one, both the old and new versions co-exist and share memory:

```cpp
Mutable:     v1 = [1, 2, 3]
             v1.push_back(4)    // v1 is now [1, 2, 3, 4] — old version lost

Persistent:  v1 = [1, 2, 3]
             v2 = v1.push_back(4)  // v1 = [1, 2, 3], v2 = [1, 2, 3, 4]
             // Both versions exist simultaneously, sharing memory
```

The key word is "sharing." A naive implementation would copy the entire structure for every modification, which would be expensive. Real persistent data structures only copy the small part of the structure that changed, and share pointers to everything else.

### Persistent Vector (using immer library)

The `immer` library provides production-quality persistent data structures for C++. Here you can see all three versions of the vector living simultaneously with no explicit copying:

<!-- compile: needs third-party library header `immer/vector.hpp` -->
```cpp
#include <immer/vector.hpp>
#include <iostream>

int main() {
    auto v1 = immer::vector<int>{1, 2, 3};
    auto v2 = v1.push_back(4);     // v1 unchanged, v2 has 4 elements
    auto v3 = v2.set(0, 99);       // v3 = [99, 2, 3, 4]

    std::cout << "v1: ";
    for (auto x : v1) std::cout << x << " ";  // 1 2 3
    std::cout << "\nv2: ";
    for (auto x : v2) std::cout << x << " ";  // 1 2 3 4
    std::cout << "\nv3: ";
    for (auto x : v3) std::cout << x << " ";  // 99 2 3 4
    std::cout << "\n";

    // Structural sharing: v1, v2, v3 share most of their memory
    // Only the "spine" is copied — O(log N) per operation
}
```

Internally, `immer::vector` uses a radix-balanced tree (a Hash Array Mapped Trie). When you call `push_back` or `set`, only the nodes along the path to the modified element are copied - all other nodes are shared between versions via reference counting.

### Simple Persistent List (from scratch)

A singly-linked list with immutable nodes is the simplest persistent data structure you can implement yourself. Since nodes are never modified after creation, sharing them between list versions is completely safe:

```cpp
#include <memory>
#include <iostream>

template<typename T>
class PList {
    struct Node {
        T value;
        std::shared_ptr<const Node> next;
    };
    std::shared_ptr<const Node> head_;

public:
    PList() = default;
    PList(T value, PList tail)
        : head_(std::make_shared<Node>(Node{std::move(value), tail.head_})) {}

    PList prepend(T value) const {
        return PList(std::move(value), *this);
    }

    bool empty() const { return !head_; }
    const T& front() const { return head_->value; }
    PList tail() const { return PList{head_->next}; }

private:
    PList(std::shared_ptr<const Node> h) : head_(std::move(h)) {}
};

int main() {
    auto l1 = PList<int>{}.prepend(3).prepend(2).prepend(1);
    auto l2 = l1.prepend(0);  // l1 is [1,2,3], l2 is [0,1,2,3]
    // l2 shares l1's nodes — zero copying

    for (auto l = l2; !l.empty(); l = l.tail())
        std::cout << l.front() << " ";  // 0 1 2 3
}
```

When `l2` is created by prepending `0` to `l1`, only one new `Node` is allocated. The rest of the list (the nodes containing 1, 2, and 3) is shared between `l1` and `l2` via their `shared_ptr` connections. No copying happens.

---

## Self-Assessment

### Q1: What is structural sharing and why is it efficient

When creating a new version of a data structure, only the changed nodes are copied; unchanged subtrees are shared between old and new versions via shared pointers. A balanced tree of N elements shares all but O(log N) nodes, making updates O(log N) in both time and space. The reason this is efficient is that most of the structure is the same between versions - you only pay for what changed.

### Q2: When are persistent data structures useful in C++

- Undo/redo systems, where each state is a version you can return to
- Concurrent programming, where immutable data needs no locks and can be safely shared
- Functional reactive programming, where you compose transformations over immutable state
- Transactional memory systems that need commit/rollback semantics
- Time-travel debugging, where you want to replay execution from any historical state

### Q3: What's the performance trade-off vs mutable structures

Persistent structures add O(log32 N) overhead per access when using radix-balanced trees. For N less than one million, this overhead is nearly constant (the tree is at most a few levels deep). The trade-off is clear: slightly slower element access in exchange for free snapshotting, thread safety without locks, and no iterator invalidation when you create new versions.

---

## Notes

- The **immer** library provides production-quality persistent vectors, maps, and sets for C++ and is well worth looking at if you need this in real code.
- Persistent structures use reference counting (`shared_ptr`) for memory management, so they integrate naturally with standard C++ idioms.
- Hash-Array Mapped Tries (HAMT) are the most common implementation strategy - used by Clojure, Scala, and immer.
- Copy-on-write (CoW) is a simpler but less efficient form of persistence - it copies the whole structure on first modification rather than sharing subtrees.
