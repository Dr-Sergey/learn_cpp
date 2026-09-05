# Design plugin architectures with dynamic loading and abstract interfaces

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Plugin architecture** allows extending an application at runtime by loading shared libraries (`.so`/`.dll`) that implement a known interface. The host application defines abstract interfaces; plugins implement them and export a factory function. This enables third-party extensibility, modular deployment, and optional features without recompilation.

The reason this works across compilers and platforms is that the boundary between host and plugin uses **C linkage** - plain `extern "C"` functions with C-compatible types. C++ name mangling, vtable layouts, and exception handling all vary between compiler versions, so you can't safely pass `std::string` or catch exceptions across that boundary. What you *can* safely pass are C types and pointers to abstract interfaces.

### Plugin Architecture Components

| Component | Role | Example |
| --- | --- | --- |
| **Host** | Loads plugins, manages lifecycle | Application executable |
| **Plugin Interface** | Abstract classes plugins must implement | `ICodecPlugin`, `IFilter` |
| **Plugin** | Shared library implementing the interface | `libmp3_codec.so` |
| **Factory Function** | C-linkage entry point for plugin creation | `create_plugin()` |
| **Plugin Manager** | Discovery, loading, version checking | Scans plugin directory |

---

## Self-Assessment

### Q1: Implement a cross-platform plugin loading system

The platform abstraction here is straightforward: `LoadLibraryW`/`GetProcAddress` on Windows, `dlopen`/`dlsym` on Linux/macOS. The `DynamicLibrary` class wraps this in a RAII object so the handle is always released. The `PluginManager` scans a directory for shared libraries, loads each one, grabs the `create_plugin` and `destroy_plugin` function pointers by name, creates an instance, checks its version, and calls `initialize()`.

Notice that plugins are shut down in *reverse* order in the destructor - this mirrors how well-behaved systems tear down: last loaded, first unloaded.

**Answer:**

```cpp
#include <memory>
#include <string>
#include <vector>
#include <filesystem>
#include <iostream>

// === Plugin Interface (shared header between host and plugins) ===
// plugin_api.h
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual const char* name() const = 0;
    virtual int version() const = 0;
    virtual bool initialize() = 0;
    virtual void shutdown() = 0;
};

// Every plugin exports these C functions:
extern "C" {
    using CreatePluginFn = IPlugin*(*)();
    using DestroyPluginFn = void(*)(IPlugin*);
}

// === Cross-platform dynamic library wrapper ===
class DynamicLibrary {
public:
    explicit DynamicLibrary(const std::filesystem::path& path) {
#ifdef _WIN32
        handle_ = LoadLibraryW(path.wstring().c_str());
#else
        handle_ = dlopen(path.c_str(), RTLD_LAZY);
#endif
        if (!handle_)
            throw std::runtime_error("Failed to load: " + path.string());
    }

    ~DynamicLibrary() {
        if (handle_) {
#ifdef _WIN32
            FreeLibrary(static_cast<HMODULE>(handle_));
#else
            dlclose(handle_);
#endif
        }
    }

    template<typename Fn>
    Fn get_function(const char* name) const {
#ifdef _WIN32
        auto fn = GetProcAddress(static_cast<HMODULE>(handle_), name);
#else
        auto fn = dlsym(handle_, name);
#endif
        return reinterpret_cast<Fn>(fn);
    }

    DynamicLibrary(DynamicLibrary&&) = default;
    DynamicLibrary& operator=(DynamicLibrary&&) = default;
    DynamicLibrary(const DynamicLibrary&) = delete;

private:
    void* handle_ = nullptr;
};

// === Plugin Manager ===
class PluginManager {
public:
    struct LoadedPlugin {
        DynamicLibrary library;
        IPlugin* instance;
        DestroyPluginFn destroy;
    };

    void scan_directory(const std::filesystem::path& dir) {
        for (const auto& entry : std::filesystem::directory_iterator(dir)) {
            auto ext = entry.path().extension();
            if (ext == ".so" || ext == ".dll" || ext == ".dylib") {
                load_plugin(entry.path());
            }
        }
    }

    void load_plugin(const std::filesystem::path& path) {
        try {
            DynamicLibrary lib(path);
            auto create = lib.get_function<CreatePluginFn>("create_plugin");
            auto destroy = lib.get_function<DestroyPluginFn>("destroy_plugin");

            if (!create || !destroy) {
                std::cerr << "Invalid plugin: " << path << "\n";
                return;
            }

            IPlugin* plugin = create();
            if (plugin->version() < MIN_API_VERSION) {
                destroy(plugin);
                return;
            }

            plugin->initialize();
            plugins_.push_back({std::move(lib), plugin, destroy});
            std::cout << "Loaded: " << plugin->name()
                      << " v" << plugin->version() << "\n";
        } catch (const std::exception& e) {
            std::cerr << "Error loading " << path << ": " << e.what() << "\n";
        }
    }

    ~PluginManager() {
        // Shutdown in reverse order
        for (auto it = plugins_.rbegin(); it != plugins_.rend(); ++it) {
            it->instance->shutdown();
            it->destroy(it->instance);
        }
    }

    static constexpr int MIN_API_VERSION = 1;
private:
    std::vector<LoadedPlugin> plugins_;
};
```

The `get_function<Fn>(name)` template is doing a `reinterpret_cast` from `void*` (what `dlsym` returns) to a function pointer type. This is technically implementation-defined, but it works on every real platform and is the standard practice for dynamic loading.

### Q2: Write a concrete plugin as a shared library

Here's the other side of the contract: the plugin itself. It defines a class that implements `IPlugin`, then exports two `extern "C"` factory functions with platform-appropriate export attributes. The `extern "C"` is critical - without it, the function names get mangled differently by every compiler, so `get_function("create_plugin")` would fail.

The `-fvisibility=hidden` compiler flag in the build comment is equally important on Linux: it makes all symbols hidden by default, so only the explicitly marked `__attribute__((visibility("default")))` symbols are exported. This keeps the shared library's symbol table clean and prevents accidental symbol clashes with other plugins.

**Answer:**

```cpp
// === mp3_codec_plugin.cpp (compiled to libmp3_codec.so) ===
#include "plugin_api.h"
#include <iostream>

class Mp3CodecPlugin : public IPlugin {
public:
    const char* name() const override { return "MP3 Codec"; }
    int version() const override { return 2; }

    bool initialize() override {
        std::cout << "MP3 codec initialized\n";
        return true;
    }

    void shutdown() override {
        std::cout << "MP3 codec shutdown\n";
    }
};

// === C-linkage factory functions ===
extern "C" {

#ifdef _WIN32
__declspec(dllexport)
#else
__attribute__((visibility("default")))
#endif
IPlugin* create_plugin() {
    return new Mp3CodecPlugin();
}

#ifdef _WIN32
__declspec(dllexport)
#else
__attribute__((visibility("default")))
#endif
void destroy_plugin(IPlugin* p) {
    delete p;
}

}  // extern "C"

// Build: g++ -shared -fPIC -fvisibility=hidden mp3_codec_plugin.cpp
//        -o libmp3_codec.so
```

The reason `destroy_plugin` exists (rather than just calling `delete` on the host side) is the **same-side allocation rule**: memory must be freed by the same runtime that allocated it. If the plugin and host use different C runtimes (which can happen on Windows), a `delete` on the host side of memory that was `new`'d on the plugin side would corrupt the heap.

### Q3: Implement version negotiation and safe ABI boundaries

ABI safety at the plugin boundary requires careful discipline. The key idea in this example is the `struct_size` field: the plugin fills in `sizeof(PluginInfo)` as *it* knows it, and the host can compare that against what *it* expects. If the plugin was compiled with an older version of the header that had fewer fields, `struct_size` will be smaller, and the host knows not to read fields beyond that offset. This pattern lets you add fields to structs in future versions without breaking old plugins.

**Answer:**

```cpp
// === Safe ABI: use C-compatible structures at the boundary ===

// plugin_abi.h - frozen ABI, never change existing fields
struct PluginInfo {
    int struct_size;        // For forward compatibility
    int api_version;
    const char* name;
    const char* description;
    const char* author;
};

// Functions use C types only at the boundary
extern "C" {
    // Plugin fills in PluginInfo; host checks api_version
    using GetPluginInfoFn = bool(*)(PluginInfo* info);

    // Host passes opaque context; plugin stores it
    using InitPluginFn = bool(*)(void* host_context);
    using ShutdownPluginFn = void(*)();
}

// === Version negotiation in host ===
bool negotiate_version(DynamicLibrary& lib) {
    auto get_info = lib.get_function<GetPluginInfoFn>("get_plugin_info");
    if (!get_info) return false;

    PluginInfo info{};
    info.struct_size = sizeof(PluginInfo);
    if (!get_info(&info)) return false;

    // Check struct_size for forward compatibility
    if (info.struct_size < static_cast<int>(offsetof(PluginInfo, api_version)
                                          + sizeof(int)))
        return false;  // Too old, missing api_version field

    if (info.api_version < MIN_API_VERSION ||
        info.api_version > MAX_API_VERSION)
        return false;  // Incompatible version

    return true;
}

// Key rules for ABI safety:
// 1. NEVER pass std::string, std::vector across DLL boundary
// 2. Use C types (const char*, int, struct with C layout)
// 3. Allocate and free on the SAME side of the boundary
// 4. Use struct_size field for forward compatibility
// 5. Never remove or reorder fields in published structs
```

The five rules in the comments are worth memorizing. The reason `std::string` across the boundary is dangerous is that its internal layout (short-string optimization, allocator, size representation) varies between STL implementations and compiler versions. A `const char*` is just a pointer to null-terminated bytes - that's stable everywhere.

---

## Notes

- **C-linkage is mandatory** for factory functions: `extern "C"` prevents name mangling.
- **Never pass C++ types across DLL boundaries**: `std::string`, `std::vector`, exceptions - ABI incompatible.
- Memory rule: whoever allocates must deallocate (hence `destroy_plugin`).
- Use `-fvisibility=hidden` and explicitly export only factory symbols.
- `struct_size` field enables forward compatibility without breaking old plugins.
- Plugin shutdown must happen before `dlclose`/`FreeLibrary`.
- Consider using `std::shared_ptr` with custom deleter to pair `create`/`destroy` automatically.
