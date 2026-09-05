# Implement copy-on-write (COW) semantics for expensive-to-copy types

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

**Copy-on-Write (COW)** is a lazy-copying strategy: instead of paying the full cost of a deep copy every time you copy an object, you let multiple copies share the same underlying data until one of them actually needs to modify it. At that point - and only at that point - does the modifying copy "detach" and allocate its own private data. It's a classic space-time trade-off that shines whenever copies are common but writes are rare.

The lifecycle looks like this:

```cpp
COW lifecycle:
  A = original        ->  A.data refcount=1
  B = A  (cheap copy)  ->  A.data refcount=2 (shared)
  B.modify()           ->  B detaches -> B.data refcount=1 (new copy)
                          A.data refcount=1 (unchanged)
```

The table below contrasts COW against eager (always-deep) copying so you can see the trade-offs clearly:

| Aspect | Eager Copy | COW |
| --- | :---: | :---: |
| Copy cost | O(n) always | O(1) until write |
| Write cost | O(1) | O(n) on first write (detach) |
| Memory usage | Higher (always copy) | Lower (shared) |
| Thread safety | Inherent | **Needs atomic refcount** |
| Cache locality | Better | May be worse (shared heap) |

COW is a great fit for scenarios where you take many read-only copies (configuration snapshots, undo stacks, functional-style data structures) and writes are the exception rather than the rule. If you write as often as you read, the detach cost makes COW a net loss.

---

## Self-Assessment

### Q1: Implement a basic COW string class

The simplest COW implementation uses `std::shared_ptr` for the shared data, which gives you atomic reference counting for free. The key discipline is: read operations go straight through, write operations call `detach()` first. Every public method that can modify data must call `detach()` before touching anything.

**Answer:**

```cpp
#include <memory>
#include <cstring>
#include <iostream>
#include <utility>

class CowString {
public:
    CowString() : data_(std::make_shared<std::string>()) {}
    explicit CowString(const char* s) : data_(std::make_shared<std::string>(s)) {}

    // Copy is cheap: just share the pointer
    CowString(const CowString&) = default;
    CowString& operator=(const CowString&) = default;

    // Read access: no copy needed
    char operator[](size_t i) const {
        return (*data_)[i];
    }

    // Write access: detach if shared
    char& at(size_t i) {
        detach();
        return (*data_)[i];
    }

    void append(const char* s) {
        detach();
        data_->append(s);
    }

    const char* c_str() const { return data_->c_str(); }
    size_t size() const { return data_->size(); }
    long use_count() const { return data_.use_count(); }

private:
    void detach() {
        if (data_.use_count() > 1) {
            data_ = std::make_shared<std::string>(*data_);  // Deep copy
        }
    }

    std::shared_ptr<std::string> data_;
};

int main() {
    CowString a("Hello, World!");
    CowString b = a;  // Cheap! Shared data
    std::cout << "After copy: a.refs=" << a.use_count()
              << " b.refs=" << b.use_count() << "\n";
    // After copy: a.refs=2 b.refs=2

    b.append(" Modified");  // Detach! b gets its own copy
    std::cout << "After modify: a.refs=" << a.use_count()
              << " b.refs=" << b.use_count() << "\n";
    // After modify: a.refs=1 b.refs=1

    std::cout << "a: " << a.c_str() << "\n";  // "Hello, World!"
    std::cout << "b: " << b.c_str() << "\n";  // "Hello, World! Modified"
    return 0;
}
```

After the copy, both `a` and `b` have a `use_count` of 2 because they point at the same `std::string`. The moment `b.append()` is called, `detach()` sees `use_count > 1`, allocates a fresh `std::string` with the same content, and replaces `b`'s pointer. Now they're independent and both have `use_count` of 1. The original `a` is completely unaffected.

### Q2: Build a thread-safe COW buffer with atomic reference counting

When you need `shared_ptr`'s reference counting semantics but want to control the exact memory layout (for example, to store size, capacity, and raw bytes in a single allocation), you can implement your own reference counting with `std::atomic<int>`. This is the approach historically used by Qt's `QByteArray` and similar libraries. It's more complex than the `shared_ptr` approach, but it gives you full control over the memory layout and eliminates the separate control-block allocation.

**Answer:**

```cpp
#include <atomic>
#include <cstring>
#include <algorithm>
#include <iostream>

class CowBuffer {
    struct Data {
        std::atomic<int> refcount{1};
        size_t size;
        size_t capacity;
        char payload[];  // Flexible array member (C-style, common in practice)

        static Data* create(size_t cap) {
            auto* d = static_cast<Data*>(::operator new(
                sizeof(Data) + cap));
            d->refcount.store(1, std::memory_order_relaxed);
            d->size = 0;
            d->capacity = cap;
            return d;
        }

        Data* clone() const {
            auto* d = create(capacity);
            d->size = size;
            std::memcpy(d->payload, payload, size);
            return d;
        }

        void release() {
            if (refcount.fetch_sub(1, std::memory_order_acq_rel) == 1)
                ::operator delete(this);
        }

        void acquire() {
            refcount.fetch_add(1, std::memory_order_relaxed);
        }
    };

    Data* data_;

    void detach() {
        if (data_->refcount.load(std::memory_order_acquire) > 1) {
            Data* new_data = data_->clone();
            data_->release();
            data_ = new_data;
        }
    }

public:
    explicit CowBuffer(size_t cap = 64)
        : data_(Data::create(cap)) {}

    ~CowBuffer() { data_->release(); }

    CowBuffer(const CowBuffer& o) : data_(o.data_) {
        data_->acquire();  // Cheap copy!
    }

    CowBuffer& operator=(const CowBuffer& o) {
        if (this != &o) {
            o.data_->acquire();
            data_->release();
            data_ = o.data_;
        }
        return *this;
    }

    // Read: no detach
    const char* data() const { return data_->payload; }
    size_t size() const { return data_->size; }

    // Write: detach first
    void write(const char* src, size_t n) {
        detach();
        std::memcpy(data_->payload + data_->size, src, n);
        data_->size += n;
    }
};
```

The `acq_rel` memory ordering on `fetch_sub` in `release()` is important and worth understanding. The `acq_rel` ensures that before the last decrement destroys the object, all previous writes to the object's data are visible to the thread doing the deletion - this prevents a race where the destructor runs before another thread's writes have propagated. A relaxed ordering on `fetch_add` in `acquire()` is safe because no shared data is being read at that point, only the counter is being bumped.

### Q3: Show COW for an immutable-externally, mutable-internally data structure

A COW vector lets you take O(1) snapshots. All snapshots share the underlying `std::vector` until any one of them writes. This is the foundation of persistent and functional data structures in C++ - the idea that "copying" a large collection should be nearly free until someone actually changes something.

**Answer:**

```cpp
#include <memory>
#include <vector>
#include <iostream>

// COW vector: snapshots are cheap, mutations detach
template<typename T>
class CowVector {
    std::shared_ptr<std::vector<T>> data_;

    void detach() {
        if (data_.use_count() > 1)
            data_ = std::make_shared<std::vector<T>>(*data_);
    }

public:
    CowVector() : data_(std::make_shared<std::vector<T>>()) {}

    // Read-only access - always safe, no detach
    const T& operator[](size_t i) const { return (*data_)[i]; }
    size_t size() const { return data_->size(); }
    auto begin() const { return data_->cbegin(); }
    auto end() const { return data_->cend(); }

    // Write access - detach on modify
    void push_back(const T& val) { detach(); data_->push_back(val); }
    void set(size_t i, const T& val) { detach(); (*data_)[i] = val; }
    void clear() { detach(); data_->clear(); }

    // Snapshot: O(1) copy for read-only sharing
    CowVector snapshot() const { return *this; }

    long ref_count() const { return data_.use_count(); }
};

int main() {
    CowVector<int> v;
    for (int i = 0; i < 1000; ++i) v.push_back(i);

    // Cheap snapshots for readers
    auto snap1 = v.snapshot();  // O(1)
    auto snap2 = v.snapshot();  // O(1)
    std::cout << "Refs: " << v.ref_count() << "\n";  // 3

    v.push_back(1000);  // Detaches v from snapshots
    std::cout << "After write: v.refs=" << v.ref_count()
              << " snap1.refs=" << snap1.ref_count() << "\n";
    // v.refs=1, snap1.refs=2 (snap1, snap2 still share)
    return 0;
}
```

After `v.push_back(1000)`, `v` detaches and now has a private copy, while `snap1` and `snap2` continue sharing the original 1000-element vector. The snapshots are unaffected, and `v` can keep growing independently. This is the pattern you'd use for an undo history: each "snapshot" is effectively free until the user makes a change.

---

## Notes

- COW was removed from `std::string` in C++11 - SSO (Small String Optimization) replaced it for short strings, and COW's thread-safety problems made it unattractive for the general case.
- `shared_ptr` provides atomic reference counting out of the box, making it the simplest way to implement COW without rolling your own.
- COW is excellent for configuration snapshots, undo history, and functional-style data structures where many readers share the same state.
- Thread safety caveat: `shared_ptr`'s reference count is atomic, but the pointed-to data is not. If two threads both hold COW copies and one writes while the other reads, you need external synchronization before the detach, not after it.
- The `detach()` check must use `use_count() > 1`, not `unique()` - `unique()` was deprecated in C++17 and removed in C++20.
- Qt's `QString`, `QByteArray`, and `QVector` all use implicit sharing (COW) via `QSharedDataPointer` - it's a proven, battle-tested pattern for GUI toolkit types.
