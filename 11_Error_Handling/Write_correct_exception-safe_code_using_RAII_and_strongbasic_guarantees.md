# Write correct exception-safe code using RAII and strong/basic guarantees

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/exceptions>  

---

## Topic Overview

Exception safety is about maintaining program invariants when an exception is thrown. It's not just "does the code catch exceptions?" but "what state is the program in after an exception?" The C++ community defines three levels of exception safety guarantees, and **RAII** is the primary technique for achieving them.

### The Three Guarantees

| Guarantee | Invariant | Analogy |
| --- | --- | --- |
| **No-throw** | Operation never throws. Marked `noexcept`. | Transaction that always succeeds |
| **Strong** (commit-or-rollback) | If operation fails, state is as if it never started. | Database transaction with rollback |
| **Basic** | If operation fails, no resources leak and invariants hold, but state may differ. | File partially written but consistent |

The mental model that helps most here is the database analogy. A no-throw operation is one you can call with complete confidence. A strong-guarantee operation is like a database transaction with rollback: either everything committed, or nothing changed. A basic-guarantee operation just promises you are not leaking memory or leaving the object in an internally broken state - but it may have partially modified the data:

```cpp
No-throw (strongest):    always succeeds        -> destructors, swap, move ops
Strong (transactional):  success = new state    -> copy-and-swap assignment
                         failure = old state
Basic (minimum bar):     no leaks               -> most STL operations
                         invariants hold
                         state may change
No guarantee:            anything goes          -> NEVER acceptable in C++
```

### RAII: The Foundation of Exception Safety

RAII is the reason C++ can have exception safety at all. When an exception unwinds the stack, destructors for all local objects run automatically. If those destructors release resources, there is no leak:

```cpp
// Without RAII: resource leak if bar() throws
void bad() {
    int* p = new int(42);
    bar();    // throws! -> p is LEAKED
    delete p; // never reached
}

// With RAII: automatic cleanup
void good() {
    auto p = std::make_unique<int>(42);
    bar();    // throws! -> p's destructor runs -> memory freed
}   // p destroyed here in both success and exception paths
```

**RAII in Practice:**

| Resource | RAII Wrapper |
| --- | --- |
| Heap memory | `unique_ptr`, `shared_ptr` |
| File handle | `std::fstream`, custom RAII wrapper |
| Mutex lock | `std::lock_guard`, `std::unique_lock` |
| Database transaction | Custom `ScopedTransaction` |
| Any cleanup action | `scope_exit` (C++26 / gsl::finally) |

---

## Self-Assessment

### Q1: Explain the three exception safety guarantees: no-throw, strong, and basic

**Solution - Concrete Examples of Each Guarantee:**

Each of the three member functions below is designed to illustrate one guarantee level. Read the comments to understand what exactly is and is not promised at each level:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <memory>
#include <stdexcept>

class Widget {
    std::string name_;
    std::vector<int> data_;

public:
    Widget(std::string name, std::vector<int> data)
        : name_(std::move(name)), data_(std::move(data)) {}

    // ===== NO-THROW GUARANTEE =====
    // Swap never throws - marked noexcept
    void swap(Widget& other) noexcept {
        name_.swap(other.name_);   // std::string::swap is noexcept
        data_.swap(other.data_);   // std::vector::swap is noexcept
    }

    // Accessors should be noexcept
    const std::string& name() const noexcept { return name_; }
    size_t size() const noexcept { return data_.size(); }

    // ===== STRONG GUARANTEE =====
    // Copy-and-swap: either fully succeeds or nothing changes
    Widget& operator=(Widget other) {  // copy happens HERE (may throw)
        swap(other);                    // noexcept - just swaps
        return *this;                   // old value destroyed in 'other'
    }
    // If copy constructor throws, swap never runs -> *this unchanged

    // Strong guarantee: add element only if validation passes
    void add_validated(int value) {
        if (value < 0)
            throw std::invalid_argument("negative value");
        // vector::push_back provides strong guarantee
        data_.push_back(value);
    }

    // ===== BASIC GUARANTEE =====
    // State may change, but no leaks and invariants hold
    void load_from(const std::vector<int>& source) {
        data_.clear();                          // point of no return
        for (int v : source) {
            data_.push_back(v);                 // may throw (allocation)
            // If push_back throws here: data_ has SOME elements (not all)
            // but data_ is still a valid vector (basic guarantee)
        }
    }

    void print() const {
        std::cout << name_ << ": [";
        for (size_t i = 0; i < data_.size(); ++i)
            std::cout << (i ? ", " : "") << data_[i];
        std::cout << "]\n";
    }
};

int main() {
    Widget w1("alpha", {1, 2, 3});
    Widget w2("beta", {10, 20});

    // No-throw: swap
    w1.swap(w2);
    w1.print();  // beta: [10, 20]
    w2.print();  // alpha: [1, 2, 3]

    // Strong: assignment (copy-and-swap)
    w1 = w2;    // if copy fails, w1 unchanged
    w1.print(); // alpha: [1, 2, 3]

    // Strong: validated add
    try {
        w1.add_validated(-5);
    } catch (const std::invalid_argument& e) {
        std::cout << "Rejected: " << e.what() << "\n";
        w1.print();  // unchanged - strong guarantee
    }
}
// Expected output:
//   beta: [10, 20]
//   alpha: [1, 2, 3]
//   alpha: [1, 2, 3]
//   Rejected: negative value
//   alpha: [1, 2, 3]
```

**Which Standard Library Operations Provide Which Guarantee:**

```cpp
No-throw:    swap, destructors, clear, size, empty, begin/end
Strong:      push_back, insert (single), emplace_back
Basic:       multi-element insert, assign, resize
             (state valid but maybe partially modified)
```

---

### Q2: Implement a copy-and-swap idiom that provides the strong exception safety guarantee

**Solution - Copy-and-Swap with RAII:**

The copy-and-swap idiom is the canonical way to write an assignment operator with a strong guarantee. The trick is that the copy (which can fail) happens before any modification to `*this`. Only after the copy succeeds does the noexcept `swap` run, atomically exchanging the state. If the copy throws, `*this` was never touched:

```cpp
#include <iostream>
#include <algorithm>
#include <utility>
#include <cstring>
#include <stdexcept>

class DynamicArray {
    int* data_ = nullptr;
    size_t size_ = 0;

public:
    DynamicArray() = default;

    explicit DynamicArray(size_t n, int value = 0)
        : data_(n ? new int[n] : nullptr), size_(n)
    {
        std::fill(data_, data_ + size_, value);
    }

    // Copy constructor - may throw (allocation)
    DynamicArray(const DynamicArray& other)
        : data_(other.size_ ? new int[other.size_] : nullptr)
        , size_(other.size_)
    {
        std::copy(other.data_, other.data_ + other.size_, data_);
    }

    // Move constructor - noexcept
    DynamicArray(DynamicArray&& other) noexcept
        : data_(other.data_), size_(other.size_)
    {
        other.data_ = nullptr;
        other.size_ = 0;
    }

    // Destructor - noexcept
    ~DynamicArray() { delete[] data_; }

    // Swap - noexcept (critical for copy-and-swap)
    friend void swap(DynamicArray& a, DynamicArray& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
    }

    // COPY-AND-SWAP ASSIGNMENT - strong guarantee
    DynamicArray& operator=(DynamicArray other) {  // pass BY VALUE -> copy here
        swap(*this, other);                         // noexcept swap
        return *this;                               // 'other' destroys old data
    }
    // How it works:
    // 1. 'other' is constructed via copy (may throw - but *this untouched)
    // 2. swap is noexcept - exchanges data pointers
    // 3. 'other' destructor frees old *this data
    // If step 1 throws -> *this is UNCHANGED (strong guarantee!)
    // Also handles self-assignment correctly (copies then swaps)
    // Also handles move assignment (compiler uses move ctor for 'other')

    int& operator[](size_t i) { return data_[i]; }
    size_t size() const noexcept { return size_; }

    void print() const {
        std::cout << "[";
        for (size_t i = 0; i < size_; ++i)
            std::cout << (i ? ", " : "") << data_[i];
        std::cout << "]" << " (size=" << size_ << ")\n";
    }
};

int main() {
    DynamicArray a(5, 1);
    DynamicArray b(3, 9);

    std::cout << "Before assignment:\n";
    std::cout << "a: "; a.print();
    std::cout << "b: "; b.print();

    // Copy assignment (uses copy-and-swap)
    a = b;
    std::cout << "\nAfter a = b:\n";
    std::cout << "a: "; a.print();
    std::cout << "b: "; b.print();

    // Move assignment (compiler uses move ctor for 'other' parameter)
    a = DynamicArray(4, 7);
    std::cout << "\nAfter a = DynamicArray(4, 7):\n";
    std::cout << "a: "; a.print();

    // Self-assignment (safe with copy-and-swap)
    a = a;
    std::cout << "\nAfter a = a (self-assignment):\n";
    std::cout << "a: "; a.print();
}
// Expected output:
//   Before assignment:
//   a: [1, 1, 1, 1, 1] (size=5)
//   b: [9, 9, 9] (size=3)
//
//   After a = b:
//   a: [9, 9, 9] (size=3)
//   b: [9, 9, 9] (size=3)
//
//   After a = DynamicArray(4, 7):
//   a: [7, 7, 7, 7] (size=4)
//
//   After a = a (self-assignment):
//   a: [7, 7, 7, 7] (size=4)
```

---

### Q3: Show how a function that calls two throwing operations can leak if not written carefully

**Solution - Double-Allocation Leak Scenario:**

The reason this trips people up is that the leak looks harmless at first glance - `delete a` and `delete b` are right there in the code. But if the second allocation throws, those lines are never reached. RAII closes this gap because the destructor runs regardless of how you exit the scope:

```cpp
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>

struct Resource {
    std::string name;
    Resource(const std::string& n) : name(n) {
        std::cout << "  Acquired: " << name << "\n";
    }
    ~Resource() {
        std::cout << "  Released: " << name << "\n";
    }
};

// Simulates a function that may throw
Resource* create_resource(const std::string& name, bool should_fail) {
    if (should_fail)
        throw std::runtime_error("Failed to create " + name);
    return new Resource(name);
}

// BAD: leak if second allocation throws
void bad_approach() {
    std::cout << "=== BAD: raw pointers ===\n";
    Resource* a = create_resource("A", false);  // succeeds: A allocated
    Resource* b = create_resource("B", true);   // throws! -> A is LEAKED

    // These lines never execute:
    delete b;
    delete a;
}

// ALSO BAD: even "fixed" code can leak with temporaries
void also_bad(Resource* a, Resource* b);
void call_also_bad() {
    // Pre-C++17: order of evaluation is unspecified!
    // If create_resource("A") runs first, then create_resource("B") throws -> A leaks
    also_bad(create_resource("A", false),
             create_resource("B", true));
}

// GOOD: RAII with unique_ptr
void good_approach() {
    std::cout << "\n=== GOOD: unique_ptr ===\n";
    auto a = std::unique_ptr<Resource>(create_resource("A", false));
    auto b = std::unique_ptr<Resource>(create_resource("B", true));  // throws!
    // a's destructor runs -> A is properly released
}

// BEST: make_unique (avoids the temporary raw pointer entirely)
void best_approach() {
    std::cout << "\n=== BEST: make_unique ===\n";
    // Can't use make_unique with create_resource, but for regular types:
    auto a = std::make_unique<Resource>("A");
    auto b = std::make_unique<Resource>("B");
}

// GOOD: multi-resource transaction with strong guarantee
class Transaction {
    std::unique_ptr<Resource> res_a_;
    std::unique_ptr<Resource> res_b_;
public:
    Transaction() {
        // Each resource managed immediately upon creation
        auto a = std::make_unique<Resource>("TxnA");
        auto b = std::make_unique<Resource>("TxnB");
        // Both succeeded - commit by moving
        res_a_ = std::move(a);
        res_b_ = std::move(b);
    }
};

int main() {
    try { bad_approach(); }
    catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << "\n";
        std::cout << "  <- Resource A was LEAKED!\n";
    }

    try { good_approach(); }
    catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << "\n";
        std::cout << "  <- Resource A was properly cleaned up\n";
    }
}
// Expected output:
//   === BAD: raw pointers ===
//     Acquired: A
//     Caught: Failed to create B
//     <- Resource A was LEAKED!
//
//   === GOOD: unique_ptr ===
//     Acquired: A
//     Released: A
//     Caught: Failed to create B
//     <- Resource A was properly cleaned up
```

**The key principle:**

```cpp
RULE: Every resource must be owned by an RAII object
      from the MOMENT it is acquired.

// BAD: gap between acquire and exception safety
Resource* p = acquire();   // if use() throws -> p leaked
use(p);

// GOOD: owned immediately
auto p = make_unique<R>(); // if use() throws -> p cleaned up
use(*p);
```

---

## Notes

- **"Exception-safe" is not "uses try/catch"** - exception safety comes from RAII, not from catching exceptions everywhere.
- **Destructors should be `noexcept`** - a throwing destructor during stack unwinding calls `std::terminate`.
- **Move operations should be `noexcept`** - enables `std::vector` to use moves during reallocation (strong guarantee).
- **Copy-and-swap is the canonical pattern** for strong-guarantee assignment operators.
- **`std::vector::push_back` provides the strong guarantee** - if reallocation fails, the vector is unchanged. This requires `noexcept` move constructors; otherwise it falls back to copying.
- **Abrahams Guarantee Levels** (David Abrahams formalized these) are the standard vocabulary for discussing exception safety in C++.
