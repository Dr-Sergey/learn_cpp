# Know when to use exceptions vs error codes vs std::expected

**Category:** Error Handling  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/error/exception>  

---

## Topic Overview

C++ offers three main error-handling mechanisms: **exceptions**, **error codes**, and **`std::expected`** (C++23). Each has distinct trade-offs in performance, expressiveness, and suitability for different domains. The hard part isn't understanding any one of them in isolation - it's knowing which one belongs where, especially when your codebase spans multiple layers or has performance requirements.

### The Three Mechanisms at a Glance

| Mechanism | API Style | Error Path Cost | Happy Path Cost | Composability |
| --- | --- | --- | --- | --- |
| **Exceptions** | `throw` / `try-catch` | High (~1-10 us per throw) | Zero (table-based unwinding) | Automatic propagation |
| **Error codes** | Return `int`/`enum`, check at call site | Near-zero | Small (branch per call) | Manual propagation |
| **`std::expected<T,E>`** | Return value-or-error | Near-zero | Small (slightly larger return) | Monadic chaining (C++23) |

### Decision Flow

Use this as a mental checklist rather than a rigid algorithm. The most important factor is whether the failure is genuinely unexpected or a routine outcome.

```cpp
Is this an EXCEPTIONAL condition (rare, unexpected)?
  ├─ YES -> Is the codebase exceptions-enabled?
  │         ├─ YES -> throw an exception
  │         └─ NO  -> error code or std::expected
  └─ NO (common/expected failure like "file not found", "invalid input")
           ├─ C++23 available? -> std::expected<T,E>
           └─ Pre-C++23?      -> error code / std::optional / out-param

Additional constraints:
  ├─ Real-time / embedded / -fno-exceptions -> error codes or expected
  ├─ noexcept API boundary                  -> error codes or expected
  └─ Cross-language boundary (C ABI)        -> error codes only
```

### Exception Model: Zero-Overhead Principle

Modern compilers use **table-based exception handling** (Itanium ABI on Linux, SEH on Windows). The zero-overhead principle means the happy path - the code that never throws - pays no runtime cost for the existence of try/catch blocks. The cost is only incurred when an exception is actually thrown.

```cpp
Happy path (no throw):
  ┌─────────────────────────────┐
  │ function()                  │
  │   ... normal code ...       │  <- ZERO runtime overhead
  │   return result;            │     (no extra branches, no checks)
  └─────────────────────────────┘

Error path (throw):
  ┌─────────────────────────────┐
  │ throw std::runtime_error(…) │
  │   1. Allocate exception obj │  <- ~100 ns
  │   2. Walk stack tables      │  <- ~1-10 us (depends on depth)
  │   3. Run destructors (RAII) │
  │   4. Transfer to catch      │
  └─────────────────────────────┘

Cost breakdown:
  Happy path:  0 extra instructions (compiler generates side tables)
  Error path:  ~1,000-10,000 ns (stack unwinding, RTTI matching)
  Code size:   +10-15% (exception tables in .eh_frame section)
```

### `std::expected<T, E>` (C++23) - Best of Both Worlds

`std::expected` gives you the type-safety of returning an error value without the overhead of exceptions on the error path, and the composability of monadic operations. Here's the basic form plus the chaining syntax.

```cpp
#include <expected>
#include <string>
#include <system_error>

// Returns value on success, error on failure — in the SAME return type
std::expected<int, std::string> parse_int(const std::string& s) {
    try {
        size_t pos;
        int val = std::stoi(s, &pos);
        if (pos != s.size())
            return std::unexpected("trailing characters");
        return val;
    } catch (const std::exception& e) {
        return std::unexpected(e.what());
    }
}

// Monadic chaining (C++23):
auto result = parse_int("42")
    .transform([](int v) { return v * 2; })          // 84
    .transform([](int v) { return std::to_string(v); }) // "84"
    .or_else([](const std::string& err) -> std::expected<std::string, std::string> {
        return std::unexpected("parse failed: " + err);
    });
```

If any step in the chain produces an error, the rest of the lambdas are skipped and the error flows through to the end. No boilerplate `if (!result) return result.error();` at each step.

### Important Notes

- **Consistency is key:** Pick one strategy per subsystem and stick with it. Mixing freely leads to unhandled errors.
- Mark non-throwing functions `noexcept` - enables better codegen and move semantics in containers.
- `std::expected` is a vocabulary type - return it from functions that can commonly fail.
- Exceptions cross abstraction boundaries naturally; error codes require manual propagation at each level.

---

## Self-Assessment

### Q1: List the scenarios where exceptions are inappropriate (embedded, real-time, `noexcept` APIs)

**Complete List of Scenarios Where Exceptions Should Be Avoided:**

| Scenario | Why Exceptions Are Problematic |
| --- | --- |
| **Real-time systems** | Stack unwinding has non-deterministic latency (~1-10 us); violates hard-deadline guarantees |
| **Embedded / bare-metal** | Exception tables consume Flash/ROM; unwinding runtime may be unavailable or too large |
| **`-fno-exceptions` codebases** | `throw` is disabled at compile time; entire codebase must be exception-free |
| **`noexcept` API boundaries** | Throwing from `noexcept` calls `std::terminate()` - undefined recovery |
| **Destructors** | Throwing during stack unwinding (already handling an exception) calls `std::terminate()` |
| **Move constructors/operators** | Throwing breaks `std::vector` reallocation strong guarantee; mark `noexcept` |
| **Swap functions** | Throwing swap breaks most standard algorithms' exception safety |
| **C ABI / FFI boundaries** | Exceptions cannot cross `extern "C"` boundaries - UB |
| **GPU / CUDA kernels** | Device code has no exception support |
| **High-frequency loops** | Even if "zero-cost" on happy path, compiler may inhibit optimizations near try/catch |
| **Expected failures** | "File not found" or "parse error" are common, not exceptional - use `expected` |

Here's a concrete example of the noexcept boundary problem. If you mark a callback `noexcept` and forget to guard against exceptions internally, any throw terminates the program with no useful diagnostic.

```cpp
// Example: noexcept boundary problem
void safe_callback() noexcept {
    // If this throws -> std::terminate()!
    // Must catch internally or use error codes
    try {
        risky_operation();
    } catch (...) {
        log_error();  // handle internally
    }
}
```

### Q2: Explain the zero-overhead principle: exceptions cost nothing if not thrown

**Detailed Explanation:**

```cpp
// The compiler generates TWO outputs for exception-enabled code:
//
// 1. Normal code path — IDENTICAL to code without exceptions
//    (no extra branches, no error checks, no "if (error) return")
//
// 2. Side tables (.eh_frame) stored in non-executed memory
//    (maps instruction ranges to cleanup actions / catch handlers)

int compute(int x) {
    std::string s = "hello";      // destructor registered in side table
    std::vector<int> v = {1,2,3}; // destructor registered in side table
    return process(s, v);         // if process() throws:
                                  //   unwinder reads tables,
                                  //   calls ~vector then ~string
}   // if no throw: s and v destroyed normally — zero overhead from exceptions
```

The key insight is that the side tables live in `.eh_frame`, a separate section that is mapped into memory but never read on the happy path. The instruction stream itself is identical to exception-free code. The overhead only materializes if an exception is actually thrown.

**Benchmark Evidence:**

```cpp
#include <iostream>
#include <chrono>
#include <stdexcept>

// Version 1: Exceptions (happy path)
int divide_exc(int a, int b) {
    if (b == 0) throw std::invalid_argument("div by zero");
    return a / b;
}

// Version 2: Error code
struct Result { int value; int error; };
Result divide_ec(int a, int b) {
    if (b == 0) return {0, -1};
    return {a / b, 0};
}

int main() {
    const int N = 100'000'000;

    // Happy path — exceptions
    auto t1 = std::chrono::high_resolution_clock::now();
    long long sum1 = 0;
    for (int i = 1; i <= N; ++i)
        sum1 += divide_exc(i, 3);
    auto t2 = std::chrono::high_resolution_clock::now();

    // Happy path — error codes
    auto t3 = std::chrono::high_resolution_clock::now();
    long long sum2 = 0;
    for (int i = 1; i <= N; ++i) {
        auto r = divide_ec(i, 3);
        sum2 += r.value;
    }
    auto t4 = std::chrono::high_resolution_clock::now();

    auto ms_exc = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    auto ms_ec  = std::chrono::duration_cast<std::chrono::milliseconds>(t4 - t3).count();

    std::cout << "Exceptions (happy): " << ms_exc << " ms, sum=" << sum1 << "\n";
    std::cout << "Error codes:        " << ms_ec  << " ms, sum=" << sum2 << "\n";
    // Typical result: both ~150-200 ms — no measurable difference on happy path
}
```

Both versions run in essentially the same time on the happy path. The performance difference only shows up on the error path, which is why "exceptions are slow" is only true for the throw/catch case, not for the surrounding code.

**Trade-offs:**

| Aspect | Happy Path | Error Path |
| --- | --- | --- |
| Runtime cost | Zero (table-based) | ~1-10 us per throw (unwinding) |
| Code size | +10-15% (exception tables) | N/A |
| Instruction cache | Slightly worse (larger binary) | N/A |
| Branch prediction | No branches for error checking | N/A |

### Q3: Rewrite a function that throws for a common case to use `std::expected` instead

**Solution - Before and After:**

```cpp
#include <iostream>
#include <expected>
#include <string>
#include <fstream>
#include <sstream>
#include <system_error>

// BAD: Throws for a COMMON failure (file not found is expected in many workflows)
std::string read_file_throws(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open())
        throw std::runtime_error("Cannot open: " + path);  // common failure!
    std::ostringstream ss;
    ss << file.rdbuf();
    if (file.bad())
        throw std::runtime_error("Read error: " + path);
    return ss.str();
}

// GOOD: Returns std::expected — caller handles failure without try/catch
enum class FileError {
    not_found,
    permission_denied,
    read_error
};

std::string to_string(FileError e) {
    switch (e) {
        case FileError::not_found:        return "file not found";
        case FileError::permission_denied: return "permission denied";
        case FileError::read_error:        return "read error";
    }
    return "unknown";
}

std::expected<std::string, FileError> read_file(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open())
        return std::unexpected(FileError::not_found);  // no throw, no unwinding
    std::ostringstream ss;
    ss << file.rdbuf();
    if (file.bad())
        return std::unexpected(FileError::read_error);
    return ss.str();  // implicitly constructs expected with value
}

int main() {
    // Using the throws version — verbose, easy to forget try/catch
    try {
        auto content = read_file_throws("config.txt");
        std::cout << content << "\n";
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
    }

    // Using the expected version — cleaner, compiler enforces checking
    auto result = read_file("config.txt");
    if (result) {
        std::cout << *result << "\n";
    } else {
        std::cerr << "Error: " << to_string(result.error()) << "\n";
    }

    // Monadic chaining (C++23):
    auto processed = read_file("data.txt")
        .transform([](const std::string& s) { return s.size(); })
        .transform([](size_t n) { return "File has " + std::to_string(n) + " bytes"; });

    std::cout << processed.value_or("Failed to read file") << "\n";

    // Composing multiple expected operations:
    auto config = read_file("settings.json")
        .and_then([](const std::string& json) -> std::expected<int, FileError> {
            // parse JSON, return port number or error
            return 8080;
        })
        .transform([](int port) { return "Server on port " + std::to_string(port); });
}
// Expected output (if files don't exist):
//   Error: Cannot open: config.txt
//   Error: file not found
//   Failed to read file
```

The `expected` version forces the caller to decide what to do with the error. With the throwing version, it's easy to call the function and forget the `try/catch` - the compiler won't remind you. With `expected`, ignoring the error means ignoring the return value, which at least produces a warning in most compilers.

**When to Use Which:**

| Situation | Use |
| --- | --- |
| Programmer bug (null deref, out-of-bounds) | `assert` / contract / terminate |
| Truly exceptional (out of memory, disk full) | `throw` exception |
| Common expected failure (parse, lookup, I/O) | `std::expected<T,E>` |
| C interop or legacy code | Error codes (`errno`, `int`) |
| Performance-critical inner loops | Error codes or `std::expected` |

---

## Notes

- **`std::expected` vs `std::optional`:** `expected<T,E>` carries an error value; `optional<T>` only says "no value" without saying why.
- **`std::unexpected(e)`** constructs the error state - it's the `expected` equivalent of `throw`.
- **Monadic operations** (`.transform()`, `.and_then()`, `.or_else()`) avoid deep nesting of `if (result)` checks.
- **Binary size:** `std::expected` adds no hidden tables, unlike exceptions which add `.eh_frame` overhead.
- **Migration tip:** Wrap legacy throwing APIs in a function returning `expected` at the boundary.
