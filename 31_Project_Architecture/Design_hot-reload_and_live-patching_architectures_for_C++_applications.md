# Design hot-reload and live-patching architectures for C++ applications

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Hot-reload** updates application logic at runtime without restarting the process. In C++ this is a genuinely hard problem because the language does not have built-in support for it - but there are several practical approaches. The most powerful is reloading shared libraries (`.so` on Linux, `.dll` on Windows), which lets you recompile a module and have the running process load the new version. Simpler cases - config files, shader files, Lua scripts - use file-watching and can reload in milliseconds.

Hot-reload is essential in game development (change gameplay logic without restarting a level), audio plugins (swap DSP code live), and long-running servers where downtime is expensive. The fundamental rule that makes it work is: **state lives in the host process, logic lives in the reloadable module**. When you reload the module, you get new code, but the existing state - player position, score, session data - is preserved because it was never inside the module in the first place.

### Hot-Reload Approaches

| Approach | Scope | Risk | Latency |
| --- | --- | --- | --- |
| **Shared library reload** | Code logic | Medium (ABI, state) | ~100ms |
| **Script reload (Lua/Python)** | Scripted behavior | Low | ~10ms |
| **Config/data reload** | Parameters, rules | Very low | ~1ms |
| **Live patching (ksplice-style)** | Binary patches | High | ~1ms |

---

## Self-Assessment

### Q1: Implement shared library hot-reload

**Answer:**

The `HotReloader` manages a single loaded module. When a file change is detected, it calls `on_unload()` on the old instance, destroys it, unloads the library, then loads the new `.so`, creates a new instance, and calls `on_load()` with a pointer to the host's state. The host's application state is untouched through all of this.

```cpp
#include <dlfcn.h>  // Linux; use LoadLibrary on Windows
#include <string>
#include <memory>
#include <iostream>
#include <sys/inotify.h>

// === Hot-reloadable module interface ===
struct IModule {
    virtual ~IModule() = default;
    virtual const char* name() const = 0;
    virtual int version() const = 0;
    virtual void on_load(void* app_state) = 0;
    virtual void on_unload() = 0;
    virtual void update(float dt) = 0;
    virtual void render() = 0;
};

extern "C" {
    using CreateModuleFn = IModule*(*)();
    using DestroyModuleFn = void(*)(IModule*);
}

// === Hot-reload manager ===
class HotReloader {
public:
    struct LoadedModule {
        void* handle = nullptr;
        IModule* instance = nullptr;
        DestroyModuleFn destroy = nullptr;
        std::string path;
        time_t last_modified = 0;
    };

    bool load(const std::string& path, void* app_state) {
        // Unload previous version
        if (module_.handle) {
            unload();
        }

        // Load new shared library
        void* handle = dlopen(path.c_str(), RTLD_NOW);
        if (!handle) {
            std::cerr << "Load failed: " << dlerror() << "\n";
            return false;
        }

        auto create = reinterpret_cast<CreateModuleFn>(
            dlsym(handle, "create_module"));
        auto destroy = reinterpret_cast<DestroyModuleFn>(
            dlsym(handle, "destroy_module"));

        if (!create || !destroy) {
            dlclose(handle);
            return false;
        }

        IModule* instance = create();
        instance->on_load(app_state);

        module_ = {handle, instance, destroy, path, file_mtime(path)};
        std::cout << "Loaded: " << instance->name()
                  << " v" << instance->version() << "\n";
        return true;
    }

    void unload() {
        if (module_.instance) {
            module_.instance->on_unload();
            module_.destroy(module_.instance);
        }
        if (module_.handle) {
            dlclose(module_.handle);
        }
        module_ = {};
    }

    // Check if file changed and reload
    bool check_and_reload(void* app_state) {
        if (module_.path.empty()) return false;
        auto mtime = file_mtime(module_.path);
        if (mtime > module_.last_modified) {
            std::cout << "Change detected, reloading...\n";
            return load(module_.path, app_state);
        }
        return false;
    }

    IModule* module() { return module_.instance; }

private:
    LoadedModule module_;

    time_t file_mtime(const std::string& path) {
        struct stat st;
        if (stat(path.c_str(), &st) == 0)
            return st.st_mtime;
        return 0;
    }
};

// === Main loop with hot-reload ===
void game_loop(HotReloader& reloader, GameState& state) {
    while (state.running) {
        // Check for changed .so every frame (cheap stat() call)
        reloader.check_and_reload(&state);

        if (auto* mod = reloader.module()) {
            mod->update(state.delta_time);
            mod->render();
        }
    }
}
```

The `create_module` and `destroy_module` functions must be declared `extern "C"` in the module's implementation. Without that, the C++ name mangler changes their names in the `.so` file and `dlsym` cannot find them. This is one of those things that fails silently at `dlsym` time and is confusing the first time you encounter it.

### Q2: Preserve state across reloads

**Answer:**

This is the core design constraint for hot-reload. The reason state cannot live in the module is that when you `dlclose` the library, all memory belonging to it is potentially invalidated. Any objects the module allocated, any statics it owned, are gone. If the host had pointers into module memory, they are now dangling.

The solution is to put all mutable state in the host and pass a pointer to it into the module on load.

```cpp
// === State preservation: host owns the state, module only has logic ===

// State lives in the HOST (never in the .so)
struct GameState {
    float player_x, player_y;
    int score;
    float delta_time;
    bool running;
    std::vector<Entity> entities;
    // All mutable state here - survives reload
};

// Module receives state pointer on load
class GameModule : public IModule {
public:
    void on_load(void* app_state) override {
        state_ = static_cast<GameState*>(app_state);
        std::cout << "Module loaded, state preserved. "
                  << "Score: " << state_->score << "\n";
    }

    void on_unload() override {
        // Flush any module-local caches
        state_ = nullptr;
    }

    void update(float dt) override {
        // Logic changes take effect immediately on reload
        state_->player_x += 5.0f * dt;  // Was 3.0f before reload
        if (/*collision*/ false)
            state_->score += 100;  // Changed scoring
    }

    void render() override {
        // Rendering logic also hot-reloadable
    }

private:
    GameState* state_ = nullptr;
};

// Key rules:
// 1. State lives in the host, NOT in the shared library
// 2. Module gets state pointer via on_load()
// 3. on_unload() must release any module-local resources
// 4. Never store pointers to module functions in state
// 5. State struct changes require full restart (ABI change)
```

Rule 5 is the hard limit of hot-reload: you can change function bodies freely, but if you change the layout of `GameState` itself, old host code and new module code will have incompatible views of what the struct looks like. That requires a full restart. This is the ABI boundary, and it is the reason hot-reload works well for logic changes but not for data structure changes.

### Q3: Implement config/data hot-reload with file watching

**Answer:**

Config hot-reload is the simpler and lower-risk version of hot-reload. No shared library loading, no ABI concerns. You watch a file for changes, re-parse it when it changes, and atomically swap in the new config so readers always see a consistent snapshot.

```cpp
#include <filesystem>
#include <fstream>
#include <atomic>
#include <thread>

// === File watcher (cross-platform) ===
class FileWatcher {
public:
    using Callback = std::function<void(const std::string&)>;

    void watch(const std::string& path, Callback callback) {
        watches_.push_back({path, std::move(callback),
                           std::filesystem::last_write_time(path)});
    }

    void poll() {
        for (auto& w : watches_) {
            auto current = std::filesystem::last_write_time(w.path);
            if (current != w.last_modified) {
                w.last_modified = current;
                w.callback(w.path);
            }
        }
    }

    // Background polling thread
    void start_background(std::chrono::milliseconds interval) {
        thread_ = std::jthread([this, interval](std::stop_token st) {
            while (!st.stop_requested()) {
                poll();
                std::this_thread::sleep_for(interval);
            }
        });
    }

private:
    struct Watch {
        std::string path;
        Callback callback;
        std::filesystem::file_time_type last_modified;
    };
    std::vector<Watch> watches_;
    std::jthread thread_;
};

// === Hot-reloadable configuration ===
class HotConfig {
public:
    HotConfig(const std::string& path) : path_(path) {
        reload();
        watcher_.watch(path, [this](const std::string&) {
            reload();
        });
        watcher_.start_background(std::chrono::milliseconds(500));
    }

    // Thread-safe read via atomic pointer swap
    std::shared_ptr<const Config> get() const {
        return std::atomic_load(&config_);
    }

private:
    void reload() {
        auto new_config = std::make_shared<Config>();
        new_config->load_file(path_);
        std::atomic_store(&config_, new_config);
        std::cout << "Config reloaded: " << path_ << "\n";
    }

    std::string path_;
    std::shared_ptr<Config> config_;
    FileWatcher watcher_;
};

// Usage:
// HotConfig config("settings.conf");
// auto cfg = config.get();  // Always returns latest
// int port = cfg->get_int("server.port");
```

The `std::atomic_load` and `std::atomic_store` with `shared_ptr` is the key thread-safety mechanism here. Any thread that calls `get()` gets a `shared_ptr` snapshot - the config object it points to will not be destroyed while that snapshot is alive, even if `reload()` runs concurrently and replaces the pointer. This is a standard lock-free config swap pattern.

---

## Notes

- State must live in the host process, never inside the reloadable module. Module replacement destroys everything the module owned.
- On Linux, you can `dlopen` the same path after rebuilding and the kernel loads the new file automatically.
- On Windows, the OS locks `.dll` files that are loaded, so you must copy the DLL to a temporary path before loading it.
- **File watching** is the simplest form of hot-reload and covers config files, Lua scripts, and shader files with very low risk.
- `std::atomic_store` and `std::atomic_load` with `shared_ptr` enable lock-free config swaps between the file-watcher thread and reader threads.
- For game development, hot-reload can save hours of iteration time per day - changing game logic without restarting a level is a significant productivity improvement.
- Production servers can use config hot-reload to adjust log levels, feature flags, and rate limits without any service downtime.
