# Understand Moved-From State Guarantees in the Standard Library

**Category:** Move Semantics and Value Categories  
**Standard:** C++11 and later  
**Reference:** <https://en.cppreference.com/w/cpp/utility/move>  

---

## Topic Overview

When an object is moved from via `std::move`, the C++ standard places it in a **valid but unspecified** state. This is a precise term: "valid" means the object's invariants hold and it can be used with any operation that has no precondition on its value; "unspecified" means you cannot predict what value it holds.

The reason this trips people up is the word "valid." People assume "valid" implies "empty" or "zero" - but it does not. The standard deliberately leaves the post-move value unspecified so that implementations can choose the cheapest possible move. For `std::vector`, stealing the heap buffer and leaving the source empty happens to be the cheapest option (and all major implementations do it), but you are not supposed to rely on that.

```cpp
┌──────────────────────────────────────────────────────────────────┐
│         Operations on Moved-From Standard Library Objects        │
├──────────────────────────┬───────────────────────────────────────┤
│  ALWAYS SAFE             │  CONDITIONALLY SAFE / UNSAFE          │
├──────────────────────────┼───────────────────────────────────────┤
│  Destruction             │  Reading .size(), .empty() -- valid   │
│  Assignment (copy or     │    but result is unspecified          │
│    move into it)         │  Iterating elements -- might be empty │
│  swap()                  │  operator[] / .at() -- might be empty │
│                          │    so index could be out of range     │
│                          │  .front() / .back() -- UB if empty   │
└──────────────────────────┴───────────────────────────────────────┘
```

The key insight is that **"valid but unspecified" does NOT mean "empty"**. A moved-from `std::vector` is typically empty (because moving steals the buffer), but the standard does not guarantee this. A moved-from `std::string` might be empty, might retain a small-buffer-optimized string, or might hold some other value - it depends entirely on the implementation. In practice, most implementations leave containers empty after a move, but writing code that depends on this is non-portable.

The standard provides **stronger guarantees** for a few specific types:

```cpp
┌─────────────────────────────────────┬────────────────────────────────────┐
│  Type                               │  Moved-from guarantee              │
├─────────────────────────────────────┼────────────────────────────────────┤
│  std::unique_ptr<T>                 │  Holds nullptr (guaranteed)        │
│  std::shared_ptr<T>                 │  Holds nullptr (guaranteed)        │
│  std::optional<T>                   │  Disengaged (holds no value)       │
│  std::future<T>                     │  Not valid (.valid() == false)     │
│  std::thread                        │  Not joinable                      │
│  std::string (typical)              │  Empty (NOT guaranteed by std)     │
│  std::vector<T> (typical)           │  Empty (NOT guaranteed by std)     │
│  Arithmetic types (int, double)     │  UNCHANGED -- move == copy         │
└─────────────────────────────────────┴────────────────────────────────────┘
```

For your own types, you should document the moved-from state explicitly. The "valid but unspecified" baseline from the standard is the minimum, but stronger guarantees (like "moved-from objects are default-constructed") make your API easier to reason about. The key design rule: a class should be destructible and assignable-to after being moved from, at minimum.

---

## Self-Assessment

### Q1: What can you safely do with a moved-from `std::vector`

The safe operations all share one property: they have no precondition on the current value of the object. Destruction, assignment, `clear()`, and `swap()` all work regardless of what state the vector is in. Anything that reads or indexes into the content is risky because you do not know what is there:

```cpp
#include <iostream>
#include <vector>
#include <string>

int main() {
    std::vector<std::string> names{"Alice", "Bob", "Charlie"};
    std::cout << "Before move: size=" << names.size() << "\n";

    // Move-construct a new vector from 'names'
    std::vector<std::string> stolen = std::move(names);
    std::cout << "Stolen: size=" << stolen.size() << "\n";

    // 'names' is now in a valid-but-unspecified state.
    // These operations are ALWAYS safe:

    // 1. Destruction -- happens automatically at scope end

    // 2. Assignment -- reset it to a known state
    names = {"David", "Eve"};
    std::cout << "After assign: size=" << names.size() << "\n";

    // 3. clear() -- has no precondition on current state
    std::vector<std::string> v2{"X", "Y"};
    auto v3 = std::move(v2);
    v2.clear();  // safe: clear has no precondition
    std::cout << "After clear: size=" << v2.size() << "\n";

    // 4. swap -- also has no precondition
    std::vector<std::string> v4{"Z"};
    auto v5 = std::move(v4);
    std::vector<std::string> fresh{"New"};
    v4.swap(fresh);  // safe
    std::cout << "After swap: v4 size=" << v4.size() << "\n";

    // RISKY: v2.front() -- if the vector happens to be empty, this is UB
    // RISKY: v2[0] -- same problem
    // RISKY: relying on specific size after move -- non-portable

    return 0;
}
```

### Q2: How do `unique_ptr` and `shared_ptr` differ from containers in their moved-from guarantee

Smart pointers are special: the standard explicitly says their moved-from state is `nullptr`. This is a stronger, documented guarantee - not just typical implementation behavior. You can rely on it unconditionally. Containers give you no such promise:

```cpp
#include <iostream>
#include <memory>
#include <cassert>

struct Resource {
    int id;
    Resource(int id) : id(id) {
        std::cout << "  Resource " << id << " created\n";
    }
    ~Resource() {
        std::cout << "  Resource " << id << " destroyed\n";
    }
};

int main() {
    // unique_ptr: moved-from state is GUARANTEED to be nullptr
    auto up = std::make_unique<Resource>(1);
    std::cout << "up before move: " << (up ? "has value" : "null") << "\n";

    auto up2 = std::move(up);
    std::cout << "up after move:  " << (up ? "has value" : "null") << "\n";
    assert(up == nullptr);  // guaranteed by the standard

    // You can safely reuse it:
    up = std::make_unique<Resource>(2);
    std::cout << "up after reassign: " << (up ? "has value" : "null") << "\n";

    // shared_ptr: moved-from state is GUARANTEED to be nullptr
    auto sp = std::make_shared<Resource>(3);
    std::cout << "\nsp use_count before move: " << sp.use_count() << "\n";

    auto sp2 = std::move(sp);
    std::cout << "sp use_count after move:  " << sp.use_count() << "\n";
    assert(sp == nullptr);  // guaranteed by the standard
    assert(sp.use_count() == 0);

    // Compare with std::string: NOT guaranteed empty
    std::string s = "Hello, World!";
    std::string s2 = std::move(s);
    // s is valid but unspecified -- it MIGHT be empty, might not
    // On most implementations: s == "" but do NOT rely on this.
    std::cout << "\nMoved-from string: \"" << s << "\"\n";
    std::cout << "Moved-from string size: " << s.size() << "\n";
    // Only safe to assign or destroy:
    s = "Reused";
    std::cout << "Reassigned string: \"" << s << "\"\n";

    return 0;
}
```

The `std::string` case at the end is deliberately left observable so you can see what your implementation does. The output may surprise you - some implementations leave a non-empty string behind due to small-buffer optimization. That is entirely conforming behavior.

### Q3: How should you document moved-from guarantees for your own types

When you write a class with move semantics, you are setting a contract. The minimum is "valid but unspecified," but offering a stronger guarantee - such as "moved-from equals default-constructed" - makes your class much easier to use correctly. This example shows how to enforce a documented guarantee explicitly in the move constructor and move assignment:

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <utility>
#include <cassert>

/// A connection pool that documents its moved-from state explicitly.
///
/// **Moved-from state guarantee**: A moved-from ConnectionPool is in
/// a valid, empty state equivalent to a default-constructed pool.
/// Specifically:
///   - name() returns ""
///   - size() returns 0
///   - is_open() returns false
///   - It can be assigned to, destroyed, or reopened.
///
/// This is STRONGER than "valid but unspecified" and makes the class
/// easier to reason about.
class ConnectionPool {
    std::string name_;
    std::vector<int> connections_;  // simplified: just ints as FDs
    bool open_ = false;

public:
    ConnectionPool() = default;

    ConnectionPool(std::string name, std::vector<int> conns)
        : name_(std::move(name))
        , connections_(std::move(conns))
        , open_(true) {}

    // Move constructor: leave source in documented default state
    ConnectionPool(ConnectionPool&& other) noexcept
        : name_(std::move(other.name_))
        , connections_(std::move(other.connections_))
        , open_(std::exchange(other.open_, false))
    {
        // Explicitly clear the source to honor our documented guarantee.
        // std::move on string/vector likely empties them, but we
        // enforce the guarantee explicitly.
        other.name_.clear();
        other.connections_.clear();
    }

    // Move assignment: same discipline
    ConnectionPool& operator=(ConnectionPool&& other) noexcept {
        if (this != &other) {
            name_ = std::move(other.name_);
            connections_ = std::move(other.connections_);
            open_ = std::exchange(other.open_, false);
            other.name_.clear();
            other.connections_.clear();
        }
        return *this;
    }

    ConnectionPool(const ConnectionPool&) = default;
    ConnectionPool& operator=(const ConnectionPool&) = default;

    const std::string& name() const { return name_; }
    std::size_t size() const { return connections_.size(); }
    bool is_open() const { return open_; }
};

int main() {
    ConnectionPool pool("MainDB", {10, 11, 12});
    std::cout << "Before move: name=" << pool.name()
              << " size=" << pool.size()
              << " open=" << pool.is_open() << "\n";

    ConnectionPool pool2 = std::move(pool);
    std::cout << "Destination:  name=" << pool2.name()
              << " size=" << pool2.size()
              << " open=" << pool2.is_open() << "\n";

    // Our documented guarantee: moved-from == default-constructed
    std::cout << "Moved-from:   name=\"" << pool.name()
              << "\" size=" << pool.size()
              << " open=" << pool.is_open() << "\n";

    assert(pool.name().empty());
    assert(pool.size() == 0);
    assert(!pool.is_open());

    // Can safely reuse:
    pool = ConnectionPool("BackupDB", {20, 21});
    std::cout << "Reused:       name=" << pool.name()
              << " size=" << pool.size() << "\n";

    return 0;
}
```

`std::exchange(other.open_, false)` is worth highlighting: it atomically reads the old value and sets the source to a known state in a single expression. It is both concise and self-documenting. This is the idiomatic way to handle bool-like validity flags in move operations.

---

## Notes

- **"Valid but unspecified"** is the standard's baseline: the object satisfies its invariants, but you don't know what value it holds. Only operations without value preconditions are safe.
- The **only universally safe** operations on any moved-from object are destruction, assignment, and swap.
- `std::unique_ptr` and `std::shared_ptr` are **guaranteed null** after move - this is explicitly stated in the standard, not just typical behavior.
- For containers, most implementations leave them empty after move, but **the standard does not guarantee this**. Do not write code relying on it.
- For your own types, document a **stronger** guarantee when practical. "Moved-from objects are equivalent to default-constructed" is a common and useful contract.
- Use `std::exchange` in move operations to set the source to a known state in a single expression - it's both concise and self-documenting.
- If your type has a `bool`-like validity flag, always reset it in the move constructor/assignment - forgetting this is a common source of double-free or use-after-move bugs.
