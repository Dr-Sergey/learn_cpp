# Generate boilerplate code - serialization, operators, constructors with AI

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

C++ makes you write a lot of code that follows mechanical patterns and serves a supporting role: special member functions, comparison operators, serialization, stream operators, hash functions, swap specializations. It is correct and necessary, but it is not where the interesting design decisions live, and getting it wrong has real consequences (a move constructor that forgets to null the source pointer, a copy assignment that does not handle self-assignment).

AI is well-suited to generating this category of code because the patterns are consistent and the rules are well-defined. The key to getting good output is to specify exactly what you need and, if you have a coding style, to show the AI an example from your codebase rather than describing the style in words. The table below tells you where to trust the output and where to double-check carefully.

### Boilerplate Types and AI Generation Quality

| Boilerplate Type | AI Quality | Risk to Watch |
| --- | --- | --- |
| Rule of Five (copy/move ctors/assign/dtor) | Excellent | Move semantics correctness |
| operator<< for debugging | Excellent | Format consistency |
| operator<=> (spaceship) | Good | Weak ordering vs strong ordering |
| JSON serialization (nlohmann) | Excellent | Missing fields, null handling |
| Protobuf converters | Good | Oneof handling |
| Hash function (std::hash) | Good | Hash quality for custom types |
| Builder pattern | Excellent | Template deduction issues |
| Visitor pattern boilerplate | Good | std::visit with variants |

---

## Self-Assessment

### Q1: Generate special member functions and operators

**Answer:**

The most error-prone part of Rule of Five generation is the copy constructor when the class has a `unique_ptr` member - a deep copy requires `std::make_unique<T>(*other.ptr_)`, and the AI needs to know that you want a deep copy rather than a shallow one. Be explicit in your prompt:

```cpp
=== PROMPT ===

"Generate the Rule of Five, operator==, operator<=>,
operator<<, and std::hash specialization for this class.
Use C++20 features. Follow Google Style Guide."
```

Starting from an input class with a `unique_ptr` member that needs deep copying, the AI generates all the required boilerplate. Notice the copy constructor's deep copy of `address_` and the `swap` helper that the copy assignment relies on:

```cpp
// Input class:
class Employee {
    std::string name_;
    int id_;
    std::vector<std::string> skills_;
    std::unique_ptr<Address> address_;  // Deep copy needed
public:
    Employee(std::string name, int id);
};

// === AI-generated boilerplate ===

class Employee {
    std::string name_;
    int id_;
    std::vector<std::string> skills_;
    std::unique_ptr<Address> address_;

public:
    // Constructor
    Employee(std::string name, int id)
        : name_(std::move(name)), id_(id) {}

    // Copy constructor (deep copy)
    Employee(const Employee& other)
        : name_(other.name_)
        , id_(other.id_)
        , skills_(other.skills_)
        , address_(other.address_
                   ? std::make_unique<Address>(*other.address_)
                   : nullptr) {}

    // Copy assignment (deep copy via copy-and-swap)
    Employee& operator=(const Employee& other) {
        if (this != &other) {
            Employee tmp(other);
            swap(*this, tmp);
        }
        return *this;
    }

    // Move constructor
    Employee(Employee&&) noexcept = default;

    // Move assignment
    Employee& operator=(Employee&&) noexcept = default;

    // Destructor
    ~Employee() = default;

    // Equality (C++20)
    bool operator==(const Employee& other) const {
        return id_ == other.id_
            && name_ == other.name_
            && skills_ == other.skills_
            && (address_ == other.address_
                || (address_ && other.address_
                    && *address_ == *other.address_));
    }

    // Ordering (by ID, then name)
    std::strong_ordering operator<=>(const Employee& other) const {
        if (auto cmp = id_ <=> other.id_; cmp != 0) return cmp;
        return name_ <=> other.name_;
    }

    // Stream output
    friend std::ostream& operator<<(std::ostream& os, const Employee& e) {
        os << "Employee{id=" << e.id_ << ", name=\"" << e.name_
           << "\", skills=[";
        for (size_t i = 0; i < e.skills_.size(); ++i) {
            if (i > 0) os << ", ";
            os << '"' << e.skills_[i] << '"';
        }
        os << "]}";
        return os;
    }

    // Swap (for copy-and-swap idiom)
    friend void swap(Employee& a, Employee& b) noexcept {
        using std::swap;
        swap(a.name_, b.name_);
        swap(a.id_, b.id_);
        swap(a.skills_, b.skills_);
        swap(a.address_, b.address_);
    }

    // Accessors for hash
    int id() const { return id_; }
    const std::string& name() const { return name_; }
};

// std::hash specialization
template<>
struct std::hash<Employee> {
    size_t operator()(const Employee& e) const noexcept {
        size_t h = std::hash<int>{}(e.id());
        h ^= std::hash<std::string>{}(e.name()) + 0x9e3779b9
             + (h << 6) + (h >> 2);
        return h;
    }
};
```

One thing to verify in AI-generated Rule of Five code: the move constructor is `= default` here, which is correct because all members (`string`, `vector`, `unique_ptr`) have their own move operations. If the class had raw pointers or non-RAII handles, `= default` would be wrong and you would need a hand-written move.

### Q2: Generate serialization code

**Answer:**

Serialization code is mechanical but has real correctness requirements: optional fields need explicit null-checking, enums need a string map, variants need type tagging, and missing fields should use sensible defaults rather than silently corrupting your struct. Specifying all of these in the prompt prevents the AI from picking a behavior you do not want:

```cpp
=== PROMPT ===

"Generate nlohmann::json serialization (to_json/from_json)
for this class hierarchy. Handle:

- Optional fields (std::optional)
- Enum serialization as strings
- Variant fields
- Null/missing field handling with defaults"
```

The code below shows what AI-generated serialization looks like for a struct with all four cases. The variant serialization is the most interesting part - it uses `std::visit` with `if constexpr` to dispatch on the active type:

```cpp
enum class Priority { Low, Medium, High, Critical };

struct Task {
    std::string title;
    Priority priority;
    std::optional<std::string> assignee;
    std::optional<std::chrono::system_clock::time_point> deadline;
    std::vector<std::string> tags;
    std::variant<int, std::string> reference;  // ticket # or URL
};

// === AI-generated serialization ===

// Enum string mapping
NLOHMANN_JSON_SERIALIZE_ENUM(Priority, {
    {Priority::Low, "low"},
    {Priority::Medium, "medium"},
    {Priority::High, "high"},
    {Priority::Critical, "critical"},
})

// Custom time_point serialization (ISO 8601)
inline void to_json(nlohmann::json& j,
                    const std::chrono::system_clock::time_point& tp) {
    auto t = std::chrono::system_clock::to_time_t(tp);
    std::ostringstream ss;
    ss << std::put_time(std::gmtime(&t), "%FT%TZ");
    j = ss.str();
}

inline void from_json(const nlohmann::json& j,
                      std::chrono::system_clock::time_point& tp) {
    std::tm tm = {};
    std::istringstream ss(j.get<std::string>());
    ss >> std::get_time(&tm, "%Y-%m-%dT%H:%M:%S");
    tp = std::chrono::system_clock::from_time_t(std::mktime(&tm));
}

// Task serialization
inline void to_json(nlohmann::json& j, const Task& t) {
    j = nlohmann::json{
        {"title", t.title},
        {"priority", t.priority},
        {"tags", t.tags}
    };
    if (t.assignee) j["assignee"] = *t.assignee;
    if (t.deadline) j["deadline"] = *t.deadline;

    // Variant serialization
    std::visit([&j](const auto& ref) {
        using T = std::decay_t<decltype(ref)>;
        if constexpr (std::is_same_v<T, int>)
            j["reference"] = {{"type", "ticket"}, {"value", ref}};
        else
            j["reference"] = {{"type", "url"}, {"value", ref}};
    }, t.reference);
}

inline void from_json(const nlohmann::json& j, Task& t) {
    j.at("title").get_to(t.title);
    j.at("priority").get_to(t.priority);
    t.tags = j.value("tags", std::vector<std::string>{});

    if (j.contains("assignee") && !j["assignee"].is_null())
        t.assignee = j["assignee"].get<std::string>();
    if (j.contains("deadline") && !j["deadline"].is_null())
        t.deadline = j["deadline"].get<
            std::chrono::system_clock::time_point>();

    // Variant deserialization
    if (j.contains("reference")) {
        auto type = j["reference"]["type"].get<std::string>();
        if (type == "ticket")
            t.reference = j["reference"]["value"].get<int>();
        else
            t.reference = j["reference"]["value"].get<std::string>();
    }
}
```

After reviewing AI-generated serialization code, specifically check that every field in the struct has a corresponding serialization path - omitted fields are a common AI error. Also check null handling in `from_json`: the `.at()` vs `.value()` choice determines whether a missing field throws or uses a default.

### Q3: Generate builder pattern and factory boilerplate

**Answer:**

The builder pattern is one of the cases where AI consistently produces clean output because the structure is entirely mechanical: one setter per field, method chaining by returning `*this`, and a `build()` method that validates and returns the final object. Using `std::expected` in `build()` gives you a way to report validation failures without exceptions:

```cpp
=== PROMPT ===

"Generate a fluent Builder pattern for this config class.
Use method chaining, validate on build(), return
std::expected<Config, std::string> from build()."
```

Here is the AI-generated builder for a server configuration struct with TLS support. Notice that the `enable_tls` method sets three fields atomically - this is the right design because you never want TLS enabled without a cert path:

```cpp
// === AI-generated builder ===
struct ServerConfig {
    std::string host;
    uint16_t port;
    size_t max_connections;
    std::chrono::seconds timeout;
    bool tls_enabled;
    std::optional<std::string> cert_path;
    std::optional<std::string> key_path;
};

class ServerConfigBuilder {
    ServerConfig config_{};

public:
    ServerConfigBuilder& host(std::string h) {
        config_.host = std::move(h); return *this;
    }
    ServerConfigBuilder& port(uint16_t p) {
        config_.port = p; return *this;
    }
    ServerConfigBuilder& max_connections(size_t n) {
        config_.max_connections = n; return *this;
    }
    ServerConfigBuilder& timeout(std::chrono::seconds t) {
        config_.timeout = t; return *this;
    }
    ServerConfigBuilder& enable_tls(
            std::string cert, std::string key) {
        config_.tls_enabled = true;
        config_.cert_path = std::move(cert);
        config_.key_path = std::move(key);
        return *this;
    }

    std::expected<ServerConfig, std::string> build() const {
        if (config_.host.empty())
            return std::unexpected("Host is required");
        if (config_.port == 0)
            return std::unexpected("Port must be non-zero");
        if (config_.max_connections == 0)
            return std::unexpected("Max connections must be > 0");
        if (config_.tls_enabled) {
            if (!config_.cert_path || !config_.key_path)
                return std::unexpected(
                    "TLS requires both cert and key paths");
        }
        return config_;
    }
};

// Usage:
auto config = ServerConfigBuilder()
    .host("0.0.0.0")
    .port(8080)
    .max_connections(1000)
    .timeout(std::chrono::seconds{30})
    .enable_tls("/etc/ssl/cert.pem", "/etc/ssl/key.pem")
    .build();

if (!config) {
    std::cerr << "Config error: " << config.error() << '\n';
    return 1;
}
```

The builder call site reads almost like prose, which is the whole point. The `build()` validation catches configuration errors at the point where the config object is created, not later when a connection attempt fails with a cryptic error.

---

## Notes

- Always specify **which C++ standard** and **coding style** in the prompt - these determine whether you get `= default`, `noexcept`, and `[[nodiscard]]` annotations.
- For `operator<=>`, tell AI whether you need **strong, weak, or partial** ordering - the choice affects what operations are defined and how `NaN` values behave.
- AI-generated `std::hash` may have poor distribution for certain key types - review for hash quality if you are using the type in performance-sensitive hash maps.
- For serialization, explicitly mention **how to handle missing and null fields** - the AI will pick one behavior by default and it may not be what you want.
- **Do not accept `unique_ptr` copy** in AI output - this is a common AI mistake because `unique_ptr` is move-only and copy is deleted. If you see `std::unique_ptr` in a copy constructor or copy assignment without `std::make_unique`, the AI got it wrong.
- Ask AI to generate boilerplate for **existing code** by pasting the class definition; the AI generates code that matches your actual members rather than inventing fields.
- For large classes, ask AI to generate boilerplate **incrementally** (operators first, then serialization) - doing too much at once degrades quality.
