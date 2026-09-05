# Understand How Deleter Types Affect `unique_ptr` Size

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/unique_ptr>  

---

## Topic Overview

### `unique_ptr` Stores the Deleter

`std::unique_ptr<T, Deleter>` stores both a pointer to the managed object and the deleter. The **type** of deleter determines the size. This matters when you have arrays of `unique_ptr` or pack them into structs - a bloated deleter multiplies across every instance.

| Deleter Type | Size of `unique_ptr` | Why |
| --- | --- | --- |
| `std::default_delete<T>` (default) | `sizeof(T*)` = 8 bytes | **Empty Base Optimization (EBO)** - empty class occupies 0 bytes |
| Stateless functor / empty lambda | `sizeof(T*)` = 8 bytes | EBO applies |
| Stateful lambda (captures) | `sizeof(T*) + sizeof(captures)` | Must store captured state |
| `std::function<void(T*)>` | `sizeof(T*) + sizeof(std::function)` ~= 40+ bytes | Heavy type-erasure overhead |
| Function pointer `void(*)(T*)` | `sizeof(T*) + sizeof(void*)` = 16 bytes | Must store the pointer |

### Empty Base Optimization (EBO)

When the deleter is an **empty class** (no data members), the compiler can overlap it with the pointer using compressed pair / EBO, so it takes **zero extra space**. The layout comparison makes this intuitive:

```cpp
unique_ptr with default_delete:        unique_ptr with function pointer:
┌──────────┐                           ┌──────────┐
│ T* ptr   │  8 bytes                  │ T* ptr   │  8 bytes
└──────────┘                           │ void(*f) │  8 bytes
                                       └──────────┘
Total: 8 bytes                         Total: 16 bytes
```

A function pointer always costs an extra 8 bytes because it is not a type - it is a value that can differ between instances, so it must be stored. An empty class carries no per-instance state, so EBO compresses it to nothing.

---

## Self-Assessment

### Q1: Show that `unique_ptr<T, std::default_delete<T>>` has the same size as a raw pointer due to EBO

The `static_assert` at the bottom of this example is the machine-checkable proof that EBO is working. If the default deleter were not empty (or if EBO were not applied), the assertion would fire.

```cpp
#include <iostream>
#include <memory>
#include <functional>

struct Widget { int x; double y; };

// Empty deleters (stateless) - size 1 as standalone, but EBO makes them 0 inside unique_ptr
struct EmptyDeleter {
    void operator()(Widget* p) const { delete p; }
};

// Stateful deleter - must store state
struct CountingDeleter {
    int count = 0;
    void operator()(Widget* p) {
        ++count;
        std::cout << "  Deleted (count=" << count << ")\n";
        delete p;
    }
};

int main() {
    using default_up = std::unique_ptr<Widget>;
    using empty_up = std::unique_ptr<Widget, EmptyDeleter>;
    using counting_up = std::unique_ptr<Widget, CountingDeleter>;
    using funcptr_up = std::unique_ptr<Widget, void(*)(Widget*)>;
    using stdfunc_up = std::unique_ptr<Widget, std::function<void(Widget*)>>;

    std::cout << "=== Size comparison ===\n";
    std::cout << "sizeof(Widget*):                              " << sizeof(Widget*) << "\n";
    std::cout << "sizeof(unique_ptr<Widget>):                   " << sizeof(default_up) << "\n";
    std::cout << "sizeof(unique_ptr<Widget, EmptyDeleter>):     " << sizeof(empty_up) << "\n";
    std::cout << "sizeof(unique_ptr<Widget, CountingDeleter>):  " << sizeof(counting_up) << "\n";
    std::cout << "sizeof(unique_ptr<Widget, void(*)(Widget*)>): " << sizeof(funcptr_up) << "\n";
    std::cout << "sizeof(unique_ptr<Widget, function<...>>):    " << sizeof(stdfunc_up) << "\n";

    std::cout << "\n=== Deleter struct sizes ===\n";
    std::cout << "sizeof(std::default_delete<Widget>): " << sizeof(std::default_delete<Widget>) << "\n";
    std::cout << "sizeof(EmptyDeleter):                " << sizeof(EmptyDeleter) << "\n";
    std::cout << "sizeof(CountingDeleter):             " << sizeof(CountingDeleter) << "\n";

    // EBO works: default_delete is empty, so unique_ptr is same size as raw pointer
    static_assert(sizeof(default_up) == sizeof(Widget*),
                  "unique_ptr with default_delete should be same size as raw pointer");

    return 0;
}
```

**Output (64-bit system):**

```text
=== Size comparison ===
sizeof(Widget*):                              8
sizeof(unique_ptr<Widget>):                   8
sizeof(unique_ptr<Widget, EmptyDeleter>):     8
sizeof(unique_ptr<Widget, CountingDeleter>):  16
sizeof(unique_ptr<Widget, void(*)(Widget*)>): 16
sizeof(unique_ptr<Widget, function<...>>):    40

=== Deleter struct sizes ===
sizeof(std::default_delete<Widget>): 1
sizeof(EmptyDeleter):                1
sizeof(CountingDeleter):             4
```

Notice that `EmptyDeleter` is 1 byte as a standalone object (the minimum required by the standard) but contributes 0 bytes inside `unique_ptr` thanks to EBO.

### Q2: Show that a stateful lambda deleter causes `unique_ptr` to store both pointer and deleter

Captures are the key distinction here. A captureless lambda is an empty type and gets the EBO treatment. As soon as you capture anything - even a single `int` - the lambda has state and must be stored alongside the pointer.

```cpp
#include <iostream>
#include <memory>

int main() {
    // Captureless lambda - empty, EBO applies
    auto stateless_del = [](int* p) {
        std::cout << "  Stateless delete\n";
        delete p;
    };
    using stateless_up = std::unique_ptr<int, decltype(stateless_del)>;
    std::cout << "sizeof(unique_ptr with stateless lambda): " << sizeof(stateless_up) << "\n";

    // Lambda capturing one int by value - stores 4 bytes of state
    int log_id = 42;
    auto stateful_del = [log_id](int* p) {
        std::cout << "  Stateful delete (log_id=" << log_id << ")\n";
        delete p;
    };
    using stateful_up = std::unique_ptr<int, decltype(stateful_del)>;
    std::cout << "sizeof(unique_ptr with stateful lambda):  " << sizeof(stateful_up) << "\n";

    // Lambda capturing a string by value - stores sizeof(string) bytes
    std::string label = "debug";
    auto heavy_del = [label](int* p) {
        std::cout << "  Heavy delete (label=" << label << ")\n";
        delete p;
    };
    using heavy_up = std::unique_ptr<int, decltype(heavy_del)>;
    std::cout << "sizeof(unique_ptr with string-capture):   " << sizeof(heavy_up) << "\n";

    std::cout << "\n=== In action ===\n";

    // Stateless - same size as raw pointer
    stateless_up p1(new int(1), stateless_del);
    std::cout << "p1 value: " << *p1 << "\n";

    // Stateful - pointer + captured int
    stateful_up p2(new int(2), stateful_del);
    std::cout << "p2 value: " << *p2 << "\n";

    std::cout << "\n--- Destruction ---\n";
    // Deleters called on scope exit
    return 0;
}
```

**Output (64-bit, typical):**

```text
sizeof(unique_ptr with stateless lambda): 8
sizeof(unique_ptr with stateful lambda):  16
sizeof(unique_ptr with string-capture):   40

=== In action ===
p1 value: 1
p2 value: 2

--- Destruction ---
  Stateful delete (log_id=42)
  Stateless delete
```

The string-capture case blows up to 40 bytes - `sizeof(string)` plus the pointer. If you are thinking "I need a log label in my deleter," consider logging from the destructor of the managed object instead.

### Q3: Write a zero-size custom deleter for a C `FILE*` and verify `sizeof` equals `sizeof(void*)`

This is the idiomatic way to wrap any C cleanup function - `fclose`, `SDL_DestroyWindow`, `curl_easy_cleanup`, and so on. An empty struct deleter gives you a self-cleaning RAII handle with zero size overhead compared to a raw pointer.

```cpp
#include <iostream>
#include <memory>
#include <cstdio>

// Zero-size deleter for FILE* - uses EBO in unique_ptr
struct FileCloser {
    void operator()(FILE* f) const {
        if (f) {
            std::cout << "  Closing file\n";
            std::fclose(f);
        }
    }
};

// Verify it's empty
static_assert(sizeof(FileCloser) == 1, "Empty class is 1 byte standalone");

// Type alias for convenience
using UniqueFile = std::unique_ptr<FILE, FileCloser>;

// Same size as raw pointer? YES - EBO kicks in
static_assert(sizeof(UniqueFile) == sizeof(FILE*),
              "UniqueFile should be same size as raw FILE*");

// For comparison: using function pointer would be 16 bytes
using FuncPtrFile = std::unique_ptr<FILE, int(*)(FILE*)>;

UniqueFile open_file(const char* path, const char* mode) {
    return UniqueFile(std::fopen(path, mode));
}

int main() {
    std::cout << "=== FILE* with zero-size deleter ===\n\n";

    std::cout << "sizeof(FILE*):          " << sizeof(FILE*) << "\n";
    std::cout << "sizeof(UniqueFile):     " << sizeof(UniqueFile) << "\n";
    std::cout << "sizeof(FuncPtrFile):    " << sizeof(FuncPtrFile) << "\n";
    std::cout << "sizeof(FileCloser):     " << sizeof(FileCloser) << " (standalone)\n";
    std::cout << "Zero overhead? " << (sizeof(UniqueFile) == sizeof(FILE*) ? "YES" : "NO") << "\n\n";

    // Use it
    {
        UniqueFile f = open_file("test_delete_size.txt", "w");
        if (f) {
            std::fputs("Hello from unique_ptr<FILE, FileCloser>!\n", f.get());
            std::cout << "Wrote to file\n";
        }
    }   // FileCloser::operator()(FILE*) called automatically

    // Read back
    {
        UniqueFile f = open_file("test_delete_size.txt", "r");
        if (f) {
            char buf[256];
            if (std::fgets(buf, sizeof(buf), f.get())) {
                std::cout << "Read: " << buf;
            }
        }
    }

    // Cleanup test file
    std::remove("test_delete_size.txt");

    return 0;
}
```

**Output:**

```text
=== FILE* with zero-size deleter ===

sizeof(FILE*):          8
sizeof(UniqueFile):     8
sizeof(FuncPtrFile):    16
sizeof(FileCloser):     1 (standalone)
Zero overhead? YES

Wrote to file
  Closing file
Read: Hello from unique_ptr<FILE, FileCloser>!
  Closing file
```

`FuncPtrFile` at 16 bytes versus `UniqueFile` at 8 bytes - that difference doubles the size of any array or struct holding these handles.

---

## Notes

- **Always prefer empty/stateless deleters** for zero-overhead `unique_ptr`. Use a functor class or captureless lambda.
- Function pointers always add `sizeof(void*)` - they can't EBO because they store different values at runtime.
- `std::function` adds even more overhead (~32 bytes) due to type erasure and heap allocation. Avoid it as a deleter.
- For C API cleanup (fclose, free, SDL_DestroyWindow, etc.), an empty struct deleter is the idiomatic pattern.
- In C++20, stateless lambdas are default-constructible, so `unique_ptr<T, decltype(lambda)>` works without passing the lambda instance.
