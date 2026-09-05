# Apply the copy-and-swap idiom for exception-safe assignment

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Copy-and-swap>  

---

## Topic Overview

The **copy-and-swap idiom** implements `operator=` with the **strong exception safety guarantee**: if an exception is thrown, the original object is left unchanged.

The reason this idiom is clever is that it turns a difficult problem (writing correct assignment with no state corruption on failure) into a problem you've already solved (copying and swapping). The copy constructor handles duplication; a `noexcept` swap handles the exchange. If the copy throws, `this` was never touched. If the copy succeeds, the swap can't throw, so the transfer completes atomically from the object's perspective.

### Pattern

The three-step sequence makes the safety guarantee obvious:

```cpp
operator=(T other)       <- Step 1: copy by value (may throw)
{                             If exception here, 'this' is untouched
    swap(*this, other);  <- Step 2: swap members (noexcept)
}                        <- Step 3: old data destroyed in 'other' (noexcept)
```

### Exception Safety Levels

| Level | Guarantee | Copy-and-swap? |
| --- | --- | --- |
| No-throw | Never fails | N/A |
| **Strong** | **Succeeds completely or no change** | **Yes** |
| Basic | Object valid but unspecified state | Manual operator= |
| None | May leave object broken | Naive operator= |

---

## Self-Assessment

### Q1: Implement `operator=` using copy-and-swap and verify strong exception safety

The key design decision is to take `other` **by value** rather than by `const&`. That single choice is what makes the idiom work: the copy (or move) happens during argument passing, before any modification to `this`, so an exception there leaves `*this` untouched:

```cpp
#include <algorithm>
#include <cstring>
#include <iostream>
#include <utility>

class String {
    char* data_;
    size_t size_;

public:
    // Constructor
    String(const char* s = "")
        : size_(std::strlen(s))
        , data_(new char[std::strlen(s) + 1]) {
        std::strcpy(data_, s);
    }

    // Copy constructor (may throw on allocation)
    String(const String& other)
        : size_(other.size_)
        , data_(new char[other.size_ + 1]) {  // may throw!
        std::strcpy(data_, other.data_);
    }

    // Move constructor (noexcept)
    String(String&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))
        , size_(std::exchange(other.size_, 0)) {}

    // Destructor
    ~String() { delete[] data_; }

    // Copy-and-swap assignment (handles both copy and move)
    String& operator=(String other) noexcept {  // Step 1: copy (may throw)
        swap(*this, other);                      // Step 2: swap (noexcept)
        return *this;                            // Step 3: old data dies with 'other'
    }

    // Friend swap (noexcept)
    friend void swap(String& a, String& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
    }

    const char* c_str() const { return data_ ? data_ : ""; }
    size_t size() const { return size_; }
};

int main() {
    String a("Hello");
    String b("World");

    std::cout << "Before: a=" << a.c_str() << " b=" << b.c_str() << '\n';

    a = b;  // copy-and-swap: copies b, swaps into a, destroys old a
    std::cout << "After copy: a=" << a.c_str() << '\n';

    a = String("Temp");  // move-and-swap: moves rvalue, swaps, destroys old
    std::cout << "After move: a=" << a.c_str() << '\n';

    a = a;  // self-assignment: copies a, swaps copy into a, destroys copy
    std::cout << "After self: a=" << a.c_str() << '\n';
}
// Expected output:
// Before: a=Hello b=World
// After copy: a=World
// After move: a=Temp
// After self: a=Temp
```

**Why this is strongly exception-safe:**

- The copy constructor (parameter `other`) runs **before** any modification to `this`.
- If `new char[]` throws, the parameter is never created, and `this` is untouched.
- `swap` and destructor are `noexcept`, so once the copy succeeds, everything completes.

### Q2: Explain why copy-and-swap handles self-assignment without an explicit check

This is one of the nicest properties of the idiom. Walk through what happens step by step when you write `a = a`:

```cpp
a = a;  (self-assignment)

Step 1: String other(a)  -> creates a COPY of a
        Now: 'this' points to original data
             'other' points to copy of original data

Step 2: swap(*this, other)  -> swaps pointers
        Now: 'this' points to copy (identical content)
             'other' points to original

Step 3: ~other() destroys old data

Result: 'a' has the same content as before. Correct!
```

**Why no explicit `if (this == &other)` check is needed:**

- The copy is made by value **before** any mutation. Even in self-assignment, the copy preserves the data.
- The swap is just pointer/size exchanges - no data is lost.
- Traditional manual `operator=` without copy-and-swap **does** need a self-assignment check to avoid `delete[] data_` before copying from `data_`.

The absence of the self-assignment check is not an oversight - it's a consequence of the design. The idiom is provably correct for self-assignment without it.

### Q3: Show the performance tradeoff of copy-and-swap vs manual check-then-copy

Copy-and-swap isn't free. The "always allocates" comment below is the key: even when `a = b` could reuse `a`'s existing buffer (because `b` is no larger), copy-and-swap allocates a new one. A manual implementation can detect that case and skip the allocation:

```cpp
#include <cstring>
#include <iostream>
#include <chrono>

class ManualString {
    char* data_;
    size_t size_;
public:
    ManualString(const char* s = "") : size_(std::strlen(s)), data_(new char[size_ + 1]) {
        std::strcpy(data_, s);
    }
    ManualString(const ManualString& o) : size_(o.size_), data_(new char[o.size_ + 1]) {
        std::strcpy(data_, o.data_);
    }
    ~ManualString() { delete[] data_; }

    // Manual operator= (basic exception safety only)
    ManualString& operator=(const ManualString& other) {
        if (this == &other) return *this;  // must check!

        char* new_data = new char[other.size_ + 1];  // allocate first
        std::strcpy(new_data, other.data_);
        delete[] data_;  // then release old
        data_ = new_data;
        size_ = other.size_;
        return *this;
    }

    const char* c_str() const { return data_; }
};

// Performance comparison:
// Copy-and-swap: ALWAYS allocates (even when sizes match)
// Manual: CAN reuse buffer if size is sufficient
//
// class OptimizedString {
//     ManualString& operator=(const ManualString& other) {
//         if (size_ >= other.size_) {  // reuse buffer!
//             std::strcpy(data_, other.data_);
//             size_ = other.size_;
//             return *this;
//         }
//         // ... allocate new buffer only if needed
//     }
// };

int main() {
    std::cout << "Copy-and-swap: Always allocates, but strongly exception-safe\n";
    std::cout << "Manual: Can reuse buffer, but basic exception safety only\n";
    std::cout << "Choice: Correctness (copy-and-swap) vs Performance (manual)\n";
}
```

**Comparison:**

| Aspect | Copy-and-Swap | Manual operator= |
| --- | --- | --- |
| Exception safety | Strong | Basic (if careful) |
| Self-assignment | Automatic | Must check manually |
| Allocation | Always allocates | Can reuse buffer |
| Code complexity | Simple (3 lines) | Complex (many cases) |
| Move assignment | Free (same overload) | Separate overload needed |

The standard library's own containers (`std::vector`, `std::string`) use hand-tuned manual assignment internally for exactly this reason - throughput matters more than code simplicity at that level. For almost everything else, copy-and-swap is the right default.

---

## Notes

- Accept the parameter **by value**, not by const reference - this handles both copy and move in one overload.
- `swap` must be `noexcept` for the idiom to provide strong exception safety.
- For types where allocation reuse matters (large buffers, frequent same-size assignment), manual operator= may be justified.
- Modern STL containers (vector, string) use optimized manual assignment internally, not copy-and-swap.
