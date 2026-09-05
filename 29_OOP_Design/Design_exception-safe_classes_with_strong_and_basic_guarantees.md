# Design exception-safe classes with strong and basic guarantees

**Category:** OOP Design  
**Standard:** C++11  

---

## Topic Overview

Exception safety guarantees are contractual promises your class makes about what happens to its state when an operation throws. Think of them as a spectrum from "anything might be broken" (no guarantee) to "if something throws, it's as if nothing happened" (strong guarantee). Knowing which level you're providing - and choosing the right one for each operation - is what separates a reliable class from a landmine.

| Guarantee | Promise | State After Throw | Example |
| --- | --- | --- | --- |
| **No-throw** | Never throws | N/A | `swap()`, destructors, `noexcept` move |
| **Strong** | Commit-or-rollback | Original state preserved | `std::vector::push_back` |
| **Basic** | No leaks, valid state | Valid but unspecified | `std::vector::insert` (partial) |
| **None** | Anything goes | Possibly corrupted | Avoid this |

### Key Techniques

The strong guarantee strategy boils down to one discipline: never touch observable state until all the work that could throw is already done. Do everything on temporaries or copies first, then commit atomically with a `noexcept` swap or move. If you remember one thing from this topic, make it that rule.

```cpp
Strong guarantee strategy:

1. Perform all throwing work on temporaries / copies
2. Commit results with non-throwing operations (swap/move)
3. Never modify observable state before all throwing ops complete
```

---

## Self-Assessment

### Q1: Write a class with the strong exception guarantee using copy-and-swap

The copy-and-swap idiom is the classic way to achieve the strong guarantee for assignment. The trick is that the operator takes its argument by value - so the copy (which can throw) happens during argument passing, before the function body even starts. Once inside the function, we only do a `noexcept` swap. Either you get a good copy and the swap succeeds, or the copy throws and the object is untouched. The object never sees a half-finished state.

```cpp
#include <algorithm>
#include <cstring>
#include <stdexcept>
#include <utility>

class DynamicBuffer {
    char* data_;
    size_t size_;
    size_t capacity_;
public:
    explicit DynamicBuffer(size_t cap = 0)
        : data_(cap ? new char[cap]{} : nullptr)
        , size_(0), capacity_(cap) {}

    ~DynamicBuffer() { delete[] data_; }

    // Copy constructor - can throw (allocates)
    DynamicBuffer(const DynamicBuffer& o)
        : data_(o.size_ ? new char[o.capacity_] : nullptr)
        , size_(o.size_), capacity_(o.capacity_) {
        std::memcpy(data_, o.data_, size_);
    }

    // Move constructor - noexcept!
    DynamicBuffer(DynamicBuffer&& o) noexcept
        : data_(std::exchange(o.data_, nullptr))
        , size_(std::exchange(o.size_, 0))
        , capacity_(std::exchange(o.capacity_, 0)) {}

    // Unified assignment via copy-and-swap -> STRONG guarantee
    DynamicBuffer& operator=(DynamicBuffer o) noexcept {
        swap(*this, o);  // o dies with old data
        return *this;
    }

    friend void swap(DynamicBuffer& a, DynamicBuffer& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
        swap(a.capacity_, b.capacity_);
    }

    // append() with STRONG guarantee
    void append(const char* src, size_t n) {
        if (size_ + n > capacity_) {
            // Step 1: All throwing work on a temporary
            DynamicBuffer tmp(std::max(capacity_ * 2, size_ + n));
            std::memcpy(tmp.data_, data_, size_);
            std::memcpy(tmp.data_ + size_, src, n);
            tmp.size_ = size_ + n;
            // Step 2: Commit with noexcept swap
            swap(*this, tmp);  // If we get here, commit
        } else {
            // No allocation needed - basic guarantee is sufficient
            std::memcpy(data_ + size_, src, n);
            size_ += n;
        }
    }

    size_t size() const noexcept { return size_; }
};
```

Notice the `append` method has two paths: when reallocation is needed, it builds a complete temporary first and only swaps at the end; when there's room, it modifies in place and only promises the basic guarantee (which is all that's needed there). Choosing the right guarantee per operation, rather than always aiming for the strongest, is the practical skill here.

### Q2: Show the "do all work, then swap" idiom for multi-member updates

When you need to update several members atomically, the pattern is the same: make copies of all the new values first (the copies can throw), then commit all of them under a lock using `noexcept` moves. If any copy throws, the object is untouched because you haven't touched it yet. The reason this trips people up is that it feels wasteful to copy everything before touching the object - but that's precisely the cost of the strong guarantee.

```cpp
#include <string>
#include <vector>
#include <mutex>

class UserProfile {
    std::string name_;
    std::string email_;
    std::vector<std::string> roles_;
    mutable std::mutex mtx_;
public:
    // STRONG guarantee: either all fields update or none do
    void update(const std::string& name, const std::string& email,
                const std::vector<std::string>& roles) {
        // Phase 1: all potentially-throwing copies happen here
        std::string new_name = name;          // may throw
        std::string new_email = email;        // may throw
        std::vector<std::string> new_roles = roles;  // may throw

        // Phase 2: commit with non-throwing operations
        std::lock_guard lk(mtx_);
        name_  = std::move(new_name);    // noexcept (move)
        email_ = std::move(new_email);   // noexcept
        roles_ = std::move(new_roles);   // noexcept
    }
    // If Phase 1 throws, object untouched -> strong guarantee
};
```

The separation into two explicit phases makes the intent clear to the next reader. Phase 1 is "gather the new values, this can fail". Phase 2 is "commit, this must not fail". Writing it this way is also self-documenting - anyone who moves a potentially-throwing operation into Phase 2 is introducing a bug, and it's obvious from the structure.

### Q3: Implement a transactional container with rollback on failure

Sometimes the "do everything on a temporary then swap" approach doesn't fit naturally - for instance, when you're building up state incrementally in a loop. The `ScopeGuard` pattern handles this: you register a rollback action upfront, and it runs automatically on scope exit unless you explicitly commit. Think of it as a transaction with an automatic abort on any unexpected exit.

```cpp
#include <vector>
#include <functional>
#include <stdexcept>
#include <iostream>

// ScopeGuard for automatic rollback
class ScopeGuard {
    std::function<void()> rollback_;
    bool committed_ = false;
public:
    explicit ScopeGuard(std::function<void()> fn) : rollback_(std::move(fn)) {}
    ~ScopeGuard() { if (!committed_) rollback_(); }
    void commit() noexcept { committed_ = true; }
};

// Transactional batch insert with strong guarantee
template<typename T>
class TransactionalVector {
    std::vector<T> data_;
public:
    // Insert multiple items - all succeed or all roll back
    void batch_insert(const std::vector<T>& items) {
        auto original_size = data_.size();
        ScopeGuard guard([&] {
            data_.resize(original_size);  // rollback
        });

        for (const auto& item : items) {
            data_.push_back(item);  // may throw
        }

        guard.commit();  // all succeeded, don't rollback
    }

    size_t size() const { return data_.size(); }
};

int main() {
    TransactionalVector<std::string> tv;
    try {
        tv.batch_insert({"alpha", "beta", "gamma"});
        std::cout << "After success: " << tv.size() << "\n";  // 3
    } catch (...) {}
    return 0;
}
```

The guard captures the original size and shrinks the vector back to it if anything goes wrong. This works because `resize` to a smaller size is `noexcept` for standard containers. The pattern generalizes: capture the "before" state, register a rollback that restores it, and commit at the end.

---

## Notes

- **Destructors must never throw** - they're implicitly `noexcept` since C++11.
- Mark move constructors/assignment `noexcept` for strong guarantees in containers.
- The copy-and-swap idiom gives the strong guarantee "for free" but costs one extra allocation.
- For multi-member updates: do all throwing work first, then commit with moves/swaps.
- `std::vector::push_back` provides the strong guarantee; `insert` in the middle may only provide the basic guarantee.
- Use ScopeGuard / RAII for multi-step operations needing rollback.
