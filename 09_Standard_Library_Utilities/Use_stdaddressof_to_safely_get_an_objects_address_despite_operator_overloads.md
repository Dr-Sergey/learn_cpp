# Use std::addressof to safely get an object's address despite operator& overloads

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/addressof>  

---

## Topic Overview

`std::addressof(obj)` obtains the actual memory address of `obj`, even if its class overloads `operator&`. This is critical in generic library code where you cannot assume `&obj` returns the true address.

### The Problem

C++ allows a class to overload unary `operator&`. When a type does this, writing `&obj` no longer gives you the object's address - it calls the overload instead and returns whatever that function returns. Here is a minimal example:

```cpp
struct Evil {
    int* operator&() { return nullptr; }  // overloaded operator&
};

Evil e;
// &e returns nullptr - NOT the real address!
// std::addressof(e) returns the actual address of e
```

### Why It Exists

- The C++ standard allows overloading unary `operator&`.
- Library implementations (allocators, containers, smart pointers) need the **real** address.
- Before C++11, implementers used `reinterpret_cast` hacks. `std::addressof` standardizes the safe approach.

### How It Works (Conceptually)

The implementation casts to `char&` first because `char` is one of the few types that cannot have an overloaded `operator&`. From there it takes the address and casts back.

```cpp
// Simplified implementation:
template<class T>
T* addressof(T& arg) noexcept {
    return reinterpret_cast<T*>(
        &const_cast<char&>(
            reinterpret_cast<const volatile char&>(arg)));
}
```

The trick: cast to `char&` (which cannot have an overloaded `operator&`), take its address, then cast back.

### Where the Standard Library Uses It

| Component | Uses `std::addressof` for |
| --- | --- |
| `std::allocator_traits::construct` | Getting the address to construct at |
| `std::vector`, `std::list`, etc. | Internal node/element pointers |
| `std::optional` | Constructing the contained value |
| `std::unique_ptr`, `std::shared_ptr` | Deleter invocations |

### Core Syntax

The two printouts for `Tricky` can differ because the overloaded `operator&` returns the address of `t.value`, not of `t` itself. `std::addressof` bypasses that entirely.

```cpp
#include <memory>
#include <iostream>

struct Normal {
    int value = 42;
};

struct Tricky {
    int value = 99;
    int* operator&() { return &value; }  // returns address of member, not object!
};

int main() {
    Normal n;
    std::cout << "&n              = " << &n << "\n";
    std::cout << "addressof(n)    = " << std::addressof(n) << "\n";
    // Both are the same - Normal doesn't overload operator&

    Tricky t;
    std::cout << "&t              = " << &t << "\n";              // address of t.value!
    std::cout << "addressof(t)    = " << std::addressof(t) << "\n"; // real address of t
    // &t and std::addressof(t) may differ!
}
```

---

## Self-Assessment

### Q1: Show that operator& can be overloaded and std::addressof bypasses it

**Answer:**

After the overloaded `operator&` fires and returns `nullptr`, notice that `std::addressof` gives back a usable pointer you can actually write through.

```cpp
#include <memory>
#include <iostream>
#include <cassert>

struct Proxy {
    int data = 10;

    // Overloaded operator& - returns something unexpected
    void* operator&() {
        std::cout << "  custom operator& called\n";
        return nullptr;  // deliberately wrong
    }
};

int main() {
    Proxy p;

    // Using &p calls the overloaded operator&
    void* via_ampersand = &p;
    std::cout << "&p              = " << via_ampersand << "\n";
    // Output:
    //   custom operator& called
    //   &p              = 0  (nullptr)

    // Using std::addressof bypasses the overload
    Proxy* real_addr = std::addressof(p);
    std::cout << "addressof(p)    = " << real_addr << "\n";
    // Output: addressof(p)    = 0x... (actual stack address)

    // Prove it's the real address
    real_addr->data = 20;
    std::cout << "p.data          = " << p.data << "\n";
    // Output: p.data          = 20

    assert(real_addr != nullptr);
    std::cout << "addressof works correctly!\n";
}
```

---

### Q2: Explain why standard library code uses std::addressof instead of &

**Answer:**

The standard library is **generic** - it operates on user-supplied types where `operator&` may be overloaded. Using the plain `&` operator would silently return the wrong address, leading to undefined behavior (constructing objects at wrong locations, corrupting memory, etc.).

**Example: a custom allocator that must use addressof:**

```cpp
#include <memory>
#include <iostream>

struct Widget {
    int val;
    // Imagine a legacy codebase overloaded this for COM interop
    int* operator&() { return &val; }
};

template<typename T>
class SafeAllocator {
public:
    using value_type = T;
    T* allocate(std::size_t n) {
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }
    void deallocate(T* p, std::size_t) { ::operator delete(p); }

    // WRONG: using & on obj
    // T* bad_address(T& obj) { return &obj; }  // calls overloaded operator&!

    // CORRECT: using std::addressof
    T* safe_address(T& obj) { return std::addressof(obj); }
};

int main() {
    SafeAllocator<Widget> alloc;
    Widget* p = alloc.allocate(1);
    new (p) Widget{42};

    Widget* addr = alloc.safe_address(*p);
    std::cout << "real address: " << addr << "\n";
    std::cout << "value:        " << addr->val << "\n";
    // Output: real address: 0x... , value: 42

    p->~Widget();
    alloc.deallocate(p, 1);
}
```

**Key rule:** In any template code that takes an arbitrary type `T`, always use `std::addressof(obj)` instead of `&obj` to get the object's address.

---

### Q3: Demonstrate a real use case: a proxy type that overloads operator& for syntax purposes

**Answer:**

A classic real-world case is COM programming on Windows, where `operator&` is overloaded on smart pointer wrappers to support the `QueryInterface` pattern:

```cpp
#include <memory>
#include <iostream>
#include <cassert>

// Simplified COM-style smart pointer
template<typename T>
class ComPtr {
    T* ptr_ = nullptr;
public:
    ComPtr() = default;
    explicit ComPtr(T* p) : ptr_(p) {}
    ~ComPtr() { delete ptr_; }

    T* operator->() const { return ptr_; }
    T& operator*() const { return *ptr_; }

    // Overload operator& to return pointer-to-pointer
    // This allows: SomeFunction(&comPtr) to fill in the internal pointer
    T** operator&() {
        // Release current and return address for reassignment
        delete ptr_;
        ptr_ = nullptr;
        return &ptr_;
    }

    T* get() const { return ptr_; }
};

struct Resource {
    int id;
};

// Simulates a factory function that fills a pointer-to-pointer
void CreateResource(Resource** out) {
    *out = new Resource{99};
}

int main() {
    ComPtr<Resource> r;

    // operator& lets us pass &r to C-style APIs naturally
    CreateResource(&r);  // calls overloaded operator&
    std::cout << "resource id = " << r->id << "\n";
    // Output: resource id = 99

    // But we need the real address of the ComPtr object itself:
    ComPtr<Resource>* real = std::addressof(r);
    std::cout << "ComPtr address = " << real << "\n";
    // Output: ComPtr address = 0x... (stack address of r)

    assert(real->get()->id == 99);
    std::cout << "addressof works with COM-style wrappers\n";
}
```

**Explanation:**

- `ComPtr::operator&()` returns `T**` so that C-style factory functions can fill in the smart pointer's internal raw pointer.
- If library code needs the address of the `ComPtr` object itself (e.g., for placement new, container storage), it must use `std::addressof`.

---

## Notes

- `std::addressof` is `constexpr` since C++17 - it can be used in constant expressions.
- It works with `const` and `volatile` qualified objects.
- Never overload `operator&` in your own types unless you have a very compelling reason (COM interop is the classic case).
- In C++20 ranges and concepts, `std::addressof` is used internally to ensure correctness with arbitrary iterator/sentinel types.
- `std::addressof` cannot be used with rvalues - it requires an lvalue reference.
