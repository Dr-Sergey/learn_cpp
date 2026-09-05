# Design Error Hierarchies with Error Domains for Large Codebases

**Category:** Error Handling  
**Standard:** C++11, C++17, C++23 (`std::expected`)  
**Reference:** [cppreference - error_category](https://en.cppreference.com/w/cpp/error/error_category), [cppreference - error_code](https://en.cppreference.com/w/cpp/error/error_code)  

---

## Topic Overview

Large C++ codebases - spanning networking, storage, authentication, and business logic - need a structured way to define, categorize, and compare errors without coupling modules together. The `<system_error>` framework provides the building blocks: **error codes** (integer + category), **error categories** (singleton classifiers), and **error conditions** (portable comparison targets).

An **error domain** is a logical grouping: one `enum` of codes plus one `error_category` singleton. Each library or subsystem owns a domain. Cross-module comparison is enabled by mapping domain-specific codes to well-known `std::errc` conditions or to shared project-wide condition enums.

Here's the shape of the architecture visually. The application layer never needs to know which subsystem produced an error - it compares against shared conditions, and each domain's category handles the translation.

```cpp
┌──────────────────────────────────────────────────────────────────┐
│                         Application                              │
│   Compares against:  AppCondition::resource_unavailable          │
│                      std::errc::permission_denied                │
└────────────┬────────────────────────┬────────────────────────────┘
             │                        │
    ┌────────▼────────┐      ┌────────▼────────┐
    │  Network Domain  │      │  Storage Domain  │
    │  NetError enum   │      │  StorageError     │
    │  net_category()  │      │  storage_category()│
    └────────┬────────┘      └────────┬──────────┘
             │                        │
             ▼                        ▼
    default_error_condition()  default_error_condition()
    maps to AppCondition /     maps to AppCondition /
    std::errc                  std::errc
```

If the diagram feels dense, the core idea is simple: each domain translates its own specific codes into shared abstract conditions. The application layer only needs to know about the abstract conditions.

| Design Rule | Rationale |
| --- | --- |
| One `enum class` per module | Avoids integer collisions across modules |
| One `error_category` singleton per enum | Categories are compared by address - must be unique |
| Map to shared conditions via `default_error_condition()` | Enables portable cross-module `==` comparison |
| Register enums with `is_error_code_enum` | Enables implicit conversion to `std::error_code` |
| Optionally define project-wide condition enums | For domain concepts not covered by `std::errc` |

A well-designed error hierarchy lets callers write `if (ec == AppCondition::resource_unavailable)` without knowing which subsystem produced the error, while still exposing the exact subsystem error through `ec.message()` for logging.

---

## Self-Assessment

### Q1: Build a multi-domain error hierarchy with a shared condition enum

This example puts the whole pattern together: a shared application-level condition enum, two independent domain enums (network and storage), and a `handle()` function that dispatches only on the abstract condition. Notice how `default_error_condition()` is the bridge between specific domain codes and the abstract conditions.

```cpp
// error_hierarchy.cpp — C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o error_hierarchy error_hierarchy.cpp
#include <iostream>
#include <string>
#include <system_error>

// ====================================================================
// 1. Project-wide condition enum (shared across all modules)
// ====================================================================
enum class AppCondition {
    success              = 0,
    resource_unavailable = 1,
    authentication_failed = 2,
    data_corrupt         = 3,
};

class AppConditionCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "app"; }
    std::string message(int ev) const override {
        switch (static_cast<AppCondition>(ev)) {
            case AppCondition::success:               return "success";
            case AppCondition::resource_unavailable:   return "resource unavailable";
            case AppCondition::authentication_failed:  return "authentication failed";
            case AppCondition::data_corrupt:           return "data corrupt";
        }
        return "unknown app condition";
    }
};

const AppConditionCategory& app_condition_category() {
    static AppConditionCategory instance;
    return instance;
}

std::error_condition make_error_condition(AppCondition e) {
    return {static_cast<int>(e), app_condition_category()};
}

namespace std {
    template <> struct is_error_condition_enum<AppCondition> : true_type {};
}

// ====================================================================
// 2. Network domain
// ====================================================================
enum class NetError {
    ok              = 0,
    conn_refused    = 1,
    tls_handshake   = 2,
    dns_failure     = 3,
};

class NetCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "network"; }

    std::string message(int ev) const override {
        switch (static_cast<NetError>(ev)) {
            case NetError::ok:            return "no error";
            case NetError::conn_refused:  return "connection refused by remote host";
            case NetError::tls_handshake: return "TLS handshake failed";
            case NetError::dns_failure:   return "DNS resolution failed";
        }
        return "unknown network error";
    }

    std::error_condition default_error_condition(int ev) const noexcept override {
        switch (static_cast<NetError>(ev)) {
            case NetError::conn_refused:
            case NetError::dns_failure:
                return AppCondition::resource_unavailable;
            case NetError::tls_handshake:
                return AppCondition::authentication_failed;
            default:
                return std::error_condition(ev, *this);
        }
    }
};

const NetCategory& net_category() { static NetCategory c; return c; }

std::error_code make_error_code(NetError e) {
    return {static_cast<int>(e), net_category()};
}
namespace std { template <> struct is_error_code_enum<NetError> : true_type {}; }

// ====================================================================
// 3. Storage domain
// ====================================================================
enum class StorageError {
    ok             = 0,
    disk_full      = 1,
    checksum_fail  = 2,
    file_locked    = 3,
};

class StorageCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "storage"; }

    std::string message(int ev) const override {
        switch (static_cast<StorageError>(ev)) {
            case StorageError::ok:            return "no error";
            case StorageError::disk_full:     return "disk full";
            case StorageError::checksum_fail: return "data checksum mismatch";
            case StorageError::file_locked:   return "file is locked by another process";
        }
        return "unknown storage error";
    }

    std::error_condition default_error_condition(int ev) const noexcept override {
        switch (static_cast<StorageError>(ev)) {
            case StorageError::disk_full:
            case StorageError::file_locked:
                return AppCondition::resource_unavailable;
            case StorageError::checksum_fail:
                return AppCondition::data_corrupt;
            default:
                return std::error_condition(ev, *this);
        }
    }
};

const StorageCategory& storage_category() { static StorageCategory c; return c; }

std::error_code make_error_code(StorageError e) {
    return {static_cast<int>(e), storage_category()};
}
namespace std { template <> struct is_error_code_enum<StorageError> : true_type {}; }

// ====================================================================
// 4. Application: handle errors by condition, not by domain
// ====================================================================
void handle(std::error_code ec) {
    std::cout << "[" << ec.category().name() << ":" << ec.value()
              << "] " << ec.message() << "\n";

    if (ec == AppCondition::resource_unavailable) {
        std::cout << "  -> Action: retry or fail over\n";
    } else if (ec == AppCondition::authentication_failed) {
        std::cout << "  -> Action: re-authenticate\n";
    } else if (ec == AppCondition::data_corrupt) {
        std::cout << "  -> Action: invalidate cache, re-fetch\n";
    }
}

int main() {
    handle(NetError::conn_refused);
    handle(NetError::tls_handshake);
    handle(StorageError::disk_full);
    handle(StorageError::checksum_fail);
}
// Output:
// [network:1] connection refused by remote host
//   -> Action: retry or fail over
// [network:2] TLS handshake failed
//   -> Action: re-authenticate
// [storage:1] disk full
//   -> Action: retry or fail over
// [storage:2] data checksum mismatch
//   -> Action: invalidate cache, re-fetch
```

The `handle` function never mentions `NetError` or `StorageError`. It doesn't need to. The `default_error_condition()` override on each category does the translation, so `==` against an `AppCondition` just works. The category name and raw value are still available for logging via `ec.category().name()` and `ec.value()`.

### Q2: Ensure category singletons are safe across shared libraries (DLL/SO boundaries)

This is one of those problems that bites you silently. If two shared libraries each have their own copy of the static local inside the category accessor function, then error code comparisons between them will fail because they compare category pointer addresses - and those addresses will differ. The fix is to use `inline` functions (C++17), which guarantees a single definition and a single address across all translation units.

```cpp
// singleton_safety.cpp — C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o singleton_safety singleton_safety.cpp
#include <iostream>
#include <system_error>

// Problem: if each shared library has its own copy of the static local,
// error_category comparison by address will fail across library boundaries.
//
// Solution: use an inline function with a Meyers singleton, and ensure the
// category is defined in a SINGLE translation unit (or header-only with inline).

// Header-only pattern (safe with inline since C++17)
enum class AuthError { ok = 0, bad_token = 1, expired = 2 };

class AuthCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "auth"; }
    std::string message(int ev) const override {
        switch (static_cast<AuthError>(ev)) {
            case AuthError::ok:        return "success";
            case AuthError::bad_token: return "invalid authentication token";
            case AuthError::expired:   return "token expired";
        }
        return "unknown auth error";
    }
};

// C++17 inline variable guarantees ONE address across all TUs
inline const AuthCategory& auth_category() noexcept {
    static const AuthCategory instance;
    return instance;
}

inline std::error_code make_error_code(AuthError e) {
    return {static_cast<int>(e), auth_category()};
}

namespace std {
    template <> struct is_error_code_enum<AuthError> : true_type {};
}

// Simulate two "libraries" getting the category
const std::error_category* get_from_lib_a() { return &auth_category(); }
const std::error_category* get_from_lib_b() { return &auth_category(); }

int main() {
    // Verify same address — this is how operator== works
    const auto* a = get_from_lib_a();
    const auto* b = get_from_lib_b();

    std::cout << "lib_a category address: " << a << "\n";
    std::cout << "lib_b category address: " << b << "\n";
    std::cout << "Same singleton? " << (a == b ? "YES" : "NO — BUG!") << "\n";

    // Prove comparison works
    std::error_code ec = AuthError::bad_token;
    if (ec == AuthError::bad_token) {
        std::cout << "Cross-TU comparison works: " << ec.message() << "\n";
    }
}
// Output:
// lib_a category address: 0x... (same)
// lib_b category address: 0x... (same)
// Same singleton? YES
// Cross-TU comparison works: invalid authentication token
```

If you see "NO - BUG!" here, you have a duplicate singleton problem and your error code comparisons across library boundaries will silently lie to you. The `inline` keyword on both the function and the category accessor is your insurance policy against this.

### Q3: Compose error domains with `std::expected` (C++23) for ergonomic propagation

C++23's `std::expected` pairs naturally with the `std::error_code` domain infrastructure. You get the domain system's clean categorization and the monadic chaining that `expected` enables - propagating errors up the call chain is just a matter of returning `std::unexpected(val.error())`.

```cpp
// expected_domains.cpp — C++23
// Compile: g++ -std=c++23 -O2 -Wall -Wextra -o expected_domains expected_domains.cpp
#include <expected>
#include <iostream>
#include <string>
#include <system_error>

// --- Reuse a simple domain -----------------------------------------
enum class ParseError { ok = 0, bad_syntax = 1, overflow = 2 };

class ParseCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "parse"; }
    std::string message(int ev) const override {
        switch (static_cast<ParseError>(ev)) {
            case ParseError::ok:         return "success";
            case ParseError::bad_syntax: return "invalid syntax";
            case ParseError::overflow:   return "numeric overflow";
        }
        return "unknown parse error";
    }
};

inline const ParseCategory& parse_category() noexcept {
    static const ParseCategory instance;
    return instance;
}

std::error_code make_error_code(ParseError e) {
    return {static_cast<int>(e), parse_category()};
}
namespace std { template <> struct is_error_code_enum<ParseError> : true_type {}; }

// --- Functions returning std::expected<T, std::error_code> ----------
using Result = std::expected<int, std::error_code>;

Result parse_int(const std::string& s) {
    try {
        size_t pos = 0;
        long long val = std::stoll(s, &pos);
        if (pos != s.size())
            return std::unexpected(ParseError::bad_syntax);
        if (val > std::numeric_limits<int>::max() || val < std::numeric_limits<int>::min())
            return std::unexpected(ParseError::overflow);
        return static_cast<int>(val);
    } catch (...) {
        return std::unexpected(ParseError::bad_syntax);
    }
}

Result double_it(const std::string& s) {
    auto val = parse_int(s);
    if (!val)
        return std::unexpected(val.error());  // propagate error_code unchanged
    return *val * 2;
}

int main() {
    for (const auto& input : {"42", "hello", "99999999999999999"}) {
        auto result = double_it(input);
        if (result) {
            std::cout << input << " -> " << *result << "\n";
        } else {
            std::error_code ec = result.error();
            std::cout << input << " -> ERROR ["
                      << ec.category().name() << ":" << ec.value()
                      << "] " << ec.message() << "\n";
        }
    }
}
// Output:
// 42 -> 84
// hello -> ERROR [parse:1] invalid syntax
// 99999999999999999 -> ERROR [parse:2] numeric overflow
```

The error code flows through `double_it` untouched, carrying both the domain identity (the category) and the specific code. The caller gets all the information it needs to either display the error or handle it by category.

---

## Notes

- **One enum + one category per module** eliminates integer collisions and provides clear ownership of error codes.
- Category singletons must have **exactly one address** across the entire program. Use `inline` functions or variables (C++17) to avoid ODR issues across shared libraries.
- Define a **project-wide `error_condition` enum** (like `AppCondition`) for concepts beyond `std::errc` - subsystem-specific conditions such as "data corrupt" or "rate limited."
- `default_error_condition()` is your mapping layer: override it to connect domain codes to shared conditions.
- `is_error_code_enum<E>` enables implicit `error_code` construction. `is_error_condition_enum<E>` enables implicit `error_condition` construction. Register the right one.
- In C++23, `std::expected<T, std::error_code>` gives monadic error propagation while retaining the full category/domain infrastructure.
- Document the mapping table (domain code -> condition) in each module's header - this is the "error contract" for callers.
