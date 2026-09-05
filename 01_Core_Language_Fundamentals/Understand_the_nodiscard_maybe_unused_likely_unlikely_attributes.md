# Understand the [[nodiscard]], [[maybe_unused]], [[likely]], [[unlikely]] attributes

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#nodiscard>  

---

## Topic Overview

C++ attributes provide standardized hints to the compiler. These four are the most commonly used.

### [[nodiscard]] (C++17, enhanced C++20)

Warns if the return value is discarded. Use this whenever ignoring the return value is almost certainly a bug - error codes are the classic example.

```cpp
[[nodiscard]] int compute() { return 42; }

compute();  // WARNING: ignoring return value with 'nodiscard' attribute

// C++20: with message
[[nodiscard("Error codes must be checked")]]
ErrorCode send(const char* data);

send("hello");  // WARNING: Error codes must be checked

// Applied to types:
struct [[nodiscard]] ErrorCode { int code; };
ErrorCode parse(const char* s);
parse("123");  // WARNING: ignoring nodiscard type
```

### [[maybe_unused]] (C++17)

Suppresses unused-variable/parameter warnings for code that is intentionally unused in some build configurations - debug-only parameters being the most common case.

```cpp
void process([[maybe_unused]] int debug_level, int data) {
    // debug_level only used in debug builds
    assert(debug_level >= 0);  // Used only when NDEBUG is not defined
    do_work(data);
}

[[maybe_unused]] static void debug_helper() {
    // Only called in debug builds
}
```

### [[likely]] / [[unlikely]] (C++20)

Tells the compiler which branch you expect to be taken most often. This lets it arrange the machine code so the hot path has no branch - improving instruction cache utilization.

```cpp
if (error_code != 0) [[unlikely]] {
    handle_error(error_code);
} else [[likely]] {
    process_data();
}

switch (state) {
    case State::Running: [[likely]]
        tick(); break;
    case State::Error: [[unlikely]]
        recover(); break;
}
```

---

## Self-Assessment

### Q1: Add [[nodiscard]] to an error code return type and show the compiler warning it generates

There are two places to put `[[nodiscard]]`: on the function itself, or on the return type. When placed on the type, the warning applies to every function returning that type - a good fit for `ErrorCode` patterns.

```cpp
#include <iostream>
#include <string>

// Method 1: [[nodiscard]] on function
[[nodiscard("Must check error code")]]
int open_file(const std::string& path) {
    if (path.empty()) return -1;
    std::cout << "Opening: " << path << "\n";
    return 0;  // 0 = success
}

// Method 2: [[nodiscard]] on type (applies to ALL functions returning it)
struct [[nodiscard("ErrorCode must be handled")]] ErrorCode {
    int code;
    bool ok() const { return code == 0; }
};

ErrorCode connect(const std::string& host) {
    if (host.empty()) return {-1};
    return {0};
}

int main() {
    // These generate compiler warnings:
    open_file("test.txt");     // WARNING: ignoring return value
    connect("localhost");      // WARNING: ErrorCode must be handled

    // Correct usage:
    int result = open_file("test.txt");
    if (result != 0) std::cout << "Failed!\n";

    if (auto ec = connect("localhost"); !ec.ok()) {
        std::cout << "Connection failed: " << ec.code << "\n";
    }

    // Intentionally discard (suppress warning):
    (void)open_file("temp.txt");    // C-style cast to void
    [[maybe_unused]] auto _ = connect("host");  // Or assign to unused var
}

// Compile: g++ -std=c++20 -Wall file.cpp
// Warnings:
//   warning: ignoring return value of 'int open_file(const string&)',
//            declared with attribute 'nodiscard': 'Must check error code'
//   warning: ignoring return value of type 'ErrorCode',
//            declared with attribute 'nodiscard': 'ErrorCode must be handled'
```

When you genuinely want to discard a `[[nodiscard]]` result, cast to `void` or assign to a `[[maybe_unused]]` variable - both suppress the warning intentionally.

**How this works:**

- `[[nodiscard]]` on a function warns if its return value is unused.
- `[[nodiscard]]` on a type warns whenever ANY function returning that type has its result discarded.
- C++20 added the optional message string for clearer diagnostics.
- `(void)expr` explicitly discards the value and suppresses the warning.

### Q2: Use [[maybe_unused]] to suppress a warning on a debug-only parameter

In release builds, `assert` expands to nothing, which leaves `debug_context` unused. Without `[[maybe_unused]]` you'd get a warning; with it, the compiler is satisfied.

```cpp
#include <iostream>
#include <cassert>
#include <string>
#include <chrono>

// Parameter only used in debug builds
void validate_and_process(
    [[maybe_unused]] const std::string& debug_context,
    int data)
{
    assert(!debug_context.empty());  // Only active without NDEBUG
    // In release builds, debug_context is unused -> [[maybe_unused]] suppresses warning

    std::cout << "Processing: " << data << "\n";
}

// Variable only used in debug
void complex_operation(int x) {
    [[maybe_unused]] auto start = std::chrono::steady_clock::now();

    // ... actual work ...
    int result = x * 2;

    // Only in debug:
    assert(result > 0);

    [[maybe_unused]] auto elapsed = std::chrono::steady_clock::now() - start;
    // In release: start and elapsed are unused -> no warning

    std::cout << "Result: " << result << "\n";
}

// Function only called in debug
[[maybe_unused]] static void dump_state(int x) {
    std::cout << "DEBUG state: " << x << "\n";
}

int main() {
    validate_and_process("main_context", 42);
    complex_operation(21);

    #ifndef NDEBUG
    dump_state(100);
    #endif
}
```

`[[maybe_unused]]` replaces both the `(void)variable;` trick and platform-specific macros like `UNREFERENCED_PARAMETER` - it's cleaner and works everywhere.

**How this works:**

- `[[maybe_unused]]` tells the compiler: "I know this might be unused - don't warn."
- Works on: variables, parameters, functions, typedefs, class members, enumerators.
- Common use: debug-only code, platform-specific code, parameters required by interface but unused in implementation.

### Q3: Add [[likely]]/[[unlikely]] to a hot code path and inspect the assembly output

The attribute guides the compiler's code layout. In the generated assembly, the likely branch typically falls through (no jump instruction), while the unlikely branch is a taken branch - which is slower on modern CPUs.

```cpp
#include <iostream>
#include <vector>
#include <random>

// Hot path: most calls succeed, errors are rare
int process_request(int code) {
    if (code < 0) [[unlikely]] {
        // Error path — compiler will move this out of the hot path
        std::cerr << "Error: " << code << "\n";
        return -1;
    } else [[likely]] {
        // Success path — compiler optimizes for this branch
        return code * 2;
    }
}

// Switch with likely/unlikely
enum class Packet : int { Data = 0, Ack = 1, Error = 2, Heartbeat = 3 };

int handle_packet(Packet p) {
    switch (p) {
        case Packet::Data: [[likely]]
            return 1;  // Most common — compiler optimizes this case
        case Packet::Ack:
            return 2;
        case Packet::Error: [[unlikely]]
            return -1; // Rare — compiler moves this out of hot path
        case Packet::Heartbeat:
            return 0;
    }
    return -1;
}

int main() {
    // Benchmark: process_request is called millions of times
    // The [[likely]] hint helps the compiler lay out code so the common
    // path has no branch mispredictions

    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, 99);

    long sum = 0;
    for (int i = 0; i < 1'000'000; ++i) {
        int code = dist(rng);
        if (code < 2) code = -1;  // ~2% error rate
        sum += process_request(code);
    }
    std::cout << "Sum: " << sum << "\n";
}

// To inspect assembly on Compiler Explorer (godbolt.org):
// Compile with: -std=c++20 -O2
// Look for: the likely branch falls through (no jump),
//           the unlikely branch is jumped to (branch taken = rare)
```

Only reach for `[[likely]]`/`[[unlikely]]` after profiling shows a real bottleneck - the CPU's own branch predictor handles most cases well without help.

**How this works:**

- `[[likely]]` / `[[unlikely]]` guide the compiler's branch prediction and code layout.
- The compiler arranges the likely path with fall-through (no jump) and moves the unlikely path away.
- This improves instruction cache utilization and reduces branch misprediction penalties.
- Effect is most visible at `-O2` or `-O3` - examine with Compiler Explorer to see the jump layout.

---

## Notes

- `[[nodiscard]]` is essential on: error codes, factory functions, RAII resource handles, locks.
- `[[maybe_unused]]` replaces the old `(void)variable;` trick and the `UNREFERENCED_PARAMETER` macro.
- `[[likely]]`/`[[unlikely]]` replace GCC's `__builtin_expect(expr, value)` - standardized in C++20.
- Don't overuse `[[likely]]`/`[[unlikely]]` - the CPU's branch predictor is usually good enough. Use only on hot paths where profiling shows a benefit.
- C++23 adds `[[assume(expr)]]` - tells the compiler an expression is always true, enabling further optimization.
