# Use node-based containers (map, set) with merge and extract (C++17)

**Category:** Standard Library Containers  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/container/map/extract>  

---

## Topic Overview

C++17 added `extract()` and `merge()` to node-based containers (`map`, `set`, `multimap`, `multiset`, and their `unordered_` variants). These operations transfer **ownership of internal nodes** between containers - no allocation, no copy, no move of the element itself.

The reason this matters: before C++17, moving an element from one `map` to another required erasing from the source and inserting into the destination - which meant destroying the value and reconstructing it (or at minimum moving it), plus a heap allocation for the new node. With `extract`/`merge`, you literally detach the node from one tree and attach it to another. No memory is allocated or freed; no constructor or destructor runs for the stored value.

There is also a unique capability that `extract` unlocks: **modifying a map key in-place**, which is otherwise impossible because map keys are `const`.

### What Is a Node

Node-based containers store each element in a separately allocated node (typically a tree node for ordered containers, a linked-list node for unordered). Before C++17, nodes were an implementation detail. Now you can access them directly:

```cpp
std::map<int, string> m:

  Tree structure (red-black tree):
       [3, "c"]              <- each box is a heap-allocated node
      /         \
  [1, "a"]    [5, "e"]
     \         /
   [2, "b"] [4, "d"]

  extract(3) -> detaches node [3,"c"] from tree
               Returns node_handle owning that memory
               No allocation, no copy, no destruction of "c"
```

The returned `node_handle` owns the node's memory. When you `insert` the handle into another container, the node is simply spliced in - pointer adjustments only, no data movement.

### extract() - Remove and Keep

Here is the core `extract` API. Notice that you can modify `node.key()` - this is the only standard way to change a map key without destroying and recreating the element:

```cpp
#include <map>
#include <iostream>

int main() {
    std::map<int, std::string> m = {{1,"a"}, {2,"b"}, {3,"c"}};

    // Extract by key - returns a node_handle
    auto node = m.extract(2);  // m now has {1,2} - node owns {2,"b"}

    if (!node.empty()) {
        std::cout << node.key() << ": " << node.mapped() << "\n";  // 2: b

        // Modify the KEY (impossible without extract!)
        node.key() = 20;

        // Re-insert with new key
        m.insert(std::move(node));  // m = {1,"a"}, {3,"c"}, {20,"b"}
    }

    // Extract by iterator - always succeeds, returns non-empty node
    auto it = m.find(3);
    auto node2 = m.extract(it);  // O(1) - iterator already located the node
}
```

The key-modification pattern (extract -> change key -> re-insert) is exception-safe in a way the old erase+insert pattern was not: while the node is held in `node_handle`, the data is fully owned and cannot be lost even if re-insertion throws.

### merge() - Transfer Nodes Between Containers

```cpp
#include <map>
#include <iostream>

int main() {
    std::map<int, std::string> a = {{1,"a"}, {3,"c"}, {5,"e"}};
    std::map<int, std::string> b = {{2,"b"}, {3,"C"}, {4,"d"}};

    a.merge(b);
    // a = {1,"a"}, {2,"b"}, {3,"c"}, {4,"d"}, {5,"e"}
    // b = {3,"C"}  <- duplicate key stayed in source!

    // merge() transfers nodes that don't conflict.
    // Conflicting keys (which already exist in target) remain in source.
}
```

After `merge`, `b` is left with only the keys that couldn't be transferred because they already existed in `a`. This is the correct behavior for a set-like operation: no data is lost, and no duplicates are silently overwritten.

### Key API

| Operation | Description | Complexity |
| --- | --- | --- |
| `extract(key)` | Remove node by key, return `node_handle` | O(log n) |
| `extract(iter)` | Remove node by iterator | O(1) amortized |
| `insert(node_handle)` | Insert extracted node | O(log n) |
| `merge(other_container)` | Transfer all non-conflicting nodes | O(n log n) worst |
| `node.key()` | Access/modify the key (mutable!) | O(1) |
| `node.mapped()` | Access/modify the mapped value | O(1) |
| `node.empty()` | Check if node handle is empty | O(1) |

---

## Self-Assessment

### Q1: Use node_type extract to move an element from one map to another without allocation

The `Tracked` type here prints every construction, copy, and move. The telling result is what you do NOT see during the extract/insert steps - no "COPIED" or "MOVED" messages appear:

```cpp
#include <iostream>
#include <map>
#include <string>

struct Tracked {
    std::string data;
    Tracked(const std::string& s) : data(s) {
        std::cout << "  Constructed: " << data << "\n";
    }
    Tracked(const Tracked& o) : data(o.data) {
        std::cout << "  COPIED: " << data << "\n";
    }
    Tracked(Tracked&& o) noexcept : data(std::move(o.data)) {
        std::cout << "  MOVED: " << data << "\n";
    }
    ~Tracked() {
        std::cout << "  Destroyed: " << data << "\n";
    }
};

int main() {
    std::cout << "=== Building maps ===\n";
    std::map<int, Tracked> source;
    source.emplace(std::piecewise_construct,
                   std::forward_as_tuple(1),
                   std::forward_as_tuple("alpha"));
    source.emplace(std::piecewise_construct,
                   std::forward_as_tuple(2),
                   std::forward_as_tuple("beta"));
    source.emplace(std::piecewise_construct,
                   std::forward_as_tuple(3),
                   std::forward_as_tuple("gamma"));

    std::map<int, Tracked> dest;

    // === Extract and re-insert: ZERO copies, ZERO moves! ===
    std::cout << "\n=== Extracting key 2 from source ===\n";
    auto node = source.extract(2);  // Detaches the node
    // No output! No copy, no move, no destruction.

    std::cout << "  Node empty? " << std::boolalpha << node.empty() << "\n";
    // Output: Node empty? false

    std::cout << "  Key: " << node.key() << "\n";
    std::cout << "  Value: " << node.mapped().data << "\n";

    std::cout << "\n=== Inserting into dest ===\n";
    auto result = dest.insert(std::move(node));
    // No output! Node was spliced into dest's tree - no copy/move of Tracked!

    std::cout << "  Inserted? " << result.inserted << "\n";
    // Output: Inserted? true

    // === Modify key before re-insertion ===
    std::cout << "\n=== Extract key 1, change to key 10 ===\n";
    auto node2 = source.extract(1);
    std::cout << "  Original key: " << node2.key() << "\n";
    node2.key() = 10;  // Change key! Only possible with extract.
    std::cout << "  New key: " << node2.key() << "\n";
    dest.insert(std::move(node2));  // Zero copies/moves again

    std::cout << "\n=== Final state ===\n";
    std::cout << "Source: ";
    for (const auto& [k, v] : source) std::cout << k << "=" << v.data << " ";
    std::cout << "\nDest:   ";
    for (const auto& [k, v] : dest) std::cout << k << "=" << v.data << " ";
    std::cout << "\n";
    // Output:
    // Source: 3=gamma
    // Dest:   2=beta 10=alpha

    std::cout << "\n=== Cleanup ===\n";
    return 0;
}
```

During "Extracting key 2" and "Inserting into dest", the `Tracked` object never prints anything. That silence is the whole point: zero construction, zero copy, zero move. The node just changed which tree's pointer structure it belongs to.

**How it works:**

- `extract(key)` removes the node from the internal tree and returns a `node_handle` that owns the node's memory. No copy or move of the stored value occurs - the node is simply unlinked from the tree.
- `insert(std::move(node))` links the node into the destination tree. Again, no copy/move - just pointer manipulation.
- The `Tracked` class proves this: no "COPIED" or "MOVED" messages appear during extract/insert.
- `node.key()` returns a **mutable reference** to the key - this is the only way to change a key in a map without destroying and re-creating the element.
- `extract(key)` returns an empty `node_handle` if the key doesn't exist. Always check `node.empty()`.

### Q2: Show how merge() transfers nodes from one std::set to another

The interesting behavior here is what happens to duplicate keys: they stay in the source rather than being silently dropped or overwriting the target's entry:

```cpp
#include <iostream>
#include <set>
#include <string>

template <typename S>
void print(const std::string& label, const S& s) {
    std::cout << label << ": {";
    bool first = true;
    for (const auto& x : s) {
        if (!first) std::cout << ", ";
        std::cout << x;
        first = false;
    }
    std::cout << "}\n";
}

int main() {
    // === Basic merge ===
    std::set<int> a = {1, 3, 5, 7, 9};
    std::set<int> b = {2, 3, 4, 5, 6};

    std::cout << "Before merge:\n";
    print("  a", a);  // a: {1, 3, 5, 7, 9}
    print("  b", b);  // b: {2, 3, 4, 5, 6}

    a.merge(b);

    std::cout << "\nAfter a.merge(b):\n";
    print("  a", a);  // a: {1, 2, 3, 4, 5, 6, 7, 9}
    print("  b", b);  // b: {3, 5}  <- duplicates stayed in source!

    // === Merge with multiset (accepts duplicates) ===
    std::multiset<int> ms = {10, 20};
    std::set<int> c = {10, 30, 40};

    ms.merge(c);
    std::cout << "\nAfter ms.merge(c):\n";
    print("  ms", ms);  // ms: {10, 10, 20, 30, 40}  <- multiset keeps both 10s
    print("  c", c);    // c: {}  <- all transferred (multiset has no conflicts)

    // === Merge between compatible types ===
    // You can merge between different allocators or comparators
    // as long as the node types are compatible.
    std::set<int, std::greater<>> desc = {100, 200, 300};
    std::set<int> asc;

    // asc.merge(desc);  // ERROR: different comparator = different node type

    // But same-typed containers work fine:
    std::set<int> x = {1, 2};
    std::set<int> y = {3, 4};
    x.merge(y);
    print("\n  x after merge", x);  // {1, 2, 3, 4}
    print("  y after merge", y);    // {} (all transferred)

    // === merge() is O(N log N) worst case ===
    // Each node is extracted and inserted individually.
    // For N nodes, each insert is O(log(size_of_target)).

    return 0;
}
```

The multiset example shows the contrast clearly: when the target is a `multiset` (which has no "conflict" concept), every single node transfers successfully and `c` ends up empty. When the target is a `set`, conflicting keys stay behind in the source.

**How it works:**

- `a.merge(b)` attempts to transfer every node from `b` to `a`. For `set`, if a key already exists in `a`, that node **stays in `b`** (no overwrite, no duplication). For `multiset`, all transfers succeed because duplicates are allowed.
- No allocation, copy, or move of element data - nodes are unlinked from `b`'s tree and linked into `a`'s tree.
- After merge, `b` contains only the elements that couldn't be transferred (duplicates in set).
- Merge works between containers with the **same node type** - typically the same container type with the same key/value/comparator types.

### Q3: Explain why these node operations are useful for lock-free data structures

`extract` and `merge` don't make containers lock-free on their own, but they significantly reduce how long locks need to be held and eliminate memory allocation under the lock. This example walks through three concrete patterns:

```cpp
#include <iostream>
#include <map>
#include <set>
#include <string>
#include <mutex>
#include <thread>
#include <vector>

// === Use Case 1: Atomic-like map key modification ===
// Without extract, changing a key requires:
//   1. Find element
//   2. Copy the value
//   3. Erase old entry  <- element briefly doesn't exist!
//   4. Insert with new key <- may fail (out of memory)
// Between steps 3 and 4, the element is GONE.
// With extract: node is detached -> key changed -> re-inserted.
// The element is always owned (either by map or by node_handle).

void key_change_safe() {
    std::map<int, std::string> m = {{1, "data"}, {2, "info"}};

    auto nh = m.extract(1);   // Step 1: detach (no data loss possible)
    nh.key() = 100;           // Step 2: modify key
    m.insert(std::move(nh));  // Step 3: re-insert (if this fails, nh still owns data)
    // At no point was the value destroyed or absent.
}

// === Use Case 2: Reducing lock hold time ===
// Instead of doing expensive work under a lock, extract the node,
// release the lock, then work on the node privately.

struct Cache {
    std::mutex mtx;
    std::map<int, std::string> data;

    void process_and_transfer(int key, std::map<int, std::string>& dest) {
        std::map<int, std::string>::node_type node;

        {  // Short critical section
            std::lock_guard lock(mtx);
            node = data.extract(key);  // O(log n), no allocation
        }  // Lock released!

        // Do expensive work on the node OUTSIDE the lock
        if (!node.empty()) {
            node.mapped() += "_processed";  // Modify privately

            // Insert into destination (different container, no lock needed)
            dest.insert(std::move(node));
        }
    }
};

// === Use Case 3: Bulk transfer between locked containers ===
void bulk_transfer() {
    std::set<int> hot_set = {1, 2, 3, 4, 5};
    std::set<int> cold_set;
    std::mutex hot_mtx, cold_mtx;

    auto transfer = [&]() {
        // Build a temporary set outside any lock
        std::set<int> temp;

        {  // Lock hot briefly - just extract
            std::lock_guard lock(hot_mtx);
            hot_set.merge(temp);  // Wait, this is backwards...
            // Actually: extract nodes from hot into temp
            for (auto it = hot_set.begin(); it != hot_set.end(); ) {
                if (*it % 2 == 0) {  // Transfer even numbers
                    temp.insert(hot_set.extract(it++));  // No allocation!
                } else {
                    ++it;
                }
            }
        }

        {  // Lock cold briefly - just merge
            std::lock_guard lock(cold_mtx);
            cold_set.merge(temp);  // Transfer all nodes
        }
    };

    transfer();
    std::cout << "Hot set:  ";
    for (int x : hot_set) std::cout << x << " ";
    std::cout << "\nCold set: ";
    for (int x : cold_set) std::cout << x << " ";
    std::cout << "\n";
    // Hot set:  1 3 5
    // Cold set: 2 4
}

int main() {
    key_change_safe();
    bulk_transfer();

    std::cout << "\n=== Why node operations help concurrency ===\n";
    std::cout << "1. No allocation under lock -> shorter critical sections\n";
    std::cout << "2. Node can be modified outside lock -> less contention\n";
    std::cout << "3. No data loss during key changes -> exception-safe\n";
    std::cout << "4. merge() is one operation -> simpler lock protocol\n";
    std::cout << "5. Node handle is move-only -> clear ownership\n";

    return 0;
}
```

Use Case 2 is the most practically useful: extract under a short lock, release the lock, do expensive work privately on the node, then insert into the destination. The old way required holding the lock while copying the value out, which held the lock longer and risked blocking other threads on a slow memory allocation.

**How it works:**

Note: `extract`/`merge` don't make containers lock-free. They help **reduce lock contention** and improve **exception safety** in concurrent code:

1. **Shorter critical sections:** Extract nodes under a lock (O(log n), no allocation), then process them outside the lock. Without extract, you'd copy data under the lock, erase, release, then insert into another container - more time holding the lock.

2. **No allocation under lock:** `extract` and `merge` never allocate or deallocate memory. This eliminates the risk of a slow malloc/free call while holding a mutex.

3. **Exception safety:** When changing a key, if `insert` fails after `extract`, the node handle still owns the data. With the old erase+insert pattern, if insert throws (e.g., `bad_alloc`), the element is lost.

4. **Transfer semantics:** `merge()` atomically (from the container's perspective) moves nodes between containers. You can lock source, extract into temp, unlock source, lock dest, merge temp into dest, unlock dest - each lock is held briefly.

---

## Notes

- **extract() invalidates iterators** to the extracted element only. All other iterators remain valid.
- **merge() invalidates iterators** to transferred elements in the source container. Iterators in the destination are not invalidated.
- **Works with unordered containers too:** `unordered_map::extract()`, `unordered_set::merge()`, etc. Same principle, different internal structure (hash table nodes instead of tree nodes).
- **node_handle is move-only** - you can't copy it. This enforces unique ownership of the node's memory.
- **Modifying keys:** `node.key()` is the ONLY standard way to modify a key without destroying and recreating the element. Before C++17, you had to erase and re-insert.
- **Performance:** Extract/insert are O(log n) for ordered containers, O(1) amortized for unordered. The real win is avoiding allocation/deallocation, which is often the expensive part.
