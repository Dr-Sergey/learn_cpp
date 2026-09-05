# Design exception-safe vs noexcept library boundaries

**Category:** API and Library Design  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#e-error-handling>  

---

## Topic Overview

Library boundaries are where your error handling strategy matters most - because it is at boundaries that different codebases, different compilers, and sometimes different languages meet. The choice between exceptions and `noexcept` is not just a style preference; it affects performance, ABI compatibility, and composability with other code.

The reason this trips people up is that C++ lets you mix both approaches, and sometimes you need to. The best-designed libraries often do exactly that.

### Design Decision Matrix

The table below summarizes which error strategy fits which situation. The key insight is that `noexcept` is not just an optimization hint - for move operations, destructors, and `swap`, it has concrete behavioral consequences.

| Scenario | Recommendation |
| --- | --- |
| Performance-critical inner loop | noexcept + error codes/expected |
| Constructor failure | Exceptions (constructors can't return error codes) |
| Destructor | Always noexcept (implicit since C++11) |
| Move operations | noexcept (required for `vector::push_back` optimization) |
| Swap | noexcept |
| C interop boundary | noexcept + error return |
| General library API | Exceptions or `std::expected` |

### The Dual-API Pattern

A clean library design often offers both: a throwing version for application code that wants simple error propagation, and a non-throwing version for performance-sensitive or embedded use cases. Here is what that looks like for a file-reading operation:

```cpp
#include <expected>
#include <stdexcept>
#include <string>
#include <fstream>
#include <system_error>

namespace mylib {

// Throwing version (convenient for scripts, applications)
std::string read_file(const std::string& path) {
    std::ifstream f(path);
    if (!f) throw std::runtime_error("Cannot open: " + path);
    return std::string(std::istreambuf_iterator<char>(f), {});
}

// Non-throwing version (for performance-critical code, embedded)
std::expected<std::string, std::error_code>
try_read_file(const std::string& path) noexcept {
    try {
        std::ifstream f(path);
        if (!f) return std::unexpected(
            std::make_error_code(std::errc::no_such_file_or_directory));
        return std::string(std::istreambuf_iterator<char>(f), {});
    } catch (...) {
        return std::unexpected(
            std::make_error_code(std::errc::io_error));
    }
}

} // namespace mylib
```

Notice that `try_read_file` is marked `noexcept` even though its body contains a `try`/`catch`. That is intentional - the `noexcept` promise is upheld because all exceptions are caught inside the function and converted to `std::unexpected` values. The caller gets an error code, never an exception.

### Exception Safety at Library Boundaries

When your shared library is loaded into a process, its exception ABI must match the caller's. Different compilers and different compiler versions can have incompatible exception ABIs. The safe rule is: exceptions must not escape a shared library boundary. This example shows the catch-everything-at-the-boundary pattern:

```cpp
// When your library throws, catch at the boundary and convert
// to a stable ABI-safe error type

#ifdef _WIN32
#define MYLIB_EXPORT __declspec(dllexport)
#else
#define MYLIB_EXPORT __attribute__((visibility("default")))
#endif

// C-compatible error return for shared library boundary
extern "C" {
    enum MyLibError { MYLIB_OK = 0, MYLIB_BAD_INPUT = 1, MYLIB_IO_ERROR = 2 };

    MYLIB_EXPORT MyLibError mylib_process(const char* input, char* output, size_t outlen) {
        try {
            // Internal C++ code can use exceptions freely
            std::string result = mylib::process(input);
            if (result.size() >= outlen) return MYLIB_BAD_INPUT;
            std::copy(result.begin(), result.end(), output);
            output[result.size()] = '\0';
            return MYLIB_OK;
        } catch (const std::invalid_argument&) {
            return MYLIB_BAD_INPUT;
        } catch (...) {
            return MYLIB_IO_ERROR;
        }
    }
}
```

Inside the library, you are completely free to use exceptions - they are caught before anything escapes. The caller across the boundary only ever sees a plain C enum.

---

## Self-Assessment

### Q1: Why must move constructors be noexcept for vector compatibility

`std::vector::push_back` uses `std::move_if_noexcept` internally. When it needs to grow its buffer, it has to move all existing elements into the new allocation. If the move constructor is not `noexcept`, vector falls back to copying during reallocation instead - because a throwing move that fails halfway through would leave some elements moved and others not, with no way to restore the original state. The vector cannot satisfy the strong exception guarantee if moves can throw. Marking your move constructor `noexcept` tells vector it is safe to use moves, which is both faster and what you almost always want.

### Q2: Design an error code enum that works with std::error_code

To plug into the `std::error_code` ecosystem (which is what `std::expected` and many libraries use), you need to define an error category and specialize `is_error_code_enum`. Here is the complete boilerplate:

```cpp
#include <system_error>

enum class LibError {
    Success = 0,
    FileNotFound = 1,
    ParseError = 2,
    Timeout = 3,
};

struct LibErrorCategory : std::error_category {
    const char* name() const noexcept override { return "mylib"; }
    std::string message(int ev) const override {
        switch (static_cast<LibError>(ev)) {
            case LibError::Success: return "success";
            case LibError::FileNotFound: return "file not found";
            case LibError::ParseError: return "parse error";
            case LibError::Timeout: return "timeout";
            default: return "unknown error";
        }
    }
};

const LibErrorCategory& lib_error_category() {
    static LibErrorCategory instance;
    return instance;
}

std::error_code make_error_code(LibError e) {
    return {static_cast<int>(e), lib_error_category()};
}

namespace std {
    template<> struct is_error_code_enum<LibError> : true_type {};
}
```

Once this is in place, `LibError` values convert implicitly to `std::error_code`, which means they compose with `std::expected<T, std::error_code>` and the rest of the standard error machinery.

### Q3: Explain the "catch everything at the boundary" pattern

At shared library boundaries (DLL or `.so`), C++ exceptions must not escape. The reason is that different compilers and different compiler versions can have incompatible exception ABIs - the runtime metadata used to unwind the stack and match `catch` clauses is not standardized at the binary level. The boundary function wraps everything in `try { ... } catch (...) { return ERROR_CODE; }`, guaranteeing that nothing propagates. Inside the library, you are free to use exceptions however you like - the catch-all at the boundary is the firewall.

---

## Notes

- **Exceptions cannot cross `extern "C"` boundaries** - always catch and convert. This is a hard rule, not a guideline.
- Move and swap should always be `noexcept` - this is the single most impactful `noexcept` decision you make for a type.
- The dual-API pattern (throwing + `expected`) serves both convenience and performance users without forcing a choice on anyone.
- `std::expected` (C++23) is the modern alternative to error codes at library boundaries - it carries either a value or an error, and forces the caller to handle both cases explicitly.
