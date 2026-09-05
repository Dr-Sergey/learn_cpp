# Handle DLL and SO loading differences: LoadLibrary vs dlopen

**Category:** Cross Platform Development  
**Standard:** C++17  
**Reference:** <https://man7.org/linux/man-pages/man3/dlopen.3.html>  

---

## Topic Overview

### Dynamic Loading - Platform Differences

Both Windows and POSIX systems support loading shared libraries at runtime, but the APIs differ significantly. Understanding the differences helps you write a portable wrapper that hides the details from application code:

| Feature | Windows | Linux/macOS |
| --- | --- | --- |
| Library extension | `.dll` | `.so` (Linux), `.dylib` (macOS) |
| Load function | `LoadLibraryW` | `dlopen` |
| Get symbol | `GetProcAddress` | `dlsym` |
| Unload | `FreeLibrary` | `dlclose` |
| Error reporting | `GetLastError()` | `dlerror()` |
| Symbol visibility | Explicit (`__declspec(dllexport)`) | Default visible (or `__attribute__((visibility("default")))`) |
| Name mangling | C++ names mangled differently | C++ names mangled differently |

### Cross-Platform Dynamic Library Wrapper

The wrapper below handles the platform differences in one place. One subtlety worth noting: on Windows, `LoadLibraryW` takes a wide-character path, so we need to convert from UTF-8 first using `MultiByteToWideChar`. On POSIX, `dlopen` takes a plain `char*`. The RAII design ensures the library is unloaded exactly once, when the wrapper is destroyed:

```cpp
#include <string>
#include <stdexcept>

#ifdef _WIN32
    #define WIN32_LEAN_AND_MEAN
    #include <windows.h>
    using LibHandle = HMODULE;
#else
    #include <dlfcn.h>
    using LibHandle = void*;
#endif

class DynamicLibrary {
    LibHandle handle_ = nullptr;
    std::string path_;

public:
    explicit DynamicLibrary(const std::string& path) : path_(path) {
#ifdef _WIN32
        // Convert UTF-8 path to wide string for Windows
        int wlen = MultiByteToWideChar(CP_UTF8, 0, path.c_str(), -1, nullptr, 0);
        std::wstring wpath(wlen, L'\0');
        MultiByteToWideChar(CP_UTF8, 0, path.c_str(), -1, wpath.data(), wlen);
        handle_ = LoadLibraryW(wpath.c_str());
        if (!handle_) {
            throw std::runtime_error("LoadLibrary failed: " + path +
                                     " (error " + std::to_string(GetLastError()) + ")");
        }
#else
        handle_ = dlopen(path.c_str(), RTLD_NOW | RTLD_LOCAL);
        if (!handle_) {
            throw std::runtime_error("dlopen failed: " + std::string(dlerror()));
        }
#endif
    }

    ~DynamicLibrary() {
        if (handle_) {
#ifdef _WIN32
            FreeLibrary(handle_);
#else
            dlclose(handle_);
#endif
        }
    }

    DynamicLibrary(const DynamicLibrary&) = delete;
    DynamicLibrary& operator=(const DynamicLibrary&) = delete;

    DynamicLibrary(DynamicLibrary&& other) noexcept
        : handle_(other.handle_), path_(std::move(other.path_)) {
        other.handle_ = nullptr;
    }

    template<typename Func>
    Func get_function(const char* name) const {
        void* sym = nullptr;
#ifdef _WIN32
        sym = reinterpret_cast<void*>(GetProcAddress(handle_, name));
#else
        sym = dlsym(handle_, name);
#endif
        if (!sym) {
            throw std::runtime_error(std::string("Symbol not found: ") + name);
        }
        return reinterpret_cast<Func>(sym);
    }

    [[nodiscard]] bool valid() const { return handle_ != nullptr; }
};
```

### Using the Wrapper

With the wrapper in place, loading a plugin becomes straightforward. The `extern "C"` declaration on the function types is critical - without it, name mangling would make `GetProcAddress`/`dlsym` unable to find the symbols. The `~DynamicLibrary` destructor takes care of the `FreeLibrary`/`dlclose` call automatically:

```cpp
// Plugin interface (shared header)
extern "C" {  // Prevent name mangling!
    using CreatePluginFunc = IPlugin* (*)();
    using DestroyPluginFunc = void (*)(IPlugin*);
}

// Loading a plugin
void load_plugin(const std::string& plugin_path) {
    DynamicLibrary lib(plugin_path);

    auto create = lib.get_function<CreatePluginFunc>("create_plugin");
    auto destroy = lib.get_function<DestroyPluginFunc>("destroy_plugin");

    IPlugin* plugin = create();
    plugin->execute();
    destroy(plugin);
    // ~DynamicLibrary unloads the shared library
}
```

### Symbol Visibility - The Key Difference

On **Linux**, all symbols are exported by default. On **Windows**, nothing is exported unless explicitly marked. This asymmetry is one of the most common sources of "works on Linux, fails on Windows" plugin bugs. The portable solution is a visibility macro that resolves to the right annotation on each platform:

```cpp
// Export macro that works on both platforms
#if defined(_WIN32)
    #ifdef MYLIB_EXPORTS
        #define MYLIB_API __declspec(dllexport)
    #else
        #define MYLIB_API __declspec(dllimport)
    #endif
#else
    #define MYLIB_API __attribute__((visibility("default")))
#endif

// Usage
class MYLIB_API Widget {
public:
    void do_work();
};

extern "C" MYLIB_API IPlugin* create_plugin();
```

On Linux, combine with `-fvisibility=hidden` to hide all symbols by default, which matches Windows behavior and brings the benefits described in Q2 below:

```cmake
target_compile_options(mylib PRIVATE -fvisibility=hidden)
target_compile_definitions(mylib PRIVATE MYLIB_EXPORTS)
```

### Platform-Specific Library Search Paths

Library filenames and the directories the OS searches are platform-specific. It helps to centralize the naming convention in a small helper rather than scattering `#ifdef` checks throughout call sites:

```cpp
std::string get_plugin_path(const std::string& name) {
#ifdef _WIN32
    return name + ".dll";
#elif defined(__APPLE__)
    return "lib" + name + ".dylib";
#else
    return "lib" + name + ".so";
#endif
}

// Search paths:
// Windows: executable directory, system PATH, current directory
// Linux: LD_LIBRARY_PATH, /usr/lib, /usr/local/lib, RPATH
// macOS: DYLD_LIBRARY_PATH, RPATH, /usr/lib
```

### CMake for Shared Libraries

CMake handles most of the shared library boilerplate, but you still need to set RPATH on Linux so the OS can find the library at runtime, and copy the DLL next to the executable on Windows (since Windows searches the executable's directory first):

```cmake
add_library(myplugin SHARED src/plugin.cpp)

# Set RPATH so the executable finds the library at runtime
if(UNIX)
    set_target_properties(myplugin PROPERTIES
        INSTALL_RPATH "$ORIGIN"     # Linux: look in same dir as executable
        BUILD_RPATH "$ORIGIN"
    )
endif()

# Windows: DLLs must be in same dir as exe (or in PATH)
if(WIN32)
    # Copy DLL next to executable after build
    add_custom_command(TARGET myapp POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            $<TARGET_FILE:myplugin>
            $<TARGET_FILE_DIR:myapp>
    )
endif()
```

---

## Self-Assessment

### Q1: Why must plugin functions use `extern "C"`

C++ **name mangling** encodes function signatures into symbol names (e.g., `_Z13create_pluginv`). The mangling scheme is compiler-specific and ABI-specific. `extern "C"` disables mangling, producing predictable symbol names (e.g., `create_plugin`) that can be found with `dlsym`/`GetProcAddress` regardless of which compiler built the plugin or the host application.

### Q2: Why use `-fvisibility=hidden` on Linux

Without it, **all** symbols in a shared library are exported. This causes several real problems:

- **Slower loading** - the dynamic linker must process thousands of symbols at startup
- **Symbol collisions** - two libraries might export the same symbol name, causing hard-to-debug crashes
- **Larger binary** - symbol tables take significant space
- **No encapsulation** - internal functions are accessible to anyone who loads the library

With `-fvisibility=hidden`, only symbols explicitly marked with `__attribute__((visibility("default")))` (or the `MYLIB_API` macro) are exported - matching Windows behavior and fixing all of the above.

### Q3: How do you handle library unloading safely

The main danger is **use-after-unload**: calling a function pointer from an unloaded library causes undefined behavior, usually a crash. The safe pattern is:

- Use RAII: the `DynamicLibrary` wrapper unloads in its destructor, so scope controls lifetime
- Ensure all objects created by the plugin are destroyed **before** unloading
- Use the factory pattern: `create_plugin()` allocates, `destroy_plugin()` deallocates
- Never store function pointers from a plugin beyond the library's lifetime

---

## Notes

- Use `RTLD_NOW` (not `RTLD_LAZY`) to catch missing symbols at load time rather than at first call.
- Windows DLLs have `DllMain` - avoid complex initialization in it (loader lock issues).
- On macOS, use `@rpath` and `install_name_tool` for library search paths.
- `std::filesystem::path` helps construct platform-correct library paths.
- Consider using a plugin framework (e.g., Boost.DLL) for production plugin systems.
