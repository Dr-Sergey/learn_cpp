# Write Correct Move Constructors and Move Assignment Operators

**Category:** Move Semantics and Value Categories  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/move_constructor>  

---

## Topic Overview

### Move Constructor & Move Assignment - The Pattern

A move constructor's job is to steal resources from the source and leave the source in a valid, destructible state. Here's the canonical pattern using `std::exchange`, which atomically reads the old value and writes the new one - exactly what you need to nullify a pointer in one step:

```cpp
class Resource {
    T* ptr_;
    size_t size_;
public:
    // Move constructor: steal resources, nullify source
    Resource(Resource&& other) noexcept
        : ptr_(std::exchange(other.ptr_, nullptr))
        , size_(std::exchange(other.size_, 0)) {}

    // Move assignment: release ours, steal theirs
    Resource& operator=(Resource&& other) noexcept {
        if (this != &other) {              // Self-assignment guard
            delete[] ptr_;                  // Release current resources
            ptr_  = std::exchange(other.ptr_, nullptr);
            size_ = std::exchange(other.size_, 0);
        }
        return *this;
    }
};
```

### Critical Requirements

Every one of these matters. The `noexcept` requirement in particular surprises people - see Q3 for why skipping it has a real performance cost.

| Requirement | Why |
| --- | --- |
| **`noexcept`** | `std::vector` uses `move_if_noexcept` - without `noexcept`, it falls back to copy |
| **Nullify source** | Source must be destructible - dangling pointers cause double-free |
| **Self-assignment safety** | `x = std::move(x)` must not corrupt state |
| **Leave source valid** | Moved-from state must be destructible and assignable |

### `std::exchange` - The Key Helper

`std::exchange` is what makes move operations concise and correct. It reads the old value and writes the new one in a single expression, which avoids the two-step "save, then nullify" pattern that is easy to get wrong:

```cpp
// std::exchange(obj, new_val) returns old value, sets obj = new_val
ptr_ = std::exchange(other.ptr_, nullptr);
// Equivalent to:  ptr_ = other.ptr_;  other.ptr_ = nullptr;
```

---

## Self-Assessment

### Q1: Implement a move constructor that leaves the source in a valid but unspecified state

This example traces construction, moving, and destruction so you can verify that the moved-from object is safely destroyed without a double-free:

```cpp
#include <iostream>
#include <cstring>
#include <utility>
#include <algorithm>

class DynamicBuffer {
    char* data_;
    size_t size_;
    size_t capacity_;

public:
    // Regular constructor
    DynamicBuffer(const char* str = "")
        : size_(std::strlen(str))
        , capacity_(size_ + 1)
        , data_(new char[std::strlen(str) + 1])
    {
        std::copy_n(str, size_ + 1, data_);
        std::cout << "  Constructed: \"" << data_ << "\" (cap=" << capacity_ << ")\n";
    }

    // Copy constructor
    DynamicBuffer(const DynamicBuffer& other)
        : size_(other.size_)
        , capacity_(other.capacity_)
        , data_(new char[other.capacity_])
    {
        std::copy_n(other.data_, size_ + 1, data_);
        std::cout << "  Copy-constructed: \"" << data_ << "\"\n";
    }

    // MOVE CONSTRUCTOR - steal resources, leave source in valid empty state
    DynamicBuffer(DynamicBuffer&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))
        , size_(std::exchange(other.size_, 0))
        , capacity_(std::exchange(other.capacity_, 0))
    {
        std::cout << "  Move-constructed: \"" << (data_ ? data_ : "(null)") << "\"\n";
        // Source is now: data_=nullptr, size_=0, capacity_=0
        // This is VALID: destructor can safely call delete[] nullptr
    }

    // Destructor
    ~DynamicBuffer() {
        std::cout << "  Destroying: \""
                  << (data_ ? data_ : "(moved-from)") << "\"\n";
        delete[] data_;  // Safe even if nullptr
    }

    // Accessors
    const char* c_str() const { return data_ ? data_ : ""; }
    size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }
};

int main() {
    std::cout << "=== Move Constructor Demo ===\n\n";

    DynamicBuffer a("Hello, World!");

    std::cout << "\nMoving a -> b:\n";
    DynamicBuffer b(std::move(a));

    std::cout << "\nAfter move:\n";
    std::cout << "  a: \"" << a.c_str() << "\" (size=" << a.size()
              << ", empty=" << a.empty() << ")\n";
    std::cout << "  b: \"" << b.c_str() << "\" (size=" << b.size() << ")\n";

    std::cout << "\nDestroying both (no double-free!):\n";
    return 0;
}
// Expected output:
//   Constructed: "Hello, World!" (cap=14)
//   Moving a -> b:
//   Move-constructed: "Hello, World!"
//   After move:
//     a: "" (size=0, empty=1)
//     b: "Hello, World!" (size=13)
//   Destroying both (no double-free!):
//   Destroying: "Hello, World!"
//   Destroying: "(moved-from)"
```

The destructor reaching `delete[] nullptr` is perfectly safe - that's the invariant the move constructor establishes by nullifying `other.data_`.

### Q2: Explain why the move assignment operator must handle self-assignment

Self-assignment with move (`x = std::move(x)`) can happen in generic code, especially through algorithms. Without a guard, the object **destroys its own resources** before stealing them - use-after-free on the next line.

```cpp
#include <iostream>
#include <utility>
#include <algorithm>
#include <numeric>

class Buffer {
    int* data_;
    size_t size_;
public:
    Buffer(size_t n) : data_(new int[n]), size_(n) {
        std::iota(data_, data_ + n, 0);
    }

    // BUGGY move assignment - no self-assignment check
    Buffer& assign_buggy(Buffer&& other) noexcept {
        delete[] data_;                          // Deletes OUR data...
        data_ = std::exchange(other.data_, nullptr);  // ...which IS other.data_!
        size_ = std::exchange(other.size_, 0);
        return *this;
    }

    // CORRECT move assignment - with self-assignment guard
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {  // Guard against self-assignment
            delete[] data_;
            data_ = std::exchange(other.data_, nullptr);
            size_ = std::exchange(other.size_, 0);
        }
        return *this;
    }

    // Alternative: swap-based (inherently self-assignment safe)
    Buffer& assign_swap(Buffer&& other) noexcept {
        std::swap(data_, other.data_);
        std::swap(size_, other.size_);
        return *this;
        // other's destructor cleans up our old data
    }

    void print() const {
        std::cout << "  [";
        for (size_t i = 0; i < size_ && i < 5; ++i)
            std::cout << (i ? ", " : "") << data_[i];
        if (size_ > 5) std::cout << ", ...";
        std::cout << "] (size=" << size_ << ")\n";
    }

    ~Buffer() { delete[] data_; }

    // Delete copy for simplicity
    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;
    Buffer(Buffer&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))
        , size_(std::exchange(other.size_, 0)) {}
};

int main() {
    std::cout << "=== Self-Assignment in Move Assignment ===\n\n";

    // Scenario: self-move can happen in algorithms
    Buffer b(5);
    std::cout << "Before: ";
    b.print();

    // This is UB without the guard:
    // b.assign_buggy(std::move(b));
    // data_ is deleted, then we read from the deleted pointer!

    // Safe with the guard:
    b = std::move(b);
    std::cout << "After self-move (guarded): ";
    b.print();

    std::cout << "\n=== When self-move happens ===\n";
    std::cout << "  - std::swap internally: a = std::move(b) where a == b\n";
    std::cout << "  - Algorithm with iterator aliasing\n";
    std::cout << "  - Generic code: move(container[i]) into container[j] where i==j\n";

    return 0;
}
```

The swap-based alternative (`assign_swap`) is worth knowing: swapping pointers is inherently safe under self-assignment because you're just exchanging values with yourself, and the old data gets cleaned up by the other object's destructor.

### Q3: Show a move constructor that is NOT `noexcept` and how this prevents use in `std::vector` reallocation

This is a concrete demonstration of why the `noexcept` rule matters. When `std::vector` needs to grow its buffer, it uses `std::move_if_noexcept` internally. If your move constructor might throw, the vector has no way to roll back a half-completed reallocation and still satisfy the strong exception guarantee - so it falls back to copying every element instead.

```cpp
#include <iostream>
#include <vector>
#include <string>

class SafeWidget {
    std::string name_;
public:
    SafeWidget(std::string n) : name_(std::move(n)) {}

    // noexcept move -> vector WILL use this during reallocation
    SafeWidget(SafeWidget&& other) noexcept : name_(std::move(other.name_)) {
        std::cout << "  MOVE: " << name_ << "\n";
    }
    SafeWidget(const SafeWidget& other) : name_(other.name_) {
        std::cout << "  COPY: " << name_ << "\n";
    }
};

class UnsafeWidget {
    std::string name_;
public:
    UnsafeWidget(std::string n) : name_(std::move(n)) {}

    // NO noexcept -> vector will FALL BACK TO COPY during reallocation!
    // Reason: if move throws mid-reallocation, original data is corrupted
    // and vector can't provide strong exception guarantee
    UnsafeWidget(UnsafeWidget&& other)  // Missing noexcept!
        : name_(std::move(other.name_)) {
        std::cout << "  MOVE: " << name_ << "\n";
    }
    UnsafeWidget(const UnsafeWidget& other) : name_(other.name_) {
        std::cout << "  COPY: " << name_ << "\n";
    }
};

int main() {
    std::cout << "=== noexcept move -> vector uses MOVE ===\n";
    {
        std::vector<SafeWidget> v;
        v.reserve(2);
        v.emplace_back("A");
        v.emplace_back("B");
        std::cout << "  Triggering reallocation (capacity 2->4):\n";
        v.emplace_back("C");  // Reallocates -> moves A and B
    }

    std::cout << "\n=== Non-noexcept move -> vector uses COPY ===\n";
    {
        std::vector<UnsafeWidget> v;
        v.reserve(2);
        v.emplace_back("A");
        v.emplace_back("B");
        std::cout << "  Triggering reallocation (capacity 2->4):\n";
        v.emplace_back("C");  // Reallocates -> COPIES A and B (not moves!)
    }

    // Check at compile time
    std::cout << "\n=== Compile-time check ===\n";
    std::cout << "  SafeWidget move noexcept:   "
              << std::is_nothrow_move_constructible_v<SafeWidget> << "\n";     // 1
    std::cout << "  UnsafeWidget move noexcept: "
              << std::is_nothrow_move_constructible_v<UnsafeWidget> << "\n";   // 0

    // vector uses std::move_if_noexcept internally:
    // If move ctor is noexcept -> returns T&& (move)
    // If move ctor may throw   -> returns const T& (copy for safety)

    return 0;
}
// Expected output (showing the critical difference):
//   === noexcept move -> vector uses MOVE ===
//     Triggering reallocation:
//     MOVE: A
//     MOVE: B
//
//   === Non-noexcept move -> vector uses COPY ===
//     Triggering reallocation:
//     COPY: A
//     COPY: B
```

The output difference is the whole story. Two identical-looking classes, one `noexcept`, one not - and the vector silently copies the entire buffer content every time it grows, for the "safe" one. You can verify this at compile time with `std::is_nothrow_move_constructible_v` if you want to catch the mistake before it becomes a performance mystery.

---

## Notes

- **Always mark move operations `noexcept`** - `std::vector` and other containers depend on it.
- Use `std::exchange` to atomically read-and-nullify in move operations.
- Moved-from objects must be in a valid state: at minimum destructible and assignable.
- Self-assignment in move (`x = std::move(x)`) must be safe - use `if (this != &other)` or swap.
- If you can, follow the Rule of Zero - let compiler-generated moves handle everything.
