# Design policy-based classes using template parameters

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

**Policy-based design**, popularized by Andrei Alexandrescu in "Modern C++ Design", is the idea of making a class's behavior configurable through template parameters instead of runtime polymorphism. Instead of subclassing to get different behaviors, you compose behaviors at compile time by passing different policy types. The result is zero-overhead customization - the compiler can see exactly which policy you chose and inline everything.

Here's the contrast with the traditional approach:

```cpp
Traditional:  class Logger : public FileOutput, public JsonFormat { ... }
Policy-based: template<typename OutputPolicy, typename FormatPolicy>
              class Logger : private OutputPolicy, private FormatPolicy { ... }
```

| Aspect | Virtual Inheritance | Policy-based |
| --- | :---: | :---: |
| Dispatch | Runtime (vtable) | Compile-time |
| Adding behaviors | Modify hierarchy | Mix-and-match policies |
| Combinations | Class explosion | Composable |
| Overhead | vtable + indirection | Zero |

The "class explosion" problem is real: if you have 3 output policies and 3 format policies, virtual inheritance tempts you to write 9 concrete subclasses. Policy-based design gives you all 9 combinations with 6 policy structs and one template class.

---

## Self-Assessment

### Q1: Implement a policy-based smart pointer

This example shows three independent policy axes - ownership, null checking, and thread safety - each as a separate template parameter. The smart pointer inherits privately from all three policies (the "implemented-in-terms-of" idiom), which gives it access to their methods while enabling the empty base optimization when a policy has no data. Private inheritance is used here rather than composition specifically to allow that size optimization - a policy struct with no members costs zero bytes when inherited privately.

```cpp
#include <iostream>
#include <mutex>

// Ownership policies
template<typename T>
struct UniqueOwnership {
    void acquire(T*) { /* no-op, single owner */ }
    bool release(T* p) {
        delete p;
        return true;  // Always delete
    }
};

template<typename T>
struct RefCounted {
    int* count_ = nullptr;
    void acquire(T*) {
        if (!count_) count_ = new int(1);
        else ++(*count_);
    }
    bool release(T*) {
        if (count_ && --(*count_) == 0) {
            delete count_;
            count_ = nullptr;
            return true;  // Last reference, delete
        }
        return false;
    }
};

// Checking policies
template<typename T>
struct NoCheck {
    void check(T*) const {}  // No overhead
};

template<typename T>
struct AssertCheck {
    void check(T* p) const {
        if (!p) {
            std::cerr << "Null pointer dereference!\n";
            std::abort();
        }
    }
};

// Threading policies
struct SingleThreaded {
    void lock() {}
    void unlock() {}
};

struct MultiThreaded {
    std::mutex mtx_;
    void lock() { mtx_.lock(); }
    void unlock() { mtx_.unlock(); }
};

// Policy-based smart pointer
template<typename T,
         template<typename> class OwnershipPolicy = UniqueOwnership,
         template<typename> class CheckPolicy = NoCheck,
         typename ThreadPolicy = SingleThreaded>
class SmartPtr : private OwnershipPolicy<T>,
                 private CheckPolicy<T>,
                 private ThreadPolicy {
    T* ptr_ = nullptr;
public:
    explicit SmartPtr(T* p = nullptr) : ptr_(p) {
        if (ptr_) OwnershipPolicy<T>::acquire(ptr_);
    }
    ~SmartPtr() {
        if (ptr_) OwnershipPolicy<T>::release(ptr_);
    }

    T& operator*() const {
        CheckPolicy<T>::check(ptr_);
        return *ptr_;
    }
    T* operator->() const {
        CheckPolicy<T>::check(ptr_);
        return ptr_;
    }
};

// Mix-and-match:
struct Widget { int x = 42; };

using FastPtr = SmartPtr<Widget, UniqueOwnership, NoCheck, SingleThreaded>;
using SafePtr = SmartPtr<Widget, UniqueOwnership, AssertCheck, SingleThreaded>;
using SharedSafePtr = SmartPtr<Widget, RefCounted, AssertCheck, MultiThreaded>;

int main() {
    FastPtr p1(new Widget());     // Zero overhead
    SafePtr p2(new Widget());     // Null-checked
    std::cout << p1->x << "\n";
    std::cout << (*p2).x << "\n";
    return 0;
}
```

The `using` aliases at the bottom are the payoff: you can define named presets for common combinations and use them throughout your codebase by name. In debug builds, `SafePtr` everywhere; in release, swap to `FastPtr` without touching any usage code.

### Q2: Build a configurable allocator using policies

This example separates allocation strategy from logging behavior, two concerns that are completely orthogonal. A pool allocator is faster for many small allocations but can't free individual objects; malloc is general purpose. Verbose logging is useful for tracking memory issues during development; you want it gone in production. Policy-based design lets you combine these independently without writing four separate container classes.

```cpp
#include <cstdlib>
#include <iostream>
#include <new>

// Allocation strategies
struct MallocAllocator {
    static void* allocate(size_t n) { return std::malloc(n); }
    static void deallocate(void* p) { std::free(p); }
};

struct PoolAllocator {
    static inline char pool[4096];
    static inline size_t offset = 0;
    static void* allocate(size_t n) {
        if (offset + n > sizeof(pool)) throw std::bad_alloc();
        void* p = pool + offset;
        offset += n;
        return p;
    }
    static void deallocate(void*) { /* Pool freed all at once */ }
};

// Logging policies
struct NoLogging {
    static void on_alloc(size_t) {}
    static void on_dealloc() {}
};

struct VerboseLogging {
    static void on_alloc(size_t n) {
        std::cout << "  [alloc " << n << " bytes]\n";
    }
    static void on_dealloc() {
        std::cout << "  [dealloc]\n";
    }
};

// Policy-based container
template<typename T,
         typename AllocPolicy = MallocAllocator,
         typename LogPolicy = NoLogging>
class SimpleVector {
    T* data_ = nullptr;
    size_t size_ = 0, cap_ = 0;
public:
    ~SimpleVector() {
        for (size_t i = 0; i < size_; ++i) data_[i].~T();
        if (data_) {
            LogPolicy::on_dealloc();
            AllocPolicy::deallocate(data_);
        }
    }
    void push_back(const T& val) {
        if (size_ == cap_) grow();
        new (&data_[size_++]) T(val);
    }
    size_t size() const { return size_; }
private:
    void grow() {
        size_t new_cap = cap_ ? cap_ * 2 : 4;
        LogPolicy::on_alloc(new_cap * sizeof(T));
        T* new_data = static_cast<T*>(AllocPolicy::allocate(new_cap * sizeof(T)));
        for (size_t i = 0; i < size_; ++i) {
            new (&new_data[i]) T(std::move(data_[i]));
            data_[i].~T();
        }
        if (data_) AllocPolicy::deallocate(data_);
        data_ = new_data;
        cap_ = new_cap;
    }
};

int main() {
    // Fast: pool allocator, no logging
    SimpleVector<int, PoolAllocator, NoLogging> fast;
    fast.push_back(1);

    // Debug: malloc with verbose logging
    SimpleVector<int, MallocAllocator, VerboseLogging> debug;
    debug.push_back(42);
    debug.push_back(99);
    return 0;
}
```

Because both policies use static methods, the compiler inlines them completely and the `NoLogging` calls disappear entirely from the generated code. There is literally zero cost to the abstraction in the fast path.

### Q3: Show policy-based design with C++20 concepts for constraints

Before concepts, the error message when you passed a wrong type as a policy was a wall of template instantiation failures. Concepts let you describe precisely what a policy must provide, and the compiler reports the mismatch at the point of use rather than deep in the instantiation chain. This is one of the biggest quality-of-life improvements C++20 brings to template-heavy code.

```cpp
#include <concepts>
#include <iostream>
#include <string>

// Concepts constrain policies
template<typename P>
concept SerializationPolicy = requires(P p, const std::string& data) {
    { p.serialize(data) } -> std::convertible_to<std::string>;
    { p.deserialize(std::string{}) } -> std::convertible_to<std::string>;
};

template<typename P>
concept CompressionPolicy = requires(P p, const std::string& data) {
    { p.compress(data) } -> std::convertible_to<std::string>;
    { p.decompress(std::string{}) } -> std::convertible_to<std::string>;
};

// Concrete policies
struct JsonFormat {
    std::string serialize(const std::string& d) const { return "{\"data\":\"" + d + "\"}"; }
    std::string deserialize(const std::string& s) const { return s.substr(9, s.size()-11); }
};

struct PlainFormat {
    std::string serialize(const std::string& d) const { return d; }
    std::string deserialize(const std::string& s) const { return s; }
};

struct NoCompression {
    std::string compress(const std::string& d) const { return d; }
    std::string decompress(const std::string& d) const { return d; }
};

// Constrained policy-based class
template<SerializationPolicy Serializer, CompressionPolicy Compressor = NoCompression>
class DataStore : private Serializer, private Compressor {
public:
    std::string save(const std::string& data) {
        auto serialized = Serializer::serialize(data);
        return Compressor::compress(serialized);
    }
    std::string load(const std::string& stored) {
        auto decompressed = Compressor::decompress(stored);
        return Serializer::deserialize(decompressed);
    }
};

int main() {
    DataStore<JsonFormat> json_store;
    auto stored = json_store.save("hello");
    std::cout << stored << "\n";  // {"data":"hello"}
    std::cout << json_store.load(stored) << "\n";  // hello

    DataStore<PlainFormat> plain_store;
    std::cout << plain_store.save("world") << "\n";  // world
    return 0;
}
```

If you try `DataStore<int>`, the concept check fires immediately with a message like "int does not satisfy SerializationPolicy" and shows exactly which `serialize`/`deserialize` requirement wasn't met. That's a fundamentally better developer experience than pre-C++20 template errors.

---

## Notes

- Policy-based design gives **compile-time** composition with **zero overhead**.
- Use `template template` parameters when policies themselves are templates.
- Private inheritance from policies = "implemented-in-terms-of" + empty base optimization.
- C++20 concepts replace SFINAE for constraining policies - much better error messages.
- Key difference from strategy pattern: policies are compile-time, strategies are runtime.
- Classic examples: `std::allocator`, `std::char_traits`, smart pointer deleters.
