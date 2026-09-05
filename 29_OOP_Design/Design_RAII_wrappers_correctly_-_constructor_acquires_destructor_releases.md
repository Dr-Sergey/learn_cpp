# Design RAII wrappers correctly - constructor acquires, destructor releases

**Category:** OOP Design  
**Standard:** C++11  

---

## Topic Overview

**RAII** (Resource Acquisition Is Initialization) is C++'s fundamental resource management idiom. The idea is beautifully simple: tie a resource's lifetime to an object's lifetime. When the object is constructed, it acquires the resource. When the object is destroyed - whether by normal scope exit, early return, or exception - the destructor releases the resource automatically. You never forget to clean up because there is nothing to forget.

```cpp
Construct  ->  Resource acquired (file, socket, lock, memory)
Scope exit ->  Destructor called -> Resource released
                  (even on exception!)
```

The standard library is full of RAII wrappers. You use them every day:

| RAII Wrapper | Resource | Standard Library |
| --- | --- | --- |
| `unique_ptr<T>` | Heap memory | `<memory>` |
| `shared_ptr<T>` | Shared heap memory | `<memory>` |
| `lock_guard<M>` | Mutex lock | `<mutex>` |
| `fstream` | File handle | `<fstream>` |
| `jthread` | Thread join | `<thread>` (C++20) |

The reason RAII matters so much is exceptions. Without it, every function that might throw needs careful cleanup on every error path. With RAII, destructors run automatically during stack unwinding - you get correct cleanup for free.

---

## Self-Assessment

### Q1: Write RAII wrappers for a file descriptor and a C library handle

The classic situation: you're calling a C API that returns a raw handle you must close manually. The solution is to wrap it in a class whose destructor calls the close function. Notice the move constructor uses `std::exchange` to transfer ownership cleanly - the moved-from object gets the sentinel value `-1`, which the destructor safely ignores.

**Answer:**

```cpp
#include <unistd.h>  // POSIX (or use _close on Windows)
#include <fcntl.h>
#include <stdexcept>
#include <utility>
#include <cstdio>

// RAII wrapper for POSIX file descriptor
class FileDescriptor {
public:
    explicit FileDescriptor(const char* path, int flags)
        : fd_(::open(path, flags)) {
        if (fd_ < 0)
            throw std::runtime_error("Failed to open file");
    }

    ~FileDescriptor() {
        if (fd_ >= 0) ::close(fd_);
    }

    // Move-only
    FileDescriptor(FileDescriptor&& o) noexcept
        : fd_(std::exchange(o.fd_, -1)) {}

    FileDescriptor& operator=(FileDescriptor&& o) noexcept {
        if (this != &o) {
            if (fd_ >= 0) ::close(fd_);
            fd_ = std::exchange(o.fd_, -1);
        }
        return *this;
    }

    // Non-copyable
    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator=(const FileDescriptor&) = delete;

    int get() const noexcept { return fd_; }
    explicit operator bool() const noexcept { return fd_ >= 0; }

private:
    int fd_;
};

// Generic RAII for C handles using unique_ptr + custom deleter
#include <memory>

struct FileCloser {
    void operator()(FILE* f) const noexcept {
        if (f) std::fclose(f);
    }
};
using UniqueFile = std::unique_ptr<FILE, FileCloser>;

UniqueFile open_file(const char* path, const char* mode) {
    UniqueFile f(std::fopen(path, mode));
    if (!f) throw std::runtime_error("fopen failed");
    return f;
}
```

The `UniqueFile` alias at the bottom is worth appreciating: you get all of `unique_ptr`'s move semantics and automatic cleanup for free, just by providing a custom deleter. This pattern handles the vast majority of C API resources with minimal boilerplate.

### Q2: Show a scoped guard for arbitrary cleanup actions

Sometimes you don't own a type that can be wrapped neatly. You just need "run this lambda when we leave this scope." That's what `ScopeGuard` does. The `dismiss()` method is the escape hatch - call it when the operation succeeded and you don't want the cleanup to run.

**Answer:**

```cpp
#include <functional>
#include <iostream>

class ScopeGuard {
public:
    explicit ScopeGuard(std::function<void()> cleanup)
        : cleanup_(std::move(cleanup)) {}

    ~ScopeGuard() {
        if (cleanup_) cleanup_();
    }

    void dismiss() noexcept { cleanup_ = nullptr; }

    ScopeGuard(ScopeGuard&& o) noexcept
        : cleanup_(std::exchange(o.cleanup_, nullptr)) {}

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;

private:
    std::function<void()> cleanup_;
};

// Macro for unique names
#define CONCAT_(a, b) a##b
#define CONCAT(a, b) CONCAT_(a, b)
#define SCOPE_EXIT auto CONCAT(_guard_, __LINE__) = ScopeGuard

void process_data() {
    auto* raw = new int[100];
    SCOPE_EXIT([&] {
        delete[] raw;
        std::cout << "Cleaned up raw buffer\n";
    });

    // Do work that may throw...
    raw[0] = 42;
    // Automatically cleaned up at scope exit!
}

// Zero-overhead version using template instead of std::function
template<typename F>
class ScopeGuardT {
    F fn_;
    bool active_ = true;
public:
    explicit ScopeGuardT(F fn) noexcept : fn_(std::move(fn)) {}
    ~ScopeGuardT() { if (active_) fn_(); }
    void dismiss() noexcept { active_ = false; }
    ScopeGuardT(ScopeGuardT&& o) noexcept
        : fn_(std::move(o.fn_)), active_(std::exchange(o.active_, false)) {}
};

template<typename F>
ScopeGuardT<F> make_scope_guard(F fn) {
    return ScopeGuardT<F>(std::move(fn));
}
```

The `ScopeGuardT` template version at the bottom avoids the heap allocation that `std::function` might incur for captured lambdas. If this appears in a hot path, prefer `ScopeGuardT` via `make_scope_guard`.

### Q3: Show RAII for a complex multi-resource acquisition

When you need to acquire two resources in sequence - and the second one can fail - you must ensure the first is still released. The trick is to store each resource as a member RAII wrapper. If the second acquisition throws inside the constructor body, C++ will destroy all fully-constructed members in reverse order. So the database connection cleans itself up even if `tx_begin` throws, because `db_` was fully constructed before the throw happened.

**Answer:**

```cpp
#include <memory>
#include <stdexcept>
#include <iostream>

// Simulated C API resources
struct DbHandle { int id; };
struct TxHandle { int id; };

DbHandle* db_connect(const char*) { static DbHandle h{1}; return &h; }
void db_disconnect(DbHandle* h) { std::cout << "Disconnected db " << h->id << "\n"; }
TxHandle* tx_begin(DbHandle*) { static TxHandle t{1}; return &t; }
void tx_commit(TxHandle* t) { std::cout << "Committed tx " << t->id << "\n"; }
void tx_rollback(TxHandle* t) { std::cout << "Rolled back tx " << t->id << "\n"; }

// RAII wrapper for database connection + transaction
class DatabaseSession {
public:
    explicit DatabaseSession(const char* conn_str)
        : db_(db_connect(conn_str), db_disconnect)  // RAII step 1
    {
        if (!db_) throw std::runtime_error("Connection failed");
        // If tx_begin throws, db_ destructor still runs!
        tx_.reset(tx_begin(db_.get()));  // RAII step 2
    }

    void commit() {
        tx_commit(tx_.release());  // Release ownership
        committed_ = true;
    }

    ~DatabaseSession() {
        if (tx_) tx_rollback(tx_.get());  // Auto-rollback
    }

private:
    struct TxDeleter { void operator()(TxHandle*) const {} };  // tx_rollback in ~DatabaseSession
    std::unique_ptr<DbHandle, void(*)(DbHandle*)> db_;
    std::unique_ptr<TxHandle, TxDeleter> tx_;
    bool committed_ = false;
};

int main() {
    try {
        DatabaseSession session("host=localhost");
        // Do queries...
        session.commit();
        // If exception before commit -> auto-rollback!
    } catch (const std::exception& e) {
        std::cerr << e.what() << "\n";
    }
    return 0;
}
```

The commit-or-rollback pattern is very common: you want the transaction to roll back automatically unless you explicitly call `commit()`. Here `tx_.release()` transfers ownership out of the `unique_ptr` on success, so the destructor's `if (tx_)` check is false and no rollback happens.

---

## Notes

- **Rule 1:** Every resource ownership must be wrapped in an RAII type.
- **Rule 2:** Destructors must be `noexcept` - never throw from a destructor.
- Use `std::exchange(o.member_, sentinel)` in move constructors for clean ownership transfer.
- `unique_ptr` with a custom deleter handles most C API wrappers with minimal boilerplate.
- With multiple resources, order members so earlier members are cleaned up if later ones throw in the constructor.
- For two-phase commit (acquire/commit), use a guard that rolls back unless `dismiss()` is called.
