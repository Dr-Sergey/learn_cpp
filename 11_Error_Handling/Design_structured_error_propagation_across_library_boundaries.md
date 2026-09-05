# Design Structured Error Propagation Across Library Boundaries

**Category:** Error Handling  
**Standard:** C++11, C++17, C++23 (`std::expected`)  
**Reference:** [cppreference - std::expected](https://en.cppreference.com/w/cpp/utility/expected), [cppreference - std::error_code](https://en.cppreference.com/w/cpp/error/error_code)  

---

## Topic Overview

When code crosses a library boundary - a shared library (`.so`/`.dll`), a module boundary, or a third-party SDK - error propagation becomes a design problem with real constraints. Exceptions may not cross ABI boundaries safely, error enums from one library are opaque to another, and version changes can break error contracts.

| Mechanism | Pros | Cons | Best for |
| --- | --- | --- | --- |
| Exceptions | Natural RAII unwinding, rich payloads | ABI-unsafe across compilers/versions, performance cost | Internal code, same-compiler boundaries |
| `std::error_code` | ABI-stable (int + pointer), extensible | No payload, requires category registration | C-compatible boundaries, OS-level APIs |
| `std::expected<T,E>` | Monadic chaining, zero overhead | C++23 minimum, value type (copies the error) | Modern intra-project APIs |
| C-style `int` return | Maximum ABI stability | No type safety, global errno patterns | Pure C interfaces, FFI |

A robust boundary design typically uses a **layered approach**: the public ABI returns a stable error type (`error_code` or C `int`), while internal code uses exceptions or `std::expected`. A thin translation layer at the boundary catches exceptions and converts them to error codes.

Here's the three-layer picture. The internal code can be as expressive as you like; the translation layer is the firewall.

```cpp
┌─────────────────────────────────────────────────────────────────┐
│  Application (C++23)                                            │
│  Uses: std::expected<T, AppError> internally                    │
│  Catches exceptions from dependencies                           │
├─────────────────────────────────────────── ABI BOUNDARY ────────┤
│  Public Library API (C++17 / extern "C")                        │
│  Returns: std::error_code  or  int error + out-param            │
│  Translates: internal exceptions -> error_code                  │
├─────────────────────────────────────────── ABI BOUNDARY ────────┤
│  Platform / Third-Party SDK                                     │
│  Returns: errno, HRESULT, or its own error codes                │
│  Translation at import: platform code -> std::error_code        │
└─────────────────────────────────────────────────────────────────┘
```

Key design principles:

1. **Never let exceptions escape a library boundary** unless the library and caller are guaranteed to use the same compiler, standard library, and exception ABI.
2. **Version your error enums.** Add new codes at the end. Never remove or renumber. Provide an `unknown` fallback.
3. **Provide a `message()` function** that returns a human-readable string allocated by the library - the caller must not free or interpret the format.
4. **Map incoming platform errors to your domain** at the boundary - don't expose raw `errno` or `HRESULT` to callers.

---

## Self-Assessment

### Q1: Implement a boundary translation layer that catches exceptions and returns `error_code`

The key insight here is the `noexcept` on the public API function. That's the promise to callers that nothing will escape. Inside, a `try/catch(...)` with fine-grained catches maps each known internal exception type to a specific public error code. Any other exception becomes `internal_error`, which is honest but doesn't leak internal details.

```cpp
// boundary_translation.cpp — C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o boundary_translation boundary_translation.cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <system_error>

// ====================================================================
// Library internals — uses exceptions freely
// ====================================================================
namespace internal {

struct ParseResult {
    int value;
    std::string unit;
};

ParseResult parse_measurement(const std::string& input) {
    if (input.empty())
        throw std::invalid_argument("empty input");
    if (input == "overflow")
        throw std::overflow_error("value exceeds range");
    if (input == "unknown")
        throw std::runtime_error("unrecognized format");
    return {42, "kg"};
}

}  // namespace internal

// ====================================================================
// Public API error codes
// ====================================================================
enum class MeasureError {
    ok              = 0,
    invalid_input   = 1,
    out_of_range    = 2,
    internal_error  = 3,   // catch-all for unexpected exceptions
};

class MeasureCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "measure"; }
    std::string message(int ev) const override {
        switch (static_cast<MeasureError>(ev)) {
            case MeasureError::ok:             return "success";
            case MeasureError::invalid_input:  return "invalid measurement input";
            case MeasureError::out_of_range:   return "measurement out of range";
            case MeasureError::internal_error: return "internal measurement error";
        }
        return "unknown measurement error";
    }
};

inline const MeasureCategory& measure_category() noexcept {
    static const MeasureCategory instance;
    return instance;
}

std::error_code make_error_code(MeasureError e) {
    return {static_cast<int>(e), measure_category()};
}

namespace std {
    template <> struct is_error_code_enum<MeasureError> : true_type {};
}

// ====================================================================
// Public API — catches all exceptions at the boundary
// ====================================================================
struct MeasureResult {
    int value;
    char unit[16];
};

std::error_code parse_measurement_api(const char* input, MeasureResult* out) noexcept {
    try {
        auto result = internal::parse_measurement(input ? input : "");
        if (out) {
            out->value = result.value;
            snprintf(out->unit, sizeof(out->unit), "%s", result.unit.c_str());
        }
        return {};
    } catch (const std::invalid_argument&) {
        return MeasureError::invalid_input;
    } catch (const std::overflow_error&) {
        return MeasureError::out_of_range;
    } catch (...) {
        return MeasureError::internal_error;
    }
}

// ====================================================================
// Client code
// ====================================================================
void test(const char* input) {
    MeasureResult result{};
    auto ec = parse_measurement_api(input, &result);
    if (!ec) {
        std::cout << "\"" << input << "\" -> " << result.value
                  << " " << result.unit << "\n";
    } else {
        std::cout << "\"" << input << "\" -> ERROR ["
                  << ec.category().name() << ":" << ec.value()
                  << "] " << ec.message() << "\n";
    }
}

int main() {
    test("100kg");
    test("");
    test("overflow");
    test("unknown");
}
// Output:
// "100kg" -> 42 kg
// "" -> ERROR [measure:1] invalid measurement input
// "overflow" -> ERROR [measure:2] measurement out of range
// "unknown" -> ERROR [measure:3] internal measurement error
```

The public function is `noexcept` and returns a plain `std::error_code`. Everything messy is contained inside. This is the fundamental shape of a well-behaved library boundary.

### Q2: Design a versioned error type for a library that must maintain ABI compatibility

When you ship a library that other codebases link against, you can't change the layout of your error type between versions - their compiled code will break. The trick is a POD struct with a `version` field so callers that were compiled against v1 can still handle v2 codes gracefully (by checking `err.version > known_version`).

```cpp
// versioned_errors.cpp — C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o versioned_errors versioned_errors.cpp
#include <cstdint>
#include <cstring>
#include <iostream>

// ====================================================================
// ABI-stable error type — works across compiler versions and languages
// ====================================================================

// Version 1 had codes 0-3. Version 2 adds codes 4-5.
// Rule: NEVER reorder, NEVER remove. Always add at end.
struct LibError {
    int32_t code;       // numeric error code
    int32_t version;    // API version that introduced this code

    // Predefined constants — part of the ABI
    static constexpr LibError ok()              { return {0, 1}; }
    static constexpr LibError not_found()       { return {1, 1}; }
    static constexpr LibError access_denied()   { return {2, 1}; }
    static constexpr LibError timeout()         { return {3, 1}; }
    // Added in v2:
    static constexpr LibError rate_limited()    { return {4, 2}; }
    static constexpr LibError deprecated()      { return {5, 2}; }

    bool operator==(const LibError& o) const { return code == o.code; }
    bool operator!=(const LibError& o) const { return code != o.code; }
    explicit operator bool() const { return code != 0; }
};

// ABI-stable message function — library owns the string lifetime
extern "C"
const char* lib_error_message(LibError err) noexcept {
    switch (err.code) {
        case 0: return "success";
        case 1: return "resource not found";
        case 2: return "access denied";
        case 3: return "operation timed out";
        case 4: return "rate limited";
        case 5: return "API deprecated";
        default: return "unknown error (check library version)";
    }
}

// ABI-stable API version query
extern "C"
int32_t lib_api_version() noexcept { return 2; }

// ====================================================================
// Simulated library function with extern "C" boundary
// ====================================================================
extern "C"
LibError lib_fetch_resource(const char* name, char* buf, int32_t buf_size) noexcept {
    if (!name || !buf || buf_size <= 0)
        return LibError::access_denied();
    if (strcmp(name, "secret") == 0)
        return LibError::access_denied();
    if (strcmp(name, "old_api") == 0)
        return LibError::deprecated();

    snprintf(buf, static_cast<size_t>(buf_size), "data_for_%s", name);
    return LibError::ok();
}

// ====================================================================
// Client: checks version, handles unknown codes gracefully
// ====================================================================
int main() {
    std::cout << "Library API version: " << lib_api_version() << "\n\n";

    auto test = [](const char* resource) {
        char buf[256] = {};
        LibError err = lib_fetch_resource(resource, buf, sizeof(buf));

        if (!err) {
            std::cout << resource << " -> " << buf << "\n";
        } else {
            std::cout << resource << " -> ERROR " << err.code
                      << " (v" << err.version << "): "
                      << lib_error_message(err) << "\n";

            // Client on API v1 might not know v2 codes
            if (err.version > 1) {
                std::cout << "  (Note: this error code was added in API v"
                          << err.version << ")\n";
            }
        }
    };

    test("report");
    test("secret");
    test("old_api");
    test(nullptr);
}
// Output:
// Library API version: 2
//
// report -> data_for_report
// secret -> ERROR 2 (v1): access denied
// old_api -> ERROR 5 (v2): API deprecated
//   (Note: this error code was added in API v2)
// (null) -> ERROR 2 (v1): access denied
```

The `version` field in the struct means a v1 client can receive a v2 error code and know "this is something I don't understand yet" rather than silently treating it as some other v1 code. That's the difference between a defensively designed API and one that breaks quietly when you update the library.

### Q3: Bridge `std::expected` (internal) with `std::error_code` (public boundary) using a translation utility

This is the cleanest pattern for modern C++ libraries: use `std::expected` throughout your internal implementation for expressive, chainable error handling, and then expose a `std::error_code`-returning public API. The bridge utility `to_error_code` is the glue.

```cpp
// expected_bridge.cpp — C++23
// Compile: g++ -std=c++23 -O2 -Wall -Wextra -o expected_bridge expected_bridge.cpp
#include <expected>
#include <iostream>
#include <string>
#include <system_error>

// ====================================================================
// Domain error codes (same pattern as previous examples)
// ====================================================================
enum class ConfigError {
    ok           = 0,
    missing_key  = 1,
    parse_failed = 2,
    access_error = 3,
};

class ConfigCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "config"; }
    std::string message(int ev) const override {
        switch (static_cast<ConfigError>(ev)) {
            case ConfigError::ok:           return "success";
            case ConfigError::missing_key:  return "configuration key not found";
            case ConfigError::parse_failed: return "configuration value parse error";
            case ConfigError::access_error: return "configuration file access denied";
        }
        return "unknown config error";
    }
};

inline const ConfigCategory& config_category() noexcept {
    static const ConfigCategory instance;
    return instance;
}

std::error_code make_error_code(ConfigError e) {
    return {static_cast<int>(e), config_category()};
}
namespace std { template <> struct is_error_code_enum<ConfigError> : true_type {}; }

// ====================================================================
// Generic bridge: std::expected<T, std::error_code> -> callback with error_code
// ====================================================================
template <typename T>
std::error_code to_error_code(
    const std::expected<T, std::error_code>& result,
    T* out) noexcept
{
    if (result.has_value()) {
        if (out) *out = result.value();
        return {};
    }
    return result.error();
}

// ====================================================================
// Internal API — uses std::expected
// ====================================================================
namespace internal {

using ConfigResult = std::expected<int, std::error_code>;

ConfigResult read_config_int(const std::string& key) {
    if (key == "port")     return 8080;
    if (key == "timeout")  return 30;
    if (key == "invalid")  return std::unexpected(ConfigError::parse_failed);
    return std::unexpected(ConfigError::missing_key);
}

ConfigResult read_and_validate(const std::string& key, int min, int max) {
    return read_config_int(key)
        .and_then([&](int val) -> ConfigResult {
            if (val < min || val > max)
                return std::unexpected(ConfigError::parse_failed);
            return val;
        });
}

}  // namespace internal

// ====================================================================
// Public boundary — returns error_code, not expected
// ====================================================================
extern "C"
std::error_code config_read_int(const char* key, int* out) noexcept {
    try {
        auto result = internal::read_and_validate(
            key ? key : "", 0, 65535);
        return to_error_code(result, out);
    } catch (...) {
        return ConfigError::access_error;  // catch-all
    }
}

// ====================================================================
// Client
// ====================================================================
void query(const char* key) {
    int value = 0;
    auto ec = config_read_int(key, &value);
    if (!ec) {
        std::cout << key << " = " << value << "\n";
    } else {
        std::cout << key << " -> ERROR [" << ec.category().name()
                  << ":" << ec.value() << "] " << ec.message() << "\n";
    }
}

int main() {
    query("port");
    query("timeout");
    query("invalid");
    query("missing");
}
// Output:
// port = 8080
// timeout = 30
// invalid -> ERROR [config:2] configuration value parse error
// missing -> ERROR [config:1] configuration key not found
```

The internal chain (`read_and_validate` calling `read_config_int` and using `.and_then`) is clean and readable. The public boundary collapses all of that into a simple output-parameter + error-code pattern that works from C or any C++ version. Best of both worlds.

---

## Notes

- **Never let C++ exceptions escape `extern "C"` boundaries** - it is undefined behavior. Always wrap in `try/catch(...)`.
- At ABI boundaries, prefer `std::error_code` (int + category pointer) or plain `int` error codes. Both are trivially copyable and ABI-stable.
- `std::expected<T, std::error_code>` is ideal for internal APIs. Bridge it to `error_code` return values at the public boundary.
- **Version your error enums**: add a version tag to new codes, never renumber old ones, and always handle `default`/`unknown` gracefully.
- Provide a library-owned `message()` function - don't force callers to maintain their own error string tables.
- `std::error_code` with custom categories works well for inter-module communication within the same process. For cross-process or cross-language boundaries, use plain C integers.
- The translation layer pattern (catch internal exceptions -> map to error codes) should be thin and mechanically verifiable. Consider code-generating it from an error definition table.
- When wrapping a third-party library, map its error codes into your domain at the import boundary - don't leak foreign error categories to your callers.
