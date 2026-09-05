# Use std::exchange for implementing move semantics cleanly

**Category:** Best Practices and Idioms  
**Standard:** C++14  
**Reference:** <https://en.cppreference.com/w/cpp/utility/exchange>  

---

## Topic Overview

`std::exchange(obj, new_val)` replaces `obj` with `new_val` and returns the **old** value of `obj` - all in one expression. This makes move constructors and move assignment operators much cleaner because the two steps of a move (take the value, then null out the source) collapse into a single readable expression.

```cpp
#include <utility>
int old = std::exchange(x, 0);  // old = x, then x = 0
```

---

## Self-Assessment

### Q1: Implement a move constructor using `std::exchange`

A move constructor needs to do two things: take ownership of the source's resources, and leave the source in a valid-but-empty state. Without `std::exchange`, that means writing the take and the null-out as two separate statements for every member. With `std::exchange`, both happen together in the initializer list.

```cpp
#include <iostream>
#include <utility>
#include <cstddef>

class Buffer {
    int* data_;
    size_t size_;
public:
    explicit Buffer(size_t n)
        : data_(new int[n]{}), size_(n) {}

    ~Buffer() { delete[] data_; }

    // Move constructor - clean with std::exchange
    Buffer(Buffer&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))  // take pointer, set source to null
        , size_(std::exchange(other.size_, 0))         // take size, set source to 0
    {}

    // Move assignment
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = std::exchange(other.data_, nullptr);
            size_ = std::exchange(other.size_, 0);
        }
        return *this;
    }

    // Delete copy
    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;

    size_t size() const { return size_; }
    bool empty() const { return data_ == nullptr; }
};

int main() {
    Buffer a(100);
    std::cout << "a.size = " << a.size() << ", empty = " << a.empty() << '\n';

    Buffer b(std::move(a));
    std::cout << "After move:\n";
    std::cout << "a.size = " << a.size() << ", empty = " << a.empty() << '\n';
    std::cout << "b.size = " << b.size() << ", empty = " << b.empty() << '\n';
}
// Expected output:
// a.size = 100, empty = 0
// After move:
// a.size = 0, empty = 1
// b.size = 100, empty = 0
```

After the move, `a` is left in a valid state (size 0, null pointer), so its destructor can safely call `delete[] nullptr`, which is a no-op. That is exactly the postcondition a moved-from object needs.

### Q2: Show that `std::exchange` returns the old value

`std::exchange` is not limited to move semantics. Any time you need to swap a value in place and also use the old value in the same expression, it fits. Here are a few patterns where it shines:

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <list>

int main() {
    // Basic usage: returns old, assigns new
    int x = 42;
    int old = std::exchange(x, 0);
    std::cout << "old=" << old << " x=" << x << '\n';  // old=42 x=0

    // With strings
    std::string name = "Alice";
    std::string prev = std::exchange(name, "Bob");
    std::cout << "prev=" << prev << " name=" << name << '\n';  // prev=Alice name=Bob

    // Cycling through values (rotate)
    int a = 1, b = 2, c = 3;
    a = std::exchange(b, std::exchange(c, a));  // rotate a->c->b->a
    std::cout << a << " " << b << " " << c << '\n';  // 2 3 1

    // Linked list traversal
    struct Node { int val; Node* next; };
    Node n3{3, nullptr}, n2{2, &n3}, n1{1, &n2};
    Node* curr = &n1;
    while (curr) {
        std::cout << curr->val << ' ';
        curr = std::exchange(curr->next, nullptr);  // advance and disconnect
    }
    std::cout << '\n';
}
// Expected output:
// old=42 x=0
// prev=Alice name=Bob
// 2 3 1
// 1 2
```

The linked-list traversal pattern is worth noticing: `std::exchange(curr->next, nullptr)` advances `curr` while simultaneously disconnecting the node from the rest of the list - both in one expression. This pattern appears in move constructors and destructors for intrusive data structures.

### Q3: Compare `std::exchange` with manual temp variable pattern

The reason `std::exchange` reduces bug risk is subtle: it forces you to name the new value at the same time you read the old one. With the manual three-step approach you can forget the reset step, especially when adding a new member to a class later. The `std::exchange` form makes the reset structurally impossible to omit.

```cpp
#include <iostream>
#include <utility>

class Handle {
    int fd_;
public:
    explicit Handle(int fd) : fd_(fd) {}
    ~Handle() { if (fd_ >= 0) std::cout << "close(" << fd_ << ")\n"; }

    // WITHOUT std::exchange (verbose, error-prone)
    Handle(Handle&& other) noexcept {
        fd_ = other.fd_;        // take
        other.fd_ = -1;         // reset source
        // Two separate steps - easy to forget the reset!
    }

    // WITH std::exchange (one-liner, atomic intent)
    // Handle(Handle&& other) noexcept
    //     : fd_(std::exchange(other.fd_, -1)) {}  // take + reset in one expression

    Handle& operator=(Handle&& other) noexcept {
        if (this != &other)
            fd_ = std::exchange(other.fd_, -1);
        return *this;
    }

    Handle(const Handle&) = delete;
    Handle& operator=(const Handle&) = delete;

    int get() const { return fd_; }
};

int main() {
    Handle h1(42);
    Handle h2(std::move(h1));
    std::cout << "h1=" << h1.get() << " h2=" << h2.get() << '\n';
}
// Expected output:
// h1=-1 h2=42
// close(42)
```

Comparing the two styles side by side:

| Pattern | Lines | Clarity | Bug risk |
| --- | --- | --- | --- |
| `tmp = x; x = new; use tmp;` | 3 | Low | Forget reset |
| `std::exchange(x, new)` | 1 | High | Minimal |

---

## Notes

- `std::exchange` is in `<utility>` since C++14.
- It's `constexpr` since C++20.
- Use it in move constructors, move assignment, linked-list iteration, state machines.
- It does NOT do an atomic exchange - use `std::atomic::exchange` for that.
