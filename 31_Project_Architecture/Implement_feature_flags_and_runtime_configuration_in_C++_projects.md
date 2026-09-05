# Implement feature flags and runtime configuration in C++ projects

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Feature flags** let you enable or disable functionality at runtime without pushing a new build. That sounds simple, but it unlocks some genuinely powerful workflows: A/B testing, gradual rollouts to a small percentage of users, kill switches you can flip in an emergency, and trunk-based development where half-finished features live behind a flag so they never accidentally reach users.

Feature flags are not just booleans - they can be numeric parameters, string values, or rollout percentages. The table below shows the four main archetypes you will encounter:

### Feature Flag Types

| Type | Use Case | Lifetime | Example |
| --- | --- | --- | --- |
| **Release flag** | Hide incomplete features | Short | `new_checkout_flow` |
| **Experiment flag** | A/B test | Medium | `pricing_algorithm_v2` |
| **Ops flag** | Kill switch | Long | `enable_caching` |
| **Permission flag** | User access control | Permanent | `premium_export` |

The lifetime column matters more than it looks. Short-lived flags should be removed after the rollout completes - leaving them around forever is how you get a codebase full of dead branches nobody understands. We will come back to this in Q3 with a cleanup tracker.

---

## Self-Assessment

### Q1: Implement a feature flag system

The core of any flag system is a thread-safe store that maps names to values. We use `std::variant` so a single store can hold booleans, integers, doubles, or strings. The struct also carries metadata - owner and description - so you can track who is responsible for each flag.

**Answer:**

```cpp
#include <string>
#include <unordered_map>
#include <shared_mutex>
#include <functional>
#include <optional>
#include <variant>
#include <chrono>

// === Feature flag value types ===
using FlagValue = std::variant<bool, int, double, std::string>;

// === Feature flag with metadata ===
struct FeatureFlag {
    FlagValue value;
    std::string description;
    std::string owner;  // Team responsible
    bool is_stale = false;  // Flag for cleanup
};

// === Thread-safe feature flag store ===
class FeatureFlags {
public:
    void set(const std::string& name, FlagValue value) {
        std::unique_lock lock(mutex_);
        flags_[name].value = value;
    }

    // Register flag with metadata at startup
    void define(const std::string& name, FlagValue default_value,
                const std::string& desc, const std::string& owner) {
        std::unique_lock lock(mutex_);
        flags_[name] = {default_value, desc, owner};
    }

    template<typename T>
    T get(const std::string& name, T fallback = T{}) const {
        std::shared_lock lock(mutex_);
        auto it = flags_.find(name);
        if (it == flags_.end()) return fallback;
        if (auto* val = std::get_if<T>(&it->second.value))
            return *val;
        return fallback;
    }

    bool is_enabled(const std::string& name) const {
        return get<bool>(name, false);
    }

    // Bulk update (e.g., from config server)
    void update_from(const std::unordered_map<std::string, FlagValue>& updates) {
        std::unique_lock lock(mutex_);
        for (const auto& [name, value] : updates) {
            auto it = flags_.find(name);
            if (it != flags_.end())
                it->second.value = value;
        }
    }

    // List all flags (for admin dashboard)
    std::vector<std::pair<std::string, FeatureFlag>> list_all() const {
        std::shared_lock lock(mutex_);
        return {flags_.begin(), flags_.end()};
    }

private:
    std::unordered_map<std::string, FeatureFlag> flags_;
    mutable std::shared_mutex mutex_;
};

// === Global instance ===
FeatureFlags& feature_flags() {
    static FeatureFlags instance;
    return instance;
}
```

Notice that `std::shared_mutex` is used here instead of a plain `std::mutex`. That choice matters: flag checks happen constantly on every hot code path, but flag updates are rare. A shared mutex lets many threads read simultaneously and only blocks everyone for the infrequent write. That is the right trade-off when reads vastly outnumber writes.

### Q2: Use feature flags in application code cleanly

The registration step at startup is important. Defining flags with defaults and metadata means the system always has a sane value even before a remote config server responds - and you get the admin dashboard for free.

**Answer:**

```cpp
// === Registration at startup ===
void register_flags() {
    auto& ff = feature_flags();
    ff.define("new_checkout", false,
        "V2 checkout flow", "team-checkout");
    ff.define("caching_enabled", true,
        "Response caching", "team-platform");
    ff.define("max_retries", 3,
        "Max API retry count", "team-networking");
    ff.define("discount_pct", 10.0,
        "Global discount percentage", "team-pricing");
}

// === Usage in business logic ===
class CheckoutService {
public:
    void process(Order& order) {
        auto& ff = feature_flags();

        // Boolean flag: feature gate
        if (ff.is_enabled("new_checkout")) {
            process_v2(order);
        } else {
            process_v1(order);
        }

        // Numeric flag: tunable parameter
        double discount = ff.get<double>("discount_pct", 0.0);
        order.apply_discount(discount / 100.0);
    }

private:
    void process_v1(Order& order) { /* old path */ }
    void process_v2(Order& order) { /* new path */ }
};

class ApiClient {
public:
    Response send_with_retry(const Request& req) {
        int max_retries = feature_flags().get<int>("max_retries", 3);
        for (int attempt = 0; attempt <= max_retries; ++attempt) {
            auto resp = send(req);
            if (resp.ok()) return resp;
        }
        return Response::error("Max retries exceeded");
    }
};
```

The pattern here is intentional: business logic never hard-codes limits or paths directly. If you need to change retry behavior in a production incident, you update the flag value via your config server and the change takes effect without a rebuild. The `get<int>` call with a fallback also means the feature keeps working even if the flag name is mistyped or missing.

### Q3: Implement percentage-based rollout and cleanup tracking

Percentage rollouts let you gradually expose a feature to more and more users. The deterministic hashing approach is the important one to understand: by hashing the flag name with the user ID, the same user always ends up in the same bucket. That means a user does not randomly switch between the old and new experience on every page load - they get a consistent treatment throughout the experiment.

**Answer:**

```cpp
#include <random>
#include <chrono>

// === Percentage-based rollout ===
class RolloutFlag {
public:
    RolloutFlag(const std::string& name, int percentage)
        : name_(name), percentage_(percentage) {}

    // Deterministic: same user always gets same result
    bool is_enabled_for(const std::string& user_id) const {
        auto hash = std::hash<std::string>{}(name_ + user_id);
        return (hash % 100) < static_cast<size_t>(percentage_);
    }

    // Random: used for load balancing, not user consistency
    bool is_enabled_random() const {
        static thread_local std::mt19937 rng(std::random_device{}());
        std::uniform_int_distribution<int> dist(0, 99);
        return dist(rng) < percentage_;
    }

    void set_percentage(int pct) {
        percentage_ = std::clamp(pct, 0, 100);
    }

private:
    std::string name_;
    int percentage_;
};

// === Flag cleanup tracker ===
class FlagCleanupTracker {
public:
    struct StaleFlag {
        std::string name;
        std::string owner;
        std::chrono::system_clock::time_point created;
        int days_old;
    };

    void register_flag(const std::string& name, const std::string& owner) {
        flags_[name] = {name, owner, std::chrono::system_clock::now(), 0};
    }

    std::vector<StaleFlag> get_stale_flags(int max_age_days) const {
        auto now = std::chrono::system_clock::now();
        std::vector<StaleFlag> stale;
        for (auto& [name, flag] : flags_) {
            auto age = std::chrono::duration_cast<std::chrono::hours>(
                now - flag.created).count() / 24;
            if (age > max_age_days) {
                auto f = flag;
                f.days_old = static_cast<int>(age);
                stale.push_back(f);
            }
        }
        return stale;
    }

    void report_stale() const {
        for (auto& f : get_stale_flags(30)) {
            std::cerr << "STALE FLAG: " << f.name
                      << " (owner: " << f.owner
                      << ", age: " << f.days_old << " days)\n";
        }
    }

private:
    std::unordered_map<std::string, StaleFlag> flags_;
};
```

The `FlagCleanupTracker` is easy to overlook, but in practice it is one of the most valuable pieces. A flag that was supposed to last two weeks can silently live in a codebase for two years if nobody tracks it. Running `report_stale()` in a CI job or on server startup creates a natural pressure to clean up dead flags before they accumulate.

---

## Notes

- Feature flags are technical debt - track them and clean up after rollout completes.
- Use `std::shared_mutex` for read-heavy workloads (many flag checks, rare updates).
- Percentage rollout with deterministic hashing ensures the same user always sees the same behavior.
- Keep flag checks cheap: `is_enabled()` should be O(1) with no I/O.
- For remote flag updates, poll a config server periodically (not on every check).
- Compile-time flags (`if constexpr`) are preferred when you don't need runtime switching.
- Log which path was taken for debugging: `LOG_INFO("Using checkout v" + std::to_string(version))`.
