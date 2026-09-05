# Understand std::owner_less for comparing weak_ptr in ordered containers

**Category:** Memory and Ownership  
**Standard:** C++11, C++17  
**Reference:** <https://en.cppreference.com/w/cpp/memory/owner_less>  

---

## Topic Overview

`std::weak_ptr` does not support `operator<` because pointer comparison depends on the managed object's address, which may change or become null after the object is destroyed. `std::owner_less` sidesteps this by comparing via the control block instead, and the control block stays alive for as long as any `shared_ptr` or `weak_ptr` references it. That makes ownership-based comparison safe and stable.

### The Problem

If you try to put `weak_ptr` into a `std::set`, you run straight into the missing `operator<`. Here is what that looks like:

```cpp
#include <memory>
#include <set>

auto sp1 = std::make_shared<int>(42);
auto sp2 = sp1;
std::weak_ptr<int> wp1 = sp1;
std::weak_ptr<int> wp2 = sp2;

// wp1 and wp2 both point to the same object
// But std::weak_ptr has no operator<

// std::set<std::weak_ptr<int>> s;  // Won't compile — no operator<
```

Both `wp1` and `wp2` refer to the same object, so comparing their raw addresses would be meaningless anyway. We need to compare by ownership identity, not object address.

### The Solution: owner_less

Pass `std::owner_less` as the comparator and the set works. The key behaviour to notice is that `sp3` and `sp1` share the same control block, so they count as the same key - inserting `sp3` is a no-op.

```cpp
#include <memory>
#include <set>
#include <iostream>

int main() {
    // owner_less compares by control block address, not object address
    std::set<std::weak_ptr<int>, std::owner_less<std::weak_ptr<int>>> tracked;

    auto sp1 = std::make_shared<int>(1);
    auto sp2 = std::make_shared<int>(2);
    auto sp3 = sp1;  // Same ownership as sp1

    tracked.insert(sp1);
    tracked.insert(sp2);
    tracked.insert(sp3);  // Same ownership as sp1 — NOT inserted (duplicate)

    std::cout << tracked.size() << "\n";  // 2 (not 3)

    // owner_less<void> works with any smart pointer type (C++17):
    std::set<std::weak_ptr<int>, std::owner_less<>> modern_set;
}
```

### Use Case: Observer Pattern

The classic application is an observer list where you want automatic cleanup of expired observers. Using `weak_ptr` as the set key means you hold no strong references to the observers, and `owner_less` makes the set compile.

```cpp
#include <memory>
#include <set>
#include <vector>
#include <algorithm>

class Subject {
    std::set<std::weak_ptr<Observer>, std::owner_less<>> observers_;

public:
    void subscribe(std::weak_ptr<Observer> obs) {
        observers_.insert(std::move(obs));
    }

    void notify() {
        // Remove expired observers and notify live ones
        for (auto it = observers_.begin(); it != observers_.end(); ) {
            if (auto sp = it->lock()) {
                sp->on_event();
                ++it;
            } else {
                it = observers_.erase(it);  // Expired
            }
        }
    }
};
```

The `it->lock()` call either gives you a live `shared_ptr` to work with, or returns null if the observer was already destroyed. Either way, no dangling access.

---

## Self-Assessment

### Q1: Why can't weak_ptr use regular pointer comparison

A `weak_ptr` may be expired - the managed object is deleted. Comparing by object address would mean comparing a dangling pointer, which is undefined behavior. `owner_less` compares by control block address instead, and the control block remains valid even after the managed object is destroyed. That makes the comparison well-defined for the entire lifetime of the `weak_ptr`.

### Q2: What does `owner_less<void>` provide

`owner_less<void>` (C++17) is a transparent comparator that works across `shared_ptr<T>`, `weak_ptr<T>`, and mixed comparisons. It provides `operator()` overloads for all combinations, so you can look up by `shared_ptr` in a set of `weak_ptr` without having to construct a `weak_ptr` first.

### Q3: When would you use owner_before directly

`shared_ptr::owner_before()` and `weak_ptr::owner_before()` are member functions that perform the same ownership comparison. Use them for one-off ad-hoc comparisons; use `owner_less` as the comparator template argument when you need it to plug into a sorted container.

---

## Notes

- `owner_less` is required for `std::set<weak_ptr<T>>` and `std::map<weak_ptr<T>, V>`.
- Prefer `owner_less<>` (void specialization) for maximum flexibility.
- The control block outlives the managed object - `owner_less` remains valid after expiry.
- Key use cases: observer patterns, caches, and dependency tracking.
