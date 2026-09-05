# Know the Rules of the Rule of Zero, Rule of Three, and Rule of Five

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/rule_of_three>  

---

## Topic Overview

### The Three Rules

These rules tell you when you need to define special member functions and how many. If you remember only one thing from this topic, make it this: whenever you define one of the "big five," you almost certainly need to think carefully about all five.

| Rule | When | You Must Define |
| --- | --- | --- |
| **Rule of Zero** | Class owns no raw resources | **None** - rely on compiler-generated defaults |
| **Rule of Three** | Class manages a resource (pre-C++11) | Destructor, Copy Constructor, Copy Assignment |
| **Rule of Five** | Class manages a resource (C++11+) | Destructor, Copy Ctor, Copy Assignment, Move Ctor, Move Assignment |

### Rule of Zero (Preferred)

If your class uses only RAII types (`std::string`, `std::vector`, `std::unique_ptr`, etc.), the compiler-generated special members do the right thing. You do not have to write anything, and the result is correct and efficient.

```cpp
struct Person {
    std::string name;
    std::vector<int> scores;
    // No destructor, no copy/move - compiler generates correct ones
};
```

### Rule of Three (C++03)

If you manually manage a resource (raw `new`, file handle, etc.), the compiler-generated copy constructor and copy assignment perform **shallow copies**, causing double-free. You need to define all three to get deep-copy semantics:

```cpp
Must define: ~T(), T(const T&), T& operator=(const T&)
```

### Rule of Five (C++11+)

C++11 added move semantics. Here is the tricky part: if you define any of the Big Three, the compiler **won't generate** move operations for you. Without move operations, your type cannot be moved efficiently - it will fall back to copying. So in C++11 and later, if you're managing a resource manually, you should define all five:

```cpp
Must define: ~T(), T(const T&), T& operator=(const T&),
             T(T&&) noexcept, T& operator=(T&&) noexcept
```

### Decision Flowchart

Use this as your starting point whenever you are designing a class:

```cpp
Does your class manage a raw resource?
├── NO  -> Rule of Zero (define nothing)
└── YES -> Can you wrap it in a RAII type (unique_ptr, etc.)?
    ├── YES -> Rule of Zero (wrap the resource)
    └── NO  -> Rule of Five (define all five)
```

---

## Self-Assessment

### Q1: Write a class that correctly follows the Rule of Five with all five special members defined

`DynamicBuffer` owns a raw `char*`, so it must define all five. Notice that the copy assignment uses the copy-and-swap idiom - this gives strong exception safety automatically.

```cpp
#include <iostream>
#include <cstring>
#include <algorithm>
#include <utility>

class DynamicBuffer {
    char* data_;
    size_t size_;

public:
    // Constructor
    explicit DynamicBuffer(const char* str = "")
        : size_(std::strlen(str))
        , data_(new char[std::strlen(str) + 1])
    {
        std::strcpy(data_, str);
        std::cout << "  [ctor] \"" << data_ << "\" (" << size_ << " bytes)\n";
    }

    // 1. Destructor
    ~DynamicBuffer() {
        std::cout << "  [dtor] \"" << (data_ ? data_ : "moved") << "\"\n";
        delete[] data_;
    }

    // 2. Copy Constructor
    DynamicBuffer(const DynamicBuffer& other)
        : size_(other.size_)
        , data_(new char[other.size_ + 1])
    {
        std::strcpy(data_, other.data_);
        std::cout << "  [copy ctor] \"" << data_ << "\"\n";
    }

    // 3. Copy Assignment (copy-and-swap idiom)
    DynamicBuffer& operator=(const DynamicBuffer& other) {
        std::cout << "  [copy assign] \"" << other.data_ << "\"\n";
        if (this != &other) {
            DynamicBuffer tmp(other);      // copy
            swap(*this, tmp);              // swap
        }                                  // tmp destroys old data
        return *this;
    }

    // 4. Move Constructor
    DynamicBuffer(DynamicBuffer&& other) noexcept
        : data_(other.data_)
        , size_(other.size_)
    {
        other.data_ = nullptr;
        other.size_ = 0;
        std::cout << "  [move ctor] \"" << data_ << "\"\n";
    }

    // 5. Move Assignment
    DynamicBuffer& operator=(DynamicBuffer&& other) noexcept {
        std::cout << "  [move assign]\n";
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }

    // Swap helper
    friend void swap(DynamicBuffer& a, DynamicBuffer& b) noexcept {
        std::swap(a.data_, b.data_);
        std::swap(a.size_, b.size_);
    }

    const char* c_str() const { return data_ ? data_ : "(null)"; }
    size_t size() const { return size_; }
};

int main() {
    std::cout << "--- Construction ---\n";
    DynamicBuffer a("Hello");

    std::cout << "\n--- Copy construction ---\n";
    DynamicBuffer b(a);

    std::cout << "\n--- Copy assignment ---\n";
    DynamicBuffer c("World");
    c = a;

    std::cout << "\n--- Move construction ---\n";
    DynamicBuffer d(std::move(a));
    std::cout << "a after move: \"" << a.c_str() << "\"\n";

    std::cout << "\n--- Move assignment ---\n";
    DynamicBuffer e("Temp");
    e = std::move(b);

    std::cout << "\n--- Destruction ---\n";
    return 0;
}
```

**Output:**

```text
--- Construction ---
  [ctor] "Hello" (5 bytes)

--- Copy construction ---
  [copy ctor] "Hello"

--- Copy assignment ---
  [ctor] "World" (5 bytes)
  [copy assign] "Hello"
  [copy ctor] "Hello"
  [dtor] "World"

--- Move construction ---
  [move ctor] "Hello"
a after move: "(null)"

--- Move assignment ---
  [ctor] "Temp" (4 bytes)
  [move assign]

--- Destruction ---
  [dtor] "Hello"
  [dtor] "moved"
  [dtor] "Hello"
  [dtor] "moved"
  [dtor] "Hello"
```

### Q2: Explain when you can rely on the Rule of Zero (compiler-generated members are sufficient)

The Rule of Zero applies when **all data members are RAII types** that manage their own resources. The `Employee` class below looks like it might need special handling because of `unique_ptr`, but actually the compiler gets it right automatically - `unique_ptr` makes the class move-only, which is usually what you want.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>

// Rule of Zero - no special members needed!
class Employee {
    std::string name_;
    int age_;
    std::vector<std::string> skills_;
    std::unique_ptr<int> badge_id_;  // unique_ptr handles ownership

public:
    Employee(std::string name, int age, int id)
        : name_(std::move(name))
        , age_(age)
        , badge_id_(std::make_unique<int>(id))
    {}

    // No destructor - string, vector, unique_ptr clean up
    // No copy constructor - unique_ptr makes class move-only (correct!)
    // No move constructor - compiler generates one
    // No copy/move assignment - compiler handles it

    void add_skill(std::string s) { skills_.push_back(std::move(s)); }

    void print() const {
        std::cout << name_ << " (age " << age_ << ", badge #"
                  << *badge_id_ << "), skills: ";
        for (const auto& s : skills_) std::cout << s << " ";
        std::cout << "\n";
    }
};

int main() {
    Employee e("Alice", 30, 42);
    e.add_skill("C++");
    e.add_skill("Python");
    e.print();

    // Move works automatically
    Employee e2 = std::move(e);
    e2.print();

    // Copy is deleted (unique_ptr makes it move-only)
    // Employee e3 = e2;  // ERROR: deleted copy constructor

    return 0;
}
```

**When Rule of Zero applies:**

| Data Members | Rule |
| --- | --- |
| `int`, `double`, POD types | Zero (trivially copyable) |
| `std::string`, `std::vector` | Zero (value semantics) |
| `std::unique_ptr` | Zero (move-only, auto cleanup) |
| `std::shared_ptr` | Zero (shared ownership) |
| Raw `T*` with ownership | **Five** - must manage manually |
| `FILE*`, `int fd` | **Five** - must manage manually |

### Q3: Show how violating the Rule of Three leads to a double-free bug

This is the classic example of why you cannot define only a destructor and ignore copy. The compiler-generated copy gives you a shallow copy - two objects pointing to the same memory - and when both go out of scope, the memory is freed twice.

```cpp
#include <iostream>
#include <cstring>

// BAD: Manages a resource but only defines destructor
class BrokenString {
    char* data_;
public:
    BrokenString(const char* s) : data_(new char[std::strlen(s) + 1]) {
        std::strcpy(data_, s);
        std::cout << "  [ctor] data_=" << (void*)data_ << " \"" << data_ << "\"\n";
    }

    ~BrokenString() {
        std::cout << "  [dtor] deleting data_=" << (void*)data_ << "\n";
        delete[] data_;  // DOUBLE FREE when copied!
    }

    // NO copy constructor defined - compiler generates shallow copy
    // NO copy assignment defined - compiler generates shallow copy

    const char* c_str() const { return data_; }
};

int main() {
    std::cout << "=== The double-free bug ===\n\n";

    std::cout << "1. Create original:\n";
    BrokenString a("Hello");

    std::cout << "\n2. Copy (shallow - both point to same memory!):\n";
    BrokenString b = a;  // Compiler-generated copy: b.data_ = a.data_
    std::cout << "  a.data_ address: " << (void*)a.c_str() << "\n";
    std::cout << "  b.data_ address: " << (void*)b.c_str() << "\n";
    std::cout << "  Same pointer? " << (a.c_str() == b.c_str() ? "YES - BUG!" : "no") << "\n";

    std::cout << "\n3. Destruction (reverse order):\n";
    // b destructor runs first: delete[] data_ (OK)
    // a destructor runs second: delete[] data_ (DOUBLE FREE - UB!)

    // NOTE: This program has undefined behavior.
    // Run with -fsanitize=address to see the error clearly.

    return 0;
}
```

**Output (before crash/UB):**

```text
=== The double-free bug ===

1. Create original:
  [ctor] data_=0x... "Hello"

2. Copy (shallow - both point to same memory!):
  a.data_ address: 0x12345
  b.data_ address: 0x12345
  Same pointer? YES - BUG!

3. Destruction (reverse order):
  [dtor] deleting data_=0x12345
  [dtor] deleting data_=0x12345    <- DOUBLE FREE!
```

The fix is to implement all three (Rule of Three) and give each object its own copy of the data:

```cpp
class FixedString {
    char* data_;
public:
    FixedString(const char* s) : data_(new char[std::strlen(s) + 1]) {
        std::strcpy(data_, s);
    }

    // 1. Destructor
    ~FixedString() { delete[] data_; }

    // 2. Copy Constructor - deep copy
    FixedString(const FixedString& other)
        : data_(new char[std::strlen(other.data_) + 1])
    {
        std::strcpy(data_, other.data_);
    }

    // 3. Copy Assignment - deep copy with self-assignment check
    FixedString& operator=(const FixedString& other) {
        if (this != &other) {
            delete[] data_;
            data_ = new char[std::strlen(other.data_) + 1];
            std::strcpy(data_, other.data_);
        }
        return *this;
    }
};
```

---

## Notes

- **Always prefer Rule of Zero** - wrap raw resources in RAII wrappers (`unique_ptr` with custom deleter, etc.).
- If you must write a destructor, you almost certainly need copy/move operations too.
- Mark move operations `noexcept` - containers like `std::vector` only use move if it's `noexcept`.
- The copy-and-swap idiom provides strong exception safety for copy assignment.
- `= default` and `= delete` let you be explicit about which operations are available.
