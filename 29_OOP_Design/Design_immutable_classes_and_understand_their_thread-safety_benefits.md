# Design immutable classes and understand their thread-safety benefits

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

An **immutable class** is one where all state is fixed at construction and never changes afterward. Every getter returns data, and there are no setters. This sounds restrictive, but it comes with a significant payoff: you get thread safety for free. The reason is simple - data races only happen when at least one thread is writing to shared data. If no thread ever writes after construction, there's nothing to race on. You can hand an immutable object to a hundred threads and never worry about synchronization.

| Property | Mutable Class | Immutable Class |
| --- | :---: | :---: |
| Thread-safe by default | No | Yes |
| Needs locking for shared state | Yes | No |
| Can be used as map key / set element | Maybe | Always |
| Performance for frequent updates | Better (modify in-place) | Worse (copy needed) |
| Reasoning about state | Complex | Simple |

### Design Checklist

Before reading the examples, internalize this checklist. A genuinely immutable class satisfies all of these:

```cpp
// All members declared const (or private with no mutators)
// Constructor fully initializes all state
// No setters, no mutable members (except caching)
// Return copies or const references only
// If returned by shared_ptr, clients share safely
```

---

## Self-Assessment

### Q1: Design an immutable value class

The pattern here is: all data in the constructor, const members or private-with-no-setters, and "modification" methods that return a new object instead of changing this one. The `[[nodiscard]]` on the `with_*` methods is critical - they don't modify anything, so if you call one and throw away the result you've done nothing useful. The compiler can catch that mistake for you.

```cpp
#include <string>
#include <vector>
#include <iostream>

class ImmutableConfig {
public:
    // All members set at construction
    ImmutableConfig(std::string host, int port, std::vector<std::string> features)
        : host_(std::move(host))
        , port_(port)
        , features_(std::move(features)) {}

    // Getters only - no setters
    const std::string& host() const noexcept { return host_; }
    int port() const noexcept { return port_; }
    const std::vector<std::string>& features() const noexcept { return features_; }

    // "Modifier" returns a NEW object (functional update)
    [[nodiscard]] ImmutableConfig with_port(int p) const {
        return ImmutableConfig(host_, p, features_);
    }

    [[nodiscard]] ImmutableConfig with_feature(const std::string& f) const {
        auto new_feats = features_;
        new_feats.push_back(f);
        return ImmutableConfig(host_, port_, std::move(new_feats));
    }

private:
    const std::string host_;
    const int port_;
    const std::vector<std::string> features_;
};

int main() {
    ImmutableConfig cfg("localhost", 8080, {"auth", "logging"});
    auto cfg2 = cfg.with_port(9090).with_feature("metrics");

    std::cout << cfg.port() << "\n";  // 8080 (original unchanged)
    std::cout << cfg2.port() << "\n"; // 9090
    return 0;
}
```

Notice that `cfg` is completely unaffected by the chain of `with_*` calls. Each call creates a new independent object. You can safely pass `cfg` to ten different threads without any synchronization - they're all just reading the same frozen state.

### Q2: Show thread-safety benefits of immutable objects

Here is where the benefit gets concrete. The `ConfigHolder` class holds a `shared_ptr<const ImmutableSettings>`. Readers call `get()` to obtain their own `shared_ptr` copy (which atomically increments the reference count), then use the object lock-free for as long as they need it. The writer just atomically swaps the stored pointer. Because the settings object itself is immutable, a reader who got a snapshot before the swap just keeps using the old version safely - no partial reads, no corruption.

```cpp
#include <memory>
#include <shared_mutex>
#include <thread>
#include <atomic>
#include <vector>
#include <iostream>

class ImmutableSettings {
public:
    ImmutableSettings(int timeout, int retries)
        : timeout_(timeout), retries_(retries) {}
    int timeout() const noexcept { return timeout_; }
    int retries() const noexcept { return retries_; }
private:
    const int timeout_;
    const int retries_;
};

// Thread-safe config holder using shared_ptr swap
class ConfigHolder {
    std::shared_ptr<const ImmutableSettings> config_;
    mutable std::shared_mutex mtx_;
public:
    explicit ConfigHolder(std::shared_ptr<const ImmutableSettings> cfg)
        : config_(std::move(cfg)) {}

    // Readers: no lock needed once they have the shared_ptr
    std::shared_ptr<const ImmutableSettings> get() const {
        std::shared_lock lk(mtx_);
        return config_;  // Atomic refcount increment
    }

    // Writers: swap entire config atomically
    void update(std::shared_ptr<const ImmutableSettings> new_cfg) {
        std::unique_lock lk(mtx_);
        config_ = std::move(new_cfg);
    }
};
// Readers never see partial updates!
// No locking needed after get() - object is immutable

int main() {
    auto holder = std::make_shared<ConfigHolder>(
        std::make_shared<const ImmutableSettings>(1000, 3));

    // Reader threads - safe, lock-free after get()
    auto reader = [&] {
        auto cfg = holder->get();
        std::cout << "timeout=" << cfg->timeout() << "\n";
    };

    // Writer thread - atomic swap
    auto writer = [&] {
        holder->update(std::make_shared<const ImmutableSettings>(2000, 5));
    };

    std::thread t1(reader), t2(reader), t3(writer);
    t1.join(); t2.join(); t3.join();
    return 0;
}
```

The short mutex hold in `get()` is only to safely copy the `shared_ptr` itself (pointer copy, refcount bump). After that, each thread holds its own reference and works without any lock at all. Compare this to a mutable config that would need the mutex held for the entire duration of every read - the immutable approach dramatically reduces lock contention in read-heavy code.

### Q3: Implement immutable with lazy caching using mutable

Sometimes computing a derived value is expensive, and you want to compute it once on first access rather than at construction time. This is the one legitimate use of `mutable` in an immutable class. The key is that the cached value is logically derived from the immutable data - it doesn't represent a new fact about the world, just a precomputed version of an existing one. The object is still logically immutable even though a private field can change.

```cpp
#include <string>
#include <optional>
#include <mutex>
#include <functional>

class ImmutableDocument {
public:
    ImmutableDocument(std::string title, std::string body)
        : title_(std::move(title)), body_(std::move(body)) {}

    const std::string& title() const { return title_; }
    const std::string& body() const { return body_; }

    // Lazy-computed hash - mutable cache is the ONE exception
    size_t hash() const {
        std::call_once(hash_flag_, [this] {
            std::hash<std::string> h;
            cached_hash_ = h(title_) ^ (h(body_) << 1);
        });
        return cached_hash_;
    }

    bool operator==(const ImmutableDocument& o) const {
        return hash() == o.hash() && title_ == o.title_ && body_ == o.body_;
    }

private:
    const std::string title_;
    const std::string body_;
    // Mutable cache - doesn't break logical immutability
    mutable std::once_flag hash_flag_;
    mutable size_t cached_hash_ = 0;
};
// Thread-safe: std::call_once guarantees single initialization
// Logically immutable: hash() is a pure derived value
```

`std::call_once` is the right tool here because it guarantees the lambda runs exactly once even if multiple threads call `hash()` simultaneously. After the first call, subsequent calls bypass the flag entirely. The document is still logically immutable because the hash is deterministically derived from the title and body, which never change.

---

## Notes

- Immutable objects are **inherently thread-safe** - no data races possible on read-only data.
- Use `shared_ptr<const T>` for shared ownership of immutable objects - readers hold a snapshot.
- The `mutable` keyword is acceptable for caching (lazy evaluation) - it preserves logical immutability.
- Functional-style "modifiers" (`with_xxx()`) return new objects - mark them `[[nodiscard]]`.
- Downsides: frequent updates cause many allocations; consider COW or persistent data structures for hot paths.
- `const` members prevent move semantics - consider private members with no public setters instead if moves matter.
