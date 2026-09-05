# Leverage AI for refactoring legacy C++ code to modern standards

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

If you've inherited a pre-C++11 codebase, you already know the frustration: raw pointers everywhere, `NULL` instead of `nullptr`, `typedef` soup, and threading code written against platform-specific APIs. AI assistants are powerful tools for **modernizing legacy C++ code**. They can identify pre-C++11 patterns and suggest modern replacements, explain what old code does, and produce refactored versions that are safer, more readable, and often more performant. The key is providing enough context about constraints - compiler version, platform, coding standards - so the AI doesn't suggest features you can't use yet.

### Common Legacy -> Modern Transformations

The table below groups transformations by how much risk they carry. Low-risk changes are essentially mechanical and can be done in bulk. High-risk changes affect semantics in ways that require careful thought before touching them.

| Legacy Pattern | Modern Replacement | Risk Level |
| --- | --- | --- |
| Raw `new`/`delete` | `std::unique_ptr`, `std::shared_ptr` | Medium |
| C-style casts | `static_cast`, `dynamic_cast` | Low |
| `NULL` / `0` as null pointer | `nullptr` | Low |
| `typedef` | `using` alias | Low |
| Manual loops | Range-for, algorithms, ranges | Medium |
| `#define` constants | `constexpr`, `inline constexpr` | Low |
| Function pointers | `std::function`, lambdas | Medium |
| Output params | Return by value, structured bindings | Medium |
| `boost::optional` | `std::optional` | Low |
| Exception specs (`throw()`) | `noexcept` | Low |
| `auto_ptr` | `unique_ptr` (different semantics!) | High |

---

## Self-Assessment

### Q1: Modernize raw pointer ownership patterns

**Answer:**

When asking AI to refactor ownership code, it's important to tell it exactly what rules to follow. The prompt below is specific about the constraints, which keeps the AI from making creative choices you didn't ask for.

```cpp
=== PROMPT ===

"Refactor this legacy C++ code to use modern C++17 idioms.
Rules:

- Replace raw new/delete with smart pointers
- Use RAII for all resources
- Keep the same public API behavior
- Mark ownership transfers clearly
- Add noexcept where appropriate

Do NOT change the class's observable behavior."
```

Here's the legacy code and what the AI produces. Pay attention to how the refactored version eliminates the manual destructor entirely - all cleanup happens automatically through `std::vector<std::unique_ptr<Resource>>`.

```cpp
// === LEGACY CODE ===
class ResourceManager {
    struct Resource {
        char* data;
        int size;
        Resource* next;
    };
    Resource* head_;
    int count_;

public:
    ResourceManager() : head_(NULL), count_(0) {}

    ~ResourceManager() {
        Resource* curr = head_;
        while (curr) {
            Resource* next = curr->next;
            delete[] curr->data;
            delete curr;
            curr = next;
        }
    }

    void add(const char* data, int size) {
        Resource* r = new Resource;
        r->data = new char[size];
        memcpy(r->data, data, size);
        r->size = size;
        r->next = head_;
        head_ = r;
        count_++;
    }

    Resource* find(const char* prefix) {
        for (Resource* r = head_; r != NULL; r = r->next) {
            if (strncmp(r->data, prefix, strlen(prefix)) == 0)
                return r;
        }
        return NULL;
    }

    // No copy constructor or assignment operator defined!
};


// === AI-REFACTORED MODERN C++17 ===
class ResourceManager {
    struct Resource {
        std::vector<char> data;
        // No 'next' pointer - use container instead
    };
    std::vector<std::unique_ptr<Resource>> resources_;

public:
    ResourceManager() = default;
    ~ResourceManager() = default;

    // Explicitly non-copyable (was implicitly broken before)
    ResourceManager(const ResourceManager&) = delete;
    ResourceManager& operator=(const ResourceManager&) = delete;

    // Movable
    ResourceManager(ResourceManager&&) noexcept = default;
    ResourceManager& operator=(ResourceManager&&) noexcept = default;

    void add(std::span<const char> data) {
        auto r = std::make_unique<Resource>();
        r->data.assign(data.begin(), data.end());
        resources_.push_back(std::move(r));
    }

    // Return non-owning pointer (or nullptr)
    const Resource* find(std::string_view prefix) const {
        auto it = std::ranges::find_if(resources_,
            [prefix](const auto& r) {
                return std::string_view(r->data.data(), r->data.size())
                    .starts_with(prefix);
            });
        return it != resources_.end() ? it->get() : nullptr;
    }

    size_t count() const noexcept { return resources_.size(); }
};
```

Notice that the AI also fixed a latent bug - the original had no copy constructor or copy assignment operator defined, which meant copying a `ResourceManager` would perform a shallow pointer copy and cause a double-free. The refactored version explicitly deletes copy operations and provides correct move semantics.

### Q2: Modernize callback and threading patterns

**Answer:**

Threading code is where legacy C++ gets especially painful. `pthread` APIs require manual lock/unlock pairing (forgetting one is a deadlock or data race waiting to happen), and function pointers with `void*` context arguments are the C way of faking closures. Here's how to prompt for this modernization.

```cpp
=== PROMPT ===

"Modernize this callback-based async code to use
std::thread, std::mutex, std::function, and RAII.
The legacy code uses pthreads and function pointers."
```

Look at the legacy code's `while(1)` busy-wait with `usleep` - that's a classic pattern that burns CPU and introduces latency. The modern version uses a condition variable to sleep efficiently until work arrives.

```cpp
// === LEGACY CODE ===
typedef void (*callback_fn)(void* ctx, int result);

struct WorkItem {
    callback_fn cb;
    void* ctx;
    int input;
};

pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;
std::vector<WorkItem> g_queue;  // Unprotected global

void* worker_thread(void* arg) {
    while (1) {
        pthread_mutex_lock(&g_mutex);
        if (g_queue.empty()) {
            pthread_mutex_unlock(&g_mutex);
            usleep(1000);  // Busy-wait with sleep
            continue;
        }
        WorkItem item = g_queue.back();
        g_queue.pop_back();
        pthread_mutex_unlock(&g_mutex);

        int result = item.input * 2;  // "Processing"
        item.cb(item.ctx, result);
    }
    return NULL;
}


// === AI-REFACTORED MODERN C++17 ===
class WorkQueue {
    struct WorkItem {
        std::function<void(int)> callback;
        int input;
    };

    std::queue<WorkItem> queue_;
    std::mutex mutex_;
    std::condition_variable cv_;
    std::jthread worker_;  // C++20 jthread auto-joins
    std::atomic<bool> stopping_{false};

    void worker_loop(std::stop_token stoken) {
        while (!stoken.stop_requested()) {
            std::unique_lock lock(mutex_);
            cv_.wait(lock, [&] {
                return !queue_.empty() || stoken.stop_requested();
            });
            if (stoken.stop_requested()) break;

            auto item = std::move(queue_.front());
            queue_.pop();
            lock.unlock();

            int result = item.input * 2;
            item.callback(result);
        }
    }

public:
    WorkQueue()
        : worker_([this](std::stop_token st) { worker_loop(st); }) {}

    // No explicit destructor needed - jthread auto-joins

    void submit(int input, std::function<void(int)> callback) {
        {
            std::lock_guard lock(mutex_);
            queue_.push({std::move(callback), input});
        }
        cv_.notify_one();
    }
};

// Usage:
WorkQueue queue;
queue.submit(42, [](int result) {
    std::cout << "Result: " << result << '\n';
});
```

The `std::jthread` (C++20) is the key quality-of-life improvement here - it automatically requests a stop and joins on destruction, so there's no way to forget to clean up the thread.

### Q3: Systematic legacy codebase modernization strategy

**Answer:**

For a large codebase, trying to modernize everything at once is a recipe for breaking things in hard-to-trace ways. The right approach is a phased plan that moves from risk-free mechanical changes to semantically tricky ones. Here's how to ask AI for that plan.

```cpp
=== PROMPT ===

"I have a 500KLOC C++03 codebase. I want to modernize
it incrementally to C++17 without breaking anything.
Create a prioritized modernization plan with specific
AI prompts for each step."

=== AI STRATEGY ===

"Phase 1: Low-risk, high-value (automated with AI)

- Replace NULL -> nullptr
- Replace typedef -> using
- Add override to virtual functions
- Replace #define constants -> constexpr

  Prompt: 'Find all NULL usages in [file] and replace
  with nullptr. Show the diff.'

Phase 2: Medium risk (AI + review)

- Replace raw new/delete -> smart pointers
- Replace output params -> return by value
- Replace manual loops -> range-for

  Prompt: 'Analyze ownership of [pointer] in [class].
  Is it unique, shared, or non-owning? Suggest the
  correct smart pointer replacement.'

Phase 3: Higher risk (AI + thorough testing)

- Replace boost:: -> std:: equivalents
- Modernize threading (pthreads -> std::thread)
- Add move semantics to key types

  Prompt: 'This class has expensive copy operations.
  Add move constructor and move assignment operator.
  Identify which members can be moved vs copied.'

Phase 4: Architecture improvements

- Replace inheritance hierarchies -> std::variant
- Replace virtual dispatch -> CRTP where appropriate
- Add concepts to template interfaces"
```

A concrete example of what Phase 1 looks like in practice - the AI can also flag potential bugs it discovers during the mechanical work:

```cpp
// Phase 1 example: AI-assisted override audit
// Prompt: "Add 'override' to all virtual function
// overrides in this class. Flag any that don't
// actually override a base class method."

// BEFORE:
class DerivedHandler : public BaseHandler {
    virtual void on_connect(int fd);     // Override? Unknown
    virtual void on_data(int fd, char* buf, int len);
    virtual void on_close(int fd);
    virtual void on_timeout();  // Might NOT override anything
};

// AI OUTPUT:
class DerivedHandler : public BaseHandler {
    void on_connect(int fd) override;
    void on_data(int fd, char* buf, int len) override;
    void on_close(int fd) override;
    // WARNING: on_timeout() does NOT override any base method.
    // Base has: on_idle_timeout(). Did you mean that?
    virtual void on_timeout();  // NOT an override - possible bug!
};
```

That last comment is the AI earning its keep - what looked like a routine modernization task surfaced a genuine bug where a method name didn't match the base class.

---

## Notes

- **Start with compiler warnings**: `-Wsuggest-override`, `-Wold-style-cast` identify modernization targets before you even open the files.
- Ask AI to **explain legacy code** before refactoring - make sure you understand the original intent, because the AI might not preserve subtle side effects.
- **`auto_ptr` to `unique_ptr` is NOT a drop-in replacement** - `auto_ptr` copies transfer ownership, so copy-based containers of `auto_ptr` behave very differently from containers of `unique_ptr`. This is the "high risk" entry in the table, and it deserves thorough testing.
- Always **run existing tests** after each refactoring step - AI can introduce subtle behavior changes even when trying to be conservative.
- AI is excellent at **bulk transformations** (`NULL` -> `nullptr`, `typedef` -> `using`) across files, where it's essentially doing pattern matching at scale.
- For large codebases, use AI to **generate clang-tidy rules** for automated modernization so the changes can be applied mechanically and reviewed in a diff.
- **Don't modernize everything at once** - incremental changes are safer, more reviewable, and much easier to revert if something breaks.
