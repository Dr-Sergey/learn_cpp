# Use std::unique_ptr with Custom Deleters for Non-Heap Resources

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/unique_ptr>  

---

## Topic Overview

### Why Custom Deleters

`std::unique_ptr` defaults to calling `delete`, but many resources are **not** heap-allocated objects. The moment you start wrapping C handles, file descriptors, or OS resources you need to tell `unique_ptr` what the release operation actually is:

| Resource | Acquire | Release | Custom deleter needed |
| --- | --- | --- | --- |
| POSIX file descriptor | `open()` | `close()` | Yes |
| C `FILE*` | `fopen()` | `fclose()` | Yes |
| Win32 `HANDLE` | `CreateFile()` | `CloseHandle()` | Yes |
| COM interface | `CoCreate...` | `p->Release()` | Yes |
| `mmap` region | `mmap()` | `munmap()` | Yes |
| C library buffer | `malloc()` | `free()` | Yes |

### Deleter Types and Size Impact

The size of the `unique_ptr` object itself depends on whether the deleter carries state. This matters in tight data structures:

```cpp
unique_ptr<T>                    -> sizeof(T*)        (default delete)
unique_ptr<T, Stateless>         -> sizeof(T*)        (EBO applies)
unique_ptr<T, void(*)(T*)>       -> sizeof(T*) x 2    (stores fn pointer)
unique_ptr<T, StatefulFunctor>   -> sizeof(T*) + sizeof(state)
```

**Empty Base Optimization (EBO):** When the deleter is a stateless type (empty struct/class or captureless lambda type), `unique_ptr` stores it at zero cost via EBO.

### Syntax Patterns

There are three common ways to attach a deleter. A captureless lambda or an empty functor struct is usually the right choice because they add no size overhead:

```cpp
// Function pointer deleter (adds sizeof(void*) to unique_ptr)
std::unique_ptr<FILE, decltype(&fclose)> f(fopen("data.txt", "r"), &fclose);

// Stateless lambda deleter (zero overhead via EBO in C++20)
auto deleter = [](FILE* f) { if (f) fclose(f); };
std::unique_ptr<FILE, decltype(deleter)> f2(fopen("data.txt", "r"), deleter);

// Functor deleter (zero overhead if empty)
struct FileCloser {
    void operator()(FILE* f) const { if (f) fclose(f); }
};
std::unique_ptr<FILE, FileCloser> f3(fopen("data.txt", "r"));
// No need to pass deleter instance - default-constructed
```

---

## Self-Assessment

### Q1: Wrap a POSIX file descriptor in `unique_ptr` with `close` as the deleter

File descriptors are integers, not pointers, so you cannot point a `unique_ptr` directly at them. The common workarounds are to wrap the fd in a small struct, or to heap-allocate the integer and use a custom deleter. Both approaches are shown here.

```cpp
#include <iostream>
#include <memory>
#include <cstring>

// Simulate POSIX API (cross-platform demonstration)
#ifdef _WIN32
#include <io.h>
#include <fcntl.h>
#define POSIX_OPEN  _open
#define POSIX_CLOSE _close
#define POSIX_WRITE _write
#define O_FLAGS     (_O_CREAT | _O_WRONLY | _O_TRUNC)
#define O_PERMS     (_S_IREAD | _S_IWRITE)
#else
#include <unistd.h>
#include <fcntl.h>
#define POSIX_OPEN  open
#define POSIX_CLOSE close
#define POSIX_WRITE write
#define O_FLAGS     (O_CREAT | O_WRONLY | O_TRUNC)
#define O_PERMS     0644
#endif

// Custom deleter for file descriptors
// We cannot use unique_ptr<int> directly because -1 is the invalid value,
// not nullptr. We wrap fd in a struct.
struct FdWrapper {
    int fd;
};

struct FdCloser {
    void operator()(FdWrapper* w) const {
        if (w && w->fd >= 0) {
            std::cout << "  Closing fd " << w->fd << "\n";
            POSIX_CLOSE(w->fd);
        }
        delete w;
    }
};

// Alternative: use unique_ptr<int, ...> with a sentinel approach
struct RawFdCloser {
    void operator()(int* fd) const {
        if (fd && *fd >= 0) {
            std::cout << "  Closing raw fd " << *fd << "\n";
            POSIX_CLOSE(*fd);
        }
        delete fd;
    }
};

int main() {
    std::cout << "=== POSIX fd with unique_ptr ===\n\n";

    // Approach 1: Wrapper struct with custom deleter
    {
        int raw = POSIX_OPEN("test_q1.txt", O_FLAGS, O_PERMS);
        if (raw < 0) {
            std::cerr << "open failed\n";
            return 1;
        }
        std::cout << "Opened fd " << raw << "\n";

        auto fd = std::unique_ptr<FdWrapper, FdCloser>(new FdWrapper{raw});

        const char* msg = "Hello from unique_ptr!\n";
        POSIX_WRITE(fd->fd, msg, static_cast<unsigned>(strlen(msg)));
        std::cout << "Wrote " << strlen(msg) << " bytes\n";

        // fd automatically closed at scope exit
    }
    std::cout << "\n";

    // Approach 2: Direct int* with custom deleter
    {
        int raw = POSIX_OPEN("test_q1b.txt", O_FLAGS, O_PERMS);
        if (raw < 0) {
            std::cerr << "open failed\n";
            return 1;
        }

        auto fd = std::unique_ptr<int, RawFdCloser>(new int(raw));
        const char* msg = "Second file\n";
        POSIX_WRITE(*fd, msg, static_cast<unsigned>(strlen(msg)));
    }

    std::cout << "\nBoth files closed automatically via RAII.\n";
    return 0;
}
```

The wrapper-struct approach is more common in practice because it keeps the invalid-sentinel logic (`fd >= 0`) inside the deleter where it belongs, rather than scattered through the calling code.

### Q2: Show how a lambda deleter enables `unique_ptr` to manage COM interface pointers

COM objects use reference counting through `AddRef` and `Release` instead of `delete`. A captureless lambda wrapping `Release` is a natural fit - and because the lambda is stateless, it incurs no size overhead on the `unique_ptr`.

```cpp
#include <iostream>
#include <memory>
#include <cstdint>

// Simulated COM-like interface (no Windows dependency)
struct IUnknown {
    uint32_t ref_count = 1;

    uint32_t AddRef() {
        return ++ref_count;
    }
    uint32_t Release() {
        std::cout << "  Release called, ref_count " << ref_count
                  << " -> " << (ref_count - 1) << "\n";
        if (--ref_count == 0) {
            std::cout << "  Object destroyed via Release()\n";
            delete this;
            return 0;
        }
        return ref_count;
    }
};

struct IWidget : IUnknown {
    void DoWork() { std::cout << "  IWidget::DoWork()\n"; }
};

// Simulated COM factory
IWidget* CreateWidget() {
    std::cout << "  COM object created (ref_count=1)\n";
    return new IWidget();
}

int main() {
    std::cout << "=== Lambda deleter for COM pointers ===\n\n";

    // Lambda captures nothing -> stateless -> zero overhead via EBO
    auto com_deleter = [](IUnknown* p) {
        if (p) p->Release();
    };

    {
        // Wrap COM pointer with lambda deleter
        std::unique_ptr<IWidget, decltype(com_deleter)> widget(
            CreateWidget(), com_deleter);

        widget->DoWork();

        // Simulated AddRef/Release cycle (another reference)
        widget->AddRef();
        std::cout << "  After AddRef: ref_count = " << widget->ref_count << "\n";
        widget->Release();  // Manual release of the extra ref

        // unique_ptr destructor calls Release() via lambda
        std::cout << "  Leaving scope...\n";
    }
    std::cout << "\n";

    // Size comparison
    std::cout << "Size of unique_ptr<IWidget>:                "
              << sizeof(std::unique_ptr<IWidget>) << " bytes\n";
    std::cout << "Size with lambda deleter:                   "
              << sizeof(std::unique_ptr<IWidget, decltype(com_deleter)>) << " bytes\n";
    std::cout << "Size with function pointer:                 "
              << sizeof(std::unique_ptr<IWidget, void(*)(IUnknown*)>) << " bytes\n";

    return 0;
}
// Expected output (64-bit):
//   COM object created (ref_count=1)
//   IWidget::DoWork()
//   After AddRef: ref_count = 2
//   Release called, ref_count 2 -> 1
//   Leaving scope...
//   Release called, ref_count 1 -> 0
//   Object destroyed via Release()
//
//   Size of unique_ptr<IWidget>:                8 bytes
//   Size with lambda deleter:                   8 bytes  (EBO!)
//   Size with function pointer:                 16 bytes
```

Notice that the captureless lambda version has the same size as the default deleter version - 8 bytes on a 64-bit system. The function pointer version doubles in size because the pointer to `Release` has to be stored alongside the object pointer.

### Q3: Explain how custom deleters affect the size of `unique_ptr` (stateless vs stateful)

The size impact of different deleter styles is a common surprise. Stateless types (empty struct, captureless lambda) cost nothing thanks to EBO. Function pointers always add one pointer's worth. Stateful deleters add however many bytes their state requires.

```cpp
#include <iostream>
#include <memory>
#include <functional>

struct Widget {
    int value;
};

// 1. Default deleter - stateless empty class (EBO applies)
// sizeof == sizeof(Widget*)

// 2. Stateless functor - empty class (EBO applies)
struct StatelessDeleter {
    void operator()(Widget* p) const { delete p; }
};

// 3. Function pointer - stores a pointer alongside the object pointer
using FnPtrDeleter = void(*)(Widget*);

// 4. Stateful functor - stores state alongside the object pointer
struct StatefulDeleter {
    std::string log_prefix;  // 32 bytes on typical implementations
    void operator()(Widget* p) const {
        std::cout << log_prefix << " deleting Widget\n";
        delete p;
    }
};

// 5. std::function - heavy: contains type-erased callable + possible heap alloc
using StdFuncDeleter = std::function<void(Widget*)>;

int main() {
    std::cout << "=== Custom Deleter Size Impact ===\n\n";

    std::cout << "sizeof(Widget*) = " << sizeof(Widget*) << " bytes\n\n";

    // Size comparison table
    std::cout << "Deleter Type                          sizeof(unique_ptr)\n";
    std::cout << "----------------------------------------------------\n";

    std::cout << "default_delete<Widget> (default)       "
              << sizeof(std::unique_ptr<Widget>) << " bytes\n";

    std::cout << "StatelessDeleter (empty struct)        "
              << sizeof(std::unique_ptr<Widget, StatelessDeleter>) << " bytes\n";

    // Captureless lambda
    auto lambda = [](Widget* p) { delete p; };
    std::cout << "Captureless lambda                     "
              << sizeof(std::unique_ptr<Widget, decltype(lambda)>) << " bytes\n";

    std::cout << "Function pointer void(*)(Widget*)      "
              << sizeof(std::unique_ptr<Widget, FnPtrDeleter>) << " bytes\n";

    std::cout << "StatefulDeleter (has std::string)       "
              << sizeof(std::unique_ptr<Widget, StatefulDeleter>) << " bytes\n";

    std::cout << "std::function<void(Widget*)>           "
              << sizeof(std::unique_ptr<Widget, StdFuncDeleter>) << " bytes\n";

    // Capturing lambda
    int x = 42;
    auto capturing = [x](Widget* p) { (void)x; delete p; };
    std::cout << "Capturing lambda [x]                   "
              << sizeof(std::unique_ptr<Widget, decltype(capturing)>) << " bytes\n";

    std::cout << "\n=== Key Takeaways ===\n";
    std::cout << "- Stateless deleters (empty struct, captureless lambda): same size as raw ptr\n";
    std::cout << "- Function pointer: adds sizeof(void*) - pointer stored alongside\n";
    std::cout << "- Stateful functor: adds sizeof(state) - stored inline in unique_ptr\n";
    std::cout << "- std::function: heaviest - type erasure overhead + possible heap alloc\n";
    std::cout << "- PREFER stateless functors or captureless lambdas for zero overhead\n";

    return 0;
}
// Expected output (64-bit, typical):
//   sizeof(Widget*) = 8 bytes
//
//   default_delete<Widget> (default)       8 bytes
//   StatelessDeleter (empty struct)        8 bytes
//   Captureless lambda                     8 bytes
//   Function pointer void(*)(Widget*)      16 bytes
//   StatefulDeleter (has std::string)       40 bytes (8 + 32)
//   std::function<void(Widget*)>           40 bytes
//   Capturing lambda [x]                   16 bytes (8 + 4, padded to 8)
```

The takeaway is simple: if you need a deleter, reach for an empty functor struct or a captureless lambda first. Reserve function pointers for cases where you need to select the deleter at runtime, and avoid `std::function` as a deleter unless flexibility genuinely outweighs the overhead.

---

## Notes

- Prefer **stateless functor** or **captureless lambda** deleters - zero size overhead via EBO.
- Function pointer deleters double the size of `unique_ptr` (store pointer + function pointer).
- For C APIs, consider a helper template:

  ```cpp
  template<auto Fn> struct CDeleter {
      template<class T> void operator()(T* p) const { Fn(p); }
  };
  // Usage: unique_ptr<FILE, CDeleter<&fclose>> file(fopen(...));
  ```

- `unique_ptr` with custom deleter is still move-only - ownership transfer works the same way.
- When wrapping integer handles (fd, socket), use a wrapper struct since `unique_ptr` requires a pointer type.
