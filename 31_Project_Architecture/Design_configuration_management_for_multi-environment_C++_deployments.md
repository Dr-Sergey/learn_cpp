# Design configuration management for multi-environment C++ deployments

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Configuration management** is how your application adapts to different environments - development on your laptop, a shared staging server, and the live production cluster. Each environment needs different database hosts, log levels, worker counts, and so on. Hardcoding any of those values is a maintenance problem waiting to happen.

A good configuration system does three things: it loads settings from multiple sources in a predictable priority order, it validates values at startup so you fail fast with a clear error rather than mysteriously crashing at runtime, and it exposes a type-safe API so your code never works with raw strings past the configuration layer. The structured-config approach goes one step further by mapping raw key-value pairs into typed C++ structs, which gives you IDE autocomplete and eliminates whole categories of typos.

### Configuration Sources (Priority Order)

Sources are applied in order from lowest to highest priority, so later sources always override earlier ones. This table summarizes the standard industry convention.

| Source | Priority | Use Case |
| --- | --- | --- |
| Command-line args | Highest | Per-run overrides |
| Environment variables | High | Container/CI settings |
| Local config file | Medium | Developer overrides |
| Environment config file | Low | Environment defaults |
| Compiled defaults | Lowest | Fallback values |

---

## Self-Assessment

### Q1: Implement a type-safe configuration system

**Answer:**

The `Config` class below follows the layered override pattern precisely. You call `load_defaults()` first to establish safe fallbacks, then load files and environment variables on top. Each later call overwrites any key it finds. The type-safe getters (`get_int`, `get_bool`, `get_string`) parse values at the point of access, and `validate()` enforces business rules before the application starts doing any real work.

```cpp
#include <string>
#include <unordered_map>
#include <optional>
#include <fstream>
#include <sstream>
#include <stdexcept>
#include <vector>
#include <cstdlib>
#include <algorithm>

// === Type-safe configuration with defaults and validation ===
class Config {
public:
    // Load from multiple sources (later sources override earlier)
    void load_defaults() {
        set("server.port", "8080");
        set("server.host", "0.0.0.0");
        set("server.workers", "4");
        set("db.connection_pool", "10");
        set("db.timeout_ms", "5000");
        set("log.level", "info");
        set("log.file", "/var/log/app.log");
    }

    void load_file(const std::string& path) {
        std::ifstream f(path);
        if (!f.is_open()) return;  // Optional file
        std::string line;
        while (std::getline(f, line)) {
            if (line.empty() || line[0] == '#') continue;
            auto eq = line.find('=');
            if (eq == std::string::npos) continue;
            set(line.substr(0, eq), line.substr(eq + 1));
        }
    }

    void load_env(const std::string& prefix = "APP_") {
        // Environment variables override file settings
        for (const auto& [key, _] : values_) {
            auto env_key = prefix + key;
            // Transform: server.port -> APP_SERVER_PORT
            std::transform(env_key.begin(), env_key.end(),
                          env_key.begin(), [](char c) {
                return c == '.' ? '_' : ::toupper(c);
            });
            if (const char* val = std::getenv(env_key.c_str()))
                set(key, val);
        }
    }

    void load_args(int argc, char* argv[]) {
        for (int i = 1; i < argc; ++i) {
            std::string arg = argv[i];
            if (arg.starts_with("--")) {
                auto eq = arg.find('=');
                if (eq != std::string::npos) {
                    auto key = arg.substr(2, eq - 2);
                    std::replace(key.begin(), key.end(), '-', '.');
                    set(key, arg.substr(eq + 1));
                }
            }
        }
    }

    // Type-safe getters
    int get_int(const std::string& key) const {
        return std::stoi(get(key));
    }
    bool get_bool(const std::string& key) const {
        auto v = get(key);
        return v == "true" || v == "1" || v == "yes";
    }
    std::string get_string(const std::string& key) const {
        return get(key);
    }
    std::optional<std::string> get_optional(const std::string& key) const {
        auto it = values_.find(key);
        return it != values_.end()
            ? std::optional{it->second} : std::nullopt;
    }

    // Validation
    void validate() const {
        require("server.port");
        require("db.connection_pool");
        auto port = get_int("server.port");
        if (port < 1 || port > 65535)
            throw std::runtime_error("Invalid port: " + std::to_string(port));
    }

private:
    void set(const std::string& key, const std::string& value) {
        values_[key] = value;
    }
    std::string get(const std::string& key) const {
        auto it = values_.find(key);
        if (it == values_.end())
            throw std::runtime_error("Config key missing: " + key);
        return it->second;
    }
    void require(const std::string& key) const {
        if (values_.find(key) == values_.end())
            throw std::runtime_error("Required config missing: " + key);
    }
    std::unordered_map<std::string, std::string> values_;
};

// === Usage ===
int main(int argc, char* argv[]) {
    Config config;
    config.load_defaults();
    config.load_file("config/base.conf");
    config.load_file("config/" + get_environment() + ".conf");
    config.load_env();
    config.load_args(argc, argv);
    config.validate();

    auto port = config.get_int("server.port");
    auto workers = config.get_int("server.workers");
}
```

The `load_file` function returns silently if the file is missing - that is intentional. Not every environment will have every config file present, and a missing file should just mean "no overrides from this source". A missing required key is a different matter, and `validate()` handles that explicitly.

### Q2: Environment-specific configuration files

**Answer:**

Here are the four configuration files that match the environment names returned by `get_environment()`. Each one only specifies what is different from the base defaults. The environment is chosen by the `APP_ENV` environment variable.

```cpp
# === config/base.conf (shared defaults) ===
server.port=8080
server.workers=4
db.timeout_ms=5000
log.level=info

# === config/development.conf ===
server.workers=1
log.level=debug
log.file=stdout
db.host=localhost
db.name=app_dev

# === config/staging.conf ===
server.workers=4
log.level=info
db.host=staging-db.internal
db.name=app_staging

# === config/production.conf ===
server.workers=16
log.level=warn
db.host=prod-db.internal
db.name=app_prod
db.connection_pool=50
```

```cpp
// === Environment detection ===
std::string get_environment() {
    if (const char* env = std::getenv("APP_ENV"))
        return env;
    return "development";  // Default
}

// Override chain for production:
// defaults -> base.conf -> production.conf -> APP_SERVER_PORT=9090 -> --server.port=9091
// Final: port=9091 (CLI wins)
```

The comment showing the override chain for production is worth reading carefully. You can trace exactly which source wins for any given key, because the priority order is fixed and deterministic.

### Q3: Structured config with schema validation

**Answer:**

Working directly with string key-value pairs throughout your codebase is fragile - you can misspell a key, get the wrong type, or check for a key that was renamed. Structured config types solve this by mapping the raw config into typed C++ structs as early as possible. From that point on, the rest of the application works with typed fields, not string lookups.

```cpp
// === Structured config with compile-time schema ===
struct ServerConfig {
    std::string host = "0.0.0.0";
    int port = 8080;
    int workers = 4;
    int max_connections = 1024;
};

struct DatabaseConfig {
    std::string host = "localhost";
    int port = 5432;
    std::string name = "app";
    int pool_size = 10;
    int timeout_ms = 5000;
};

struct LogConfig {
    std::string level = "info";
    std::string file = "/var/log/app.log";
    bool json_format = false;
};

struct AppConfig {
    ServerConfig server;
    DatabaseConfig database;
    LogConfig log;
    std::string environment = "development";

    // Load and validate
    static AppConfig load(const Config& raw) {
        AppConfig cfg;
        cfg.environment = raw.get_optional("environment")
            .value_or("development");
        cfg.server.host = raw.get_optional("server.host")
            .value_or(cfg.server.host);
        cfg.server.port = raw.get_optional("server.port")
            .transform([](const std::string& s) { return std::stoi(s); })
            .value_or(cfg.server.port);
        cfg.server.workers = raw.get_optional("server.workers")
            .transform([](const std::string& s) { return std::stoi(s); })
            .value_or(cfg.server.workers);
        // ... similarly for other fields ...

        cfg.validate();
        return cfg;
    }

    void validate() const {
        if (server.port < 1 || server.port > 65535)
            throw std::invalid_argument("Invalid server port");
        if (server.workers < 1 || server.workers > 256)
            throw std::invalid_argument("Invalid worker count");
        if (database.pool_size < 1)
            throw std::invalid_argument("Pool size must be positive");
    }
};
```

The use of `.transform()` and `.value_or()` on `std::optional` is the modern C++23 way to express "parse this string if it exists, otherwise keep the default". The whole `load()` function is doing one job: translate from the stringly-typed raw config into the type-safe structured form, with no string lookups leaking past this boundary.

---

## Notes

- The priority order (CLI > env > local file > env file > defaults) is the industry standard for multi-environment deployments.
- Environment variables work particularly well in containers, where Docker and Kubernetes inject them without touching the image.
- Never store secrets in config files - use environment variables or a dedicated secrets manager instead.
- Validate config at startup and fail fast with a clear error message rather than crashing unexpectedly at runtime.
- Structured config types like `AppConfig` give you IDE autocomplete and make typos in key names impossible.
- For complex configurations, JSON, TOML, or YAML are worth considering - use nlohmann/json, toml++, or a YAML library to parse them.
- For embedded targets, compile-time `constexpr` configuration is usually preferable because it has zero file I/O overhead.
