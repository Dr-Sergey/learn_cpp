# Know the C++ Core Guidelines' resource management checklist

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-resource>  

---

## Topic Overview

The C++ Core Guidelines resource management section (R.1-R.37) codifies best practices for handling memory, file handles, locks, and other resources. Think of it as a distilled checklist of the lessons the C++ community learned the hard way over decades. If you follow these rules, you will avoid the vast majority of resource leaks and lifetime bugs.

### Key Rules Summary

| Rule | Summary |
| --- | --- |
| R.1 | Manage resources automatically using RAII |
| R.2 | In interfaces, use raw pointers for individual objects (non-owning) |
| R.3 | Raw pointer (`T*`) does NOT convey ownership |
| R.4 | Raw reference (`T&`) does NOT convey ownership |
| R.5 | Prefer scoped objects; don't heap-allocate unnecessarily |
| R.6 | Avoid non-const global variables |
| R.10 | Avoid `malloc()`/`free()` |
| R.11 | Avoid calling `new`/`delete` explicitly |
| R.13 | Perform at most one explicit resource allocation per statement |
| R.20 | Use `unique_ptr` for exclusive ownership |
| R.21 | Prefer `unique_ptr` over `shared_ptr` unless sharing |

---

## Self-Assessment

### Q1: Enumerate and explain the Core Guidelines R.1 through R.11

**R.1: Manage resources automatically using resource handles and RAII.**

The key insight here is that C++ destructors run deterministically when an object goes out of scope - even when an exception is thrown. If you wrap every resource in an RAII handle, cleanup is automatic and exception-safe.

```cpp
// BAD:
void bad() {
    int* p = new int[100];  // manual allocation
    process(p);              // if this throws, p leaks!
    delete[] p;
}

// GOOD:
void good() {
    auto p = std::make_unique<int[]>(100);  // RAII
    process(p.get());  // if this throws, unique_ptr cleans up
}  // automatic cleanup
```

The `unique_ptr` destructor fires no matter how the function exits.

**R.2: In interfaces, use raw pointers to denote individual objects (only).**

A raw pointer in a function signature should mean "a pointer to one thing." If you need to pass a sequence of items, use `std::span` - it carries the size alongside the pointer.

```cpp
void process(int* single_item);       // OK: pointer to one int
// Use span<int> for arrays, not int*
void process(std::span<int> items);   // GOOD: safe array
```

**R.3: A raw pointer (T*) is non-owning.**

Raw pointers carry no ownership semantics by themselves. When you see `Widget*` in a signature, the convention should be that it is just a view - someone else is responsible for the lifetime. If you want to express ownership, say so explicitly with a smart pointer.

```cpp
// BAD: Is caller or function responsible for delete?
void confusing(Widget* w);  // ownership unclear

// GOOD: ownership explicit
void observe(Widget* w);                        // non-owning
void take_ownership(std::unique_ptr<Widget> w);  // owning
```

**R.4: A raw reference (T&) is non-owning.**

References are even more clearly non-owning than pointers, but it is worth stating for completeness.

```cpp
void use(Widget& w);  // borrows w, doesn't own it
```

**R.5: Prefer scoped objects, don't heap-allocate unnecessarily.**

Every `new` requires a `delete` somewhere. Stack objects have automatic lifetime and cost nothing to "clean up." Only reach for heap allocation when you genuinely need it (dynamic lifetime, large size, polymorphism).

```cpp
// BAD:
auto w = new Widget();  // unnecessary heap allocation
delete w;

// GOOD:
Widget w;  // stack allocation, automatic lifetime
```

**R.6: Avoid non-const global variables.**

Mutable globals are shared state - they are hard to reason about in multi-threaded code, and they make testing a nightmare because each test can affect the next.

```cpp
// BAD:
int global_counter = 0;  // mutable global - thread-unsafe, hard to test

// GOOD:
int get_count() {
    static int counter = 0;  // controlled access
    return ++counter;
}
```

**R.10: Avoid malloc()/free().**

`malloc` and `free` are C functions. They allocate raw memory but know nothing about C++ objects - no constructors, no destructors. When you `malloc` a C++ object, it is never properly constructed. Mixing these with `new`/`delete` is also undefined behavior.

```cpp
// BAD:
auto p = (int*)malloc(sizeof(int));  // no constructor called!
free(p);                              // no destructor called!

// GOOD:
auto p = std::make_unique<int>(42);  // constructor + destructor
```

**R.11: Avoid calling new and delete explicitly.**

Even if you pair every `new` with a `delete`, an exception thrown between them will skip the `delete`. Smart pointers eliminate this entire class of bug.

```cpp
// BAD:
auto p = new Widget();
// ... lots of code that might throw ...
delete p;  // might never reach here!

// GOOD:
auto p = std::make_unique<Widget>();
// automatic cleanup regardless of exceptions
```

### Q2: Apply rule R.3 to fix a raw-pointer ownership API

The problem with raw pointers carrying ownership is that it is entirely implicit - there is nothing in the type that tells the reader (or the compiler) who is responsible for cleanup. Here is a before-and-after showing how to make ownership explicit.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

// BAD: ownership ambiguous
class PluginManagerBad {
    std::vector<void*> plugins_;  // who owns these?
public:
    void add(void* plugin) {
        plugins_.push_back(plugin);  // takes pointer but doesn't own?
    }
    ~PluginManagerBad() {
        // Do we delete? Don't delete? UB either way!
    }
};

// GOOD: ownership explicit with unique_ptr
class Plugin {
public:
    virtual ~Plugin() = default;
    virtual std::string name() const = 0;
    virtual void execute() = 0;
};

class PluginManager {
    std::vector<std::unique_ptr<Plugin>> plugins_;  // owns plugins
public:
    // Transfer ownership: caller gives up the plugin
    void add(std::unique_ptr<Plugin> plugin) {
        std::cout << "Added plugin: " << plugin->name() << '\n';
        plugins_.push_back(std::move(plugin));
    }

    // Non-owning observation
    Plugin* get(size_t index) {
        return plugins_[index].get();  // raw pointer = non-owning
    }

    void run_all() {
        for (auto& p : plugins_)
            p->execute();
    }
};  // destructor automatically deletes all plugins

class Logger : public Plugin {
public:
    std::string name() const override { return "Logger"; }
    void execute() override { std::cout << "Logging...\n"; }
};

int main() {
    PluginManager mgr;
    mgr.add(std::make_unique<Logger>());  // ownership transfer is explicit
    mgr.run_all();
}
// Expected output:
// Added plugin: Logger
// Logging...
```

Notice that `get()` still returns a raw pointer - that is intentional. A raw pointer in the return means "I'm lending you a view; the manager still owns it."

### Q3: Identify a resource management violation and propose a fix

This example has two overlapping violations: it uses the C file API directly (R.10), and it also does raw `new`/`delete` (R.11). The dangerous part is that if anything between the allocation and the cleanup throws, both resources leak. RAII fixes both at once.

```cpp
#include <cstdio>
#include <iostream>
#include <memory>
#include <stdexcept>

// VIOLATION: manual resource management + exception unsafety
void process_file_bad(const char* path) {
    FILE* f = fopen(path, "r");  // R.10 violation: C API
    if (!f) return;

    char* buffer = new char[1024];  // R.11 violation: raw new
    // If fread or processing throws, buffer leaks!

    size_t n = fread(buffer, 1, 1024, f);
    // process(buffer, n);  // might throw!

    delete[] buffer;  // R.11: manual delete
    fclose(f);        // might not reach here
}

// FIX: RAII everywhere
void process_file_good(const char* path) {
    // R.1: RAII for file handle
    auto deleter = [](FILE* f) { if (f) fclose(f); };
    std::unique_ptr<FILE, decltype(deleter)> f(fopen(path, "r"), deleter);
    if (!f) return;

    // R.11: no raw new
    auto buffer = std::make_unique<char[]>(1024);

    size_t n = fread(buffer.get(), 1, 1024, f.get());
    // If this throws, both f and buffer are cleaned up automatically

    std::cout << "Read " << n << " bytes\n";
}  // Both resources freed automatically

int main() {
    // Create test file
    { auto f = fopen("test.txt", "w"); fputs("Hello RAII", f); fclose(f); }

    process_file_good("test.txt");
    std::remove("test.txt");
}
// Expected output:
// Read 10 bytes
```

The custom deleter lambda passed to `unique_ptr` is the standard trick for wrapping C-style handles that use a non-`delete` cleanup function. Once you have that wrapper, `unique_ptr` handles the rest automatically.

---

## Notes

- R.13 (one allocation per statement) prevents leaks in argument evaluation: `f(make_unique<A>(), make_unique<B>())` is safe; `f(new A(), new B())` can leak if one `new` throws.
- Modern C++ virtually eliminates the need for `new`/`delete` - use `make_unique`, `make_shared`, containers.
- `std::span` and `std::string_view` replace non-owning `T*`/`const char*` in interfaces.
- Run clang-tidy with `cppcoreguidelines-*` checks to auto-detect violations.
