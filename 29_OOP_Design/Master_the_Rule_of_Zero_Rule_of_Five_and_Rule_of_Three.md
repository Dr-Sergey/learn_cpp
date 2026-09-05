# Master the Rule of Zero, Rule of Five, and Rule of Three

**Category:** OOP Design  
**Standard:** C++11  

---

## Topic Overview

These rules tell you when and how to write the "special member functions" - the constructor, destructor, copy, and move operations. The reason they matter is that C++ gives you implicit implementations of all five, but those implicit implementations only do the right thing if your class is written a certain way. The rules help you decide whether to trust the compiler or take control yourself.

The short version: if your class manages a raw resource directly, you must define all five special members yourself. If it does not, define none of them and let the compiler do its job.

### Decision Flowchart

```cpp
Does your class directly manage a resource (raw pointer, fd, handle)?
  │
  ├─ NO  -> RULE OF ZERO: Don't declare ANY special members.
  │        Let compiler generate them. Use smart pointers, std::string,
  │        std::vector as members - they handle everything.
  │
  └─ YES -> Do you need copying?
            │
            ├─ YES -> RULE OF FIVE: Define ALL five:
            │        destructor, copy ctor, copy assign, move ctor, move assign
            │
            └─ NO  -> RULE OF FIVE (delete copy):
                     Define destructor + move ctor + move assign.
                     Delete copy ctor + copy assign.
```

### The Five Special Members

| Member | Signature | When called |
| --- | --- | --- |
| Destructor | `~T()` | Object lifetime ends |
| Copy constructor | `T(const T&)` | `T b = a;` |
| Copy assignment | `T& operator=(const T&)` | `b = a;` |
| Move constructor | `T(T&&) noexcept` | `T b = std::move(a);` |
| Move assignment | `T& operator=(T&&) noexcept` | `b = std::move(a);` |

---

## Self-Assessment

### Q1: Demonstrate Rule of Zero with proper member types

The Rule of Zero is the goal you should be aiming for in nearly every class you write. The idea is that if your members are all "well-behaved" types - `std::string`, `std::vector`, `std::unique_ptr`, `std::shared_ptr` - then the compiler can generate correct copy, move, and destruction semantics automatically. You declare nothing and everything works.

```cpp
#include <string>
#include <vector>
#include <memory>

// RULE OF ZERO: No special members needed
// ALL members have correct copy/move/destroy semantics built in
class UserProfile {
    std::string name_;
    std::string email_;
    std::vector<std::string> tags_;
    std::shared_ptr<const Config> config_;  // Shared ownership

    // NO destructor, NO copy ctor, NO move ctor, NO assignment operators
    // Compiler generates ALL of them correctly!
public:
    UserProfile(std::string name, std::string email)
        : name_(std::move(name)), email_(std::move(email)) {}

    void add_tag(std::string tag) { tags_.push_back(std::move(tag)); }
};

// Even with unique ownership - Rule of Zero still works:
class Document {
    std::string title_;
    std::unique_ptr<std::string> content_;  // Unique ownership

    // Compiler generates:
    // - Destructor: unique_ptr deletes content
    // - Move ctor/assign: unique_ptr moves correctly
    // - Copy ctor/assign: DELETED (because unique_ptr is non-copyable)
    //   This is the CORRECT behavior for unique ownership!
public:
    Document(std::string title, std::string content)
        : title_(std::move(title))
        , content_(std::make_unique<std::string>(std::move(content))) {}
};

// Rule of Zero is the DEFAULT. 90%+ of classes should use it.
```

Notice the `Document` class: because `unique_ptr` is non-copyable, the compiler automatically deletes copy operations for `Document` too. You get the right behavior for free - and if someone tries to copy a `Document`, they get a clear compile error rather than a silent shallow copy.

### Q2: Implement Rule of Five for a class managing a raw resource

The Rule of Five kicks in when you are managing a raw resource directly - a raw pointer you `new` and `delete`, a file descriptor, a socket handle. The reason you must define all five is that they are deeply interdependent: if your destructor does `delete[] data_`, then your copy constructor must allocate a fresh copy, your move constructor must null out the source's pointer, and so on. Leaving any one of them to the compiler will give you disaster.

```cpp
#include <cstring>
#include <utility>
#include <iostream>
#include <algorithm>

// RULE OF FIVE: Manual resource management
class DynamicBuffer {
    char* data_;
    size_t size_;
    size_t capacity_;

public:
    // Constructor
    explicit DynamicBuffer(size_t initial_cap = 64)
        : data_(new char[initial_cap]), size_(0), capacity_(initial_cap) {}

    // 1. DESTRUCTOR
    ~DynamicBuffer() {
        delete[] data_;
    }

    // 2. COPY CONSTRUCTOR (deep copy)
    DynamicBuffer(const DynamicBuffer& other)
        : data_(new char[other.capacity_])
        , size_(other.size_)
        , capacity_(other.capacity_)
    {
        std::memcpy(data_, other.data_, size_);
    }

    // 3. COPY ASSIGNMENT (copy-and-swap idiom - exception safe!)
    DynamicBuffer& operator=(const DynamicBuffer& other) {
        if (this != &other) {
            DynamicBuffer tmp(other);     // Copy construct temp
            swap(*this, tmp);              // Swap with temp
        }                                  // Temp destroyed with old data
        return *this;
    }

    // 4. MOVE CONSTRUCTOR (steal resources, noexcept!)
    DynamicBuffer(DynamicBuffer&& other) noexcept
        : data_(other.data_)
        , size_(other.size_)
        , capacity_(other.capacity_)
    {
        other.data_ = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;
    }

    // 5. MOVE ASSIGNMENT (noexcept!)
    DynamicBuffer& operator=(DynamicBuffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;                 // Release current
            data_ = other.data_;            // Steal
            size_ = other.size_;
            capacity_ = other.capacity_;
            other.data_ = nullptr;          // Leave other in valid empty state
            other.size_ = 0;
            other.capacity_ = 0;
        }
        return *this;
    }

    // Swap helper (for copy-and-swap)
    friend void swap(DynamicBuffer& a, DynamicBuffer& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
        swap(a.capacity_, b.capacity_);
    }

    void append(const char* str, size_t len) {
        if (size_ + len > capacity_) {
            size_t new_cap = std::max(capacity_ * 2, size_ + len);
            char* new_data = new char[new_cap];
            std::memcpy(new_data, data_, size_);
            delete[] data_;
            data_ = new_data;
            capacity_ = new_cap;
        }
        std::memcpy(data_ + size_, str, len);
        size_ += len;
    }

    size_t size() const { return size_; }
};

// BETTER: Convert to Rule of Zero
#include <vector>

class BetterBuffer {
    std::vector<char> data_;  // vector handles ALL resource management!
public:
    void append(const char* str, size_t len) {
        data_.insert(data_.end(), str, str + len);
    }
    size_t size() const { return data_.size(); }
    // NO special members needed - Rule of Zero!
};
```

The `BetterBuffer` at the end is the real lesson: every time you find yourself writing a Rule of Five class, ask whether you could wrap the resource in a smart pointer or standard container and then rely on the Rule of Zero instead. The answer is yes more often than you think.

### Q3: Show the "Rule of Four and a Half" (copy-and-swap idiom)

The copy-and-swap idiom is a clever trick that lets you write a single assignment operator that handles *both* copy and move assignment correctly. The "four and a half" refers to: destructor + copy ctor + move ctor + one unified `operator=` (the four) + the `swap` helper (the half).

The key insight is that the assignment operator takes its parameter by *value*, not by reference. That one change means the compiler will call the copy constructor or move constructor on the argument before the function body even runs - depending on whether the caller passed an lvalue or an rvalue. You just swap and return.

```cpp
#include <utility>
#include <algorithm>

// Rule of 4.5: Unified assignment with swap
class Image {
    int* pixels_;
    int width_, height_;

public:
    Image(int w, int h)
        : pixels_(new int[w * h]()), width_(w), height_(h) {}

    ~Image() { delete[] pixels_; }

    // Copy constructor
    Image(const Image& other)
        : pixels_(new int[other.width_ * other.height_])
        , width_(other.width_), height_(other.height_)
    {
        std::copy_n(other.pixels_, width_ * height_, pixels_);
    }

    // Move constructor
    Image(Image&& other) noexcept
        : pixels_(other.pixels_), width_(other.width_), height_(other.height_)
    {
        other.pixels_ = nullptr;
        other.width_ = other.height_ = 0;
    }

    // THE TRICK: ONE assignment operator handles BOTH copy AND move!
    // Takes parameter BY VALUE (triggers copy ctor or move ctor depending on argument)
    Image& operator=(Image other) noexcept {  // <-- by value!
        swap(*this, other);
        return *this;
    }

    friend void swap(Image& a, Image& b) noexcept {
        using std::swap;
        swap(a.pixels_, b.pixels_);
        swap(a.width_, b.width_);
        swap(a.height_, b.height_);
    }

    // Special members count: destructor + copy ctor + move ctor + unified operator=
    // = 4 declarations + swap (the "half") = Rule of 4.5
};

// Usage:
int main() {
    Image a(100, 100);
    Image b = a;              // Copy ctor
    Image c = std::move(a);   // Move ctor
    b = c;                    // operator=(Image other) - copy ctor invoked for `other`
    b = std::move(c);         // operator=(Image other) - move ctor invoked for `other`
    return 0;
}
```

**Which rule to use - decision table:**

| Your class has... | Rule | Action |
| --- | --- | --- |
| Only std::string, vector, unique_ptr members | **Zero** | Declare nothing |
| A raw pointer you `new`/`delete` | **Five** | Or wrap in unique_ptr -> Rule of Zero |
| A file descriptor / OS handle | **Five** (delete copy) | Move-only entity |
| Virtual destructor for base class | Declare `virtual ~Base() = default;` | Still Rule of Zero if no resources |

---

## Notes

- Rule of Zero is the goal. If you find yourself writing Rule of Five, ask: "Can I wrap this in a smart pointer instead?"
- Always mark move operations `noexcept` - containers like `std::vector` won't use your move constructor without it, falling back to copying instead.
- The copy-and-swap idiom provides strong exception safety at the cost of one extra allocation.
- If you declare ANY of the five, declare ALL five (even if `= default` or `= delete`) - this documents intent and avoids surprising compiler-generated behavior.
- Modern C++ guideline: wrap every resource in an RAII class, then compose those RAII classes with Rule of Zero.
