# Design Platform Abstraction Layers in C++

**Category:** Cross Platform Development  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/if#Constexpr_if>  

---

## Topic Overview

A Platform Abstraction Layer (PAL) encapsulates all OS-specific functionality behind a stable, platform-neutral interface. The goal is that application code never sees `#ifdef _WIN32` or a raw POSIX call - it only talks to the abstraction. Getting this right requires choosing the correct dispatch strategy, because each one carries distinct trade-offs in binary size, performance, and flexibility.

Compile-time selection via `if constexpr`, `#ifdef`, or policy-based templates eliminates dead code and enables inlining, but locks the platform choice at build time. Runtime dispatch through virtual interfaces or `std::function` allows plugin-style loading (for example, choosing a Vulkan vs. Metal rendering backend at startup) at the cost of an indirect call per dispatch. A mature PAL typically combines both: compile-time for leaf operations (byte order, path separator) and runtime for subsystem-level selection (rendering, audio).

| Strategy | Dispatch Cost | Binary Size | Extensibility | Testability |
| --- | --- | --- | --- | --- |
| `#ifdef` blocks | Zero | Minimal (dead code removed) | Recompile required | Hard to mock |
| `if constexpr` | Zero | Minimal | Recompile required | Moderate (traits injection) |
| Virtual interface | vtable indirection | All impls linked | Runtime plugin loading | Easy (mock classes) |
| `std::variant` + `visit` | Visit dispatch | All impls linked | Closed set of types | Moderate |
| Type-erased wrapper | Indirect call | All impls linked | Open set | Easy |

The factory pattern ties runtime dispatch together: a factory function returns a `std::unique_ptr<IPlatformService>` whose concrete type depends on the detected OS or a configuration flag. Combined with dependency injection, this makes subsystems independently testable. The diagram below shows the three-layer structure you are building toward:

```cpp
┌──────────────────────────────┐
│       Application Code       │
├──────────────────────────────┤
│     Platform Abstraction     │  <- stable interface
├──────┬──────┬────────────────┤
│ Win32│ POSIX│  Mock/Test     │  <- concrete implementations
└──────┴──────┴────────────────┘
```

Application code only ever sees the middle row. The concrete implementations beneath it are selected at build time or via a factory at startup.

---

## Self-Assessment

### Q1: Implement a compile-time PAL for getting the current thread ID using `if constexpr` and platform tags

The key design here is that the platform choice is baked in at compile time using tag types. The `CurrentPlatform` alias resolves to either `WindowsTag` or `PosixTag` based on preprocessor macros, and `if constexpr` inside the template selects the right code path. Because `if constexpr` discards the untaken branch entirely, you avoid the situation where non-Windows code tries to compile `GetCurrentThreadId()`:

```cpp
#include <cstdint>
#include <iostream>

// Platform tag types
struct WindowsTag {};
struct PosixTag   {};

#if defined(_WIN32)
using CurrentPlatform = WindowsTag;
#elif defined(__linux__) || defined(__APPLE__)
using CurrentPlatform = PosixTag;
#else
#error "Unsupported platform"
#endif

// Platform-specific headers included conditionally
#ifdef _WIN32
#include <windows.h>
#else
#include <pthread.h>
#endif

template <typename PlatformTag = CurrentPlatform>
class ThreadUtils {
public:
    static std::uint64_t current_thread_id() {
        if constexpr (std::is_same_v<PlatformTag, WindowsTag>) {
            #ifdef _WIN32
            return static_cast<std::uint64_t>(::GetCurrentThreadId());
            #else
            return 0; // never instantiated on non-Windows
            #endif
        } else if constexpr (std::is_same_v<PlatformTag, PosixTag>) {
            #ifndef _WIN32
            return static_cast<std::uint64_t>(::pthread_self());
            #else
            return 0;
            #endif
        }
    }

    static void set_thread_name([[maybe_unused]] const char* name) {
        if constexpr (std::is_same_v<PlatformTag, PosixTag>) {
            #if defined(__linux__)
            pthread_setname_np(pthread_self(), name);
            #elif defined(__APPLE__)
            pthread_setname_np(name);
            #endif
        }
        // Windows: SetThreadDescription requires linking to Kernel32
    }
};

int main() {
    auto tid = ThreadUtils<>::current_thread_id();
    std::cout << "Thread ID: " << tid << "\n";

    ThreadUtils<>::set_thread_name("worker-1");
    return 0;
}
```

Notice that the template parameter defaults to `CurrentPlatform`, so production code uses `ThreadUtils<>` while test code can inject a mock tag if needed.

### Q2: Design a runtime-dispatched file system watcher using an abstract interface and factory

Runtime dispatch through a virtual interface is the right choice here because the OS kernel APIs for file watching are completely different (`ReadDirectoryChangesW` on Windows, `inotify` on Linux, `FSEvents` on macOS). The application code only ever calls through the `IFileWatcher` interface. The `create_file_watcher()` factory is the single point of platform coupling - and the only place with an `#ifdef` the application ever needs to worry about:

```cpp
#include <filesystem>
#include <functional>
#include <memory>
#include <string>
#include <iostream>
#include <vector>

namespace pal {

enum class FileEvent { Created, Modified, Deleted };

using WatchCallback = std::function<void(const std::filesystem::path&, FileEvent)>;

// Abstract interface - stable ABI boundary
class IFileWatcher {
public:
    virtual ~IFileWatcher() = default;
    virtual void watch(const std::filesystem::path& dir, WatchCallback cb) = 0;
    virtual void stop() = 0;
    virtual bool is_running() const = 0;
};

// Win32 implementation (inotify on Linux, FSEvents on macOS)
#ifdef _WIN32
class Win32FileWatcher final : public IFileWatcher {
    bool running_ = false;
public:
    void watch(const std::filesystem::path& dir, WatchCallback cb) override {
        running_ = true;
        // Real impl: ReadDirectoryChangesW in a dedicated thread
        std::cout << "[Win32] Watching " << dir << "\n";
        (void)cb;
    }
    void stop() override { running_ = false; }
    bool is_running() const override { return running_; }
};
#else
class InotifyFileWatcher final : public IFileWatcher {
    bool running_ = false;
public:
    void watch(const std::filesystem::path& dir, WatchCallback cb) override {
        running_ = true;
        // Real impl: inotify_init / inotify_add_watch
        std::cout << "[inotify] Watching " << dir << "\n";
        (void)cb;
    }
    void stop() override { running_ = false; }
    bool is_running() const override { return running_; }
};
#endif

// Factory - the only point of platform coupling
std::unique_ptr<IFileWatcher> create_file_watcher() {
    #ifdef _WIN32
    return std::make_unique<Win32FileWatcher>();
    #else
    return std::make_unique<InotifyFileWatcher>();
    #endif
}

} // namespace pal

int main() {
    auto watcher = pal::create_file_watcher();

    watcher->watch("/tmp/data", [](const std::filesystem::path& p, pal::FileEvent e) {
        std::cout << "Event on " << p << " type=" << static_cast<int>(e) << "\n";
    });

    // ... application logic ...
    watcher->stop();
    return 0;
}
```

For testing, you can define a third implementation (`MockFileWatcher`) that records calls instead of touching the OS, and inject it without changing any application code.

### Q3: Build a PAL that selects implementations at compile time via policy templates and supports mock injection for testing

Policy-based design takes the compile-time selection idea further: instead of `if constexpr` inside a function, you parameterize the entire component on a policy type. This gives you zero-overhead abstraction - the compiler inlines the policy's `allocate` and `deallocate` calls directly - while making the mock path trivially injectable in tests by substituting a different policy type:

```cpp
#include <cstddef>
#include <cstring>
#include <iostream>
#include <vector>
#include <cassert>

// Policy interfaces (concepts, C++20)
template <typename T>
concept MemoryPolicy = requires(std::size_t n, void* p) {
    { T::allocate(n) } -> std::same_as<void*>;
    { T::deallocate(p, n) } -> std::same_as<void>;
};

// Platform policies
struct PosixMemory {
    static void* allocate(std::size_t n) {
        void* p = nullptr;
        #ifndef _WIN32
        // posix_memalign for 64-byte alignment (cache-line)
        if (::posix_memalign(&p, 64, n) != 0) return nullptr;
        #else
        p = ::_aligned_malloc(n, 64);
        #endif
        return p;
    }
    static void deallocate(void* p, [[maybe_unused]] std::size_t n) {
        #ifndef _WIN32
        ::free(p);
        #else
        ::_aligned_free(p);
        #endif
    }
};

// Mock policy for testing
struct MockMemory {
    inline static std::size_t total_allocated = 0;
    inline static std::size_t total_freed     = 0;

    static void* allocate(std::size_t n) {
        total_allocated += n;
        return std::malloc(n);
    }
    static void deallocate(void* p, std::size_t n) {
        total_freed += n;
        std::free(p);
    }
};

// PAL component parameterized by policy
template <MemoryPolicy MemPolicy>
class AlignedBuffer {
    void*       data_ = nullptr;
    std::size_t size_ = 0;
public:
    explicit AlignedBuffer(std::size_t n) : size_(n) {
        data_ = MemPolicy::allocate(n);
    }
    ~AlignedBuffer() {
        if (data_) MemPolicy::deallocate(data_, size_);
    }
    AlignedBuffer(const AlignedBuffer&) = delete;
    AlignedBuffer& operator=(const AlignedBuffer&) = delete;

    void*       data() { return data_; }
    std::size_t size() const { return size_; }
};

// Production alias
using PlatformBuffer = AlignedBuffer<PosixMemory>;

int main() {
    // Production use
    PlatformBuffer buf(4096);
    std::memset(buf.data(), 0, buf.size());
    std::cout << "Allocated " << buf.size() << " bytes (aligned)\n";

    // Test path - verify allocation tracking
    {
        AlignedBuffer<MockMemory> test_buf(1024);
        assert(MockMemory::total_allocated == 1024);
    }
    assert(MockMemory::total_freed == 1024);
    std::cout << "Mock allocator: all checks passed\n";

    return 0;
}
```

The `assert` checks confirm that the mock policy correctly tracked every allocation and deallocation. In a real test suite you would replace the `assert` calls with proper test assertions and run these checks as unit tests, completely independent of any OS.

---

## Notes

- Prefer `if constexpr` over `#ifdef` inside templates - it preserves type safety and enables better tooling support (IDE navigation, static analysis).
- Keep platform `#include` directives behind `#ifdef` guards to prevent compilation errors on non-target platforms.
- The factory pattern provides the cleanest ABI boundary for shared-library deployment; virtual interfaces are stable across compiler versions when no data members are exposed.
- Policy-based design (templates) yields zero-overhead abstractions but requires recompilation; use for performance-critical leaf code.
- Always provide a mock/stub implementation alongside real platform backends - untestable PAL code is a maintenance liability.
- Consider `std::variant<Win32Impl, PosixImpl>` + `std::visit` as a middle ground: closed set, no heap allocation, no vtable, but all implementations compiled into the binary.
