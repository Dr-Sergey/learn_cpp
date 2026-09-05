# Understand the Herb Sutter deterministic exceptions proposal and its alternatives

**Category:** Error Handling  
**Standard:** C++23  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0709r0.pdf>  

---

## Topic Overview

Herb Sutter's **P0709 "Zero-overhead deterministic exceptions"** proposal (2018) aimed to fix the fundamental problems with C++ exceptions: non-deterministic latency, code size bloat, and ABI fragility. While P0709 was not adopted as-is, it heavily influenced `std::expected<T,E>` (C++23) and ongoing work on error handling.

### The Problem with Traditional C++ Exceptions

To understand why P0709 mattered, you need to understand what the traditional exception model actually costs at runtime. The core issue is that "zero-cost" only means zero cost on the happy path on the Itanium ABI. There are real costs elsewhere:

```cpp
// Traditional Exception Model (Itanium ABI):
//
// throw site:                          catch site:
//   throw runtime_error  ----------->  catch(exception& e)
//     1. Allocate obj      unwinding     handle error
//     2. Copy/move exc   <---------->
//     3. Walk stack        RTTI match
//
// Problems:
//   1. Non-deterministic latency: stack depth determines cost (~1-10 us)
//   2. Code size: .eh_frame tables add 10-15% to binary
//   3. ABI lock-in: exception tables are ABI-specific, hard to change
//   4. Can't use in embedded/RT: many codebases compile with -fno-exceptions
//   5. Success path: ACTUALLY zero-cost on Itanium ABI, but some embedded
//      ABIs (e.g., legacy ARM EHABI) have non-zero happy-path overhead
```

The non-determinism is the killer for real-time and embedded systems. You cannot say "this function will return within X microseconds" if throwing an exception deep in the call stack might spend an unpredictable amount of time walking the stack.

### Sutter's P0709 Proposal: `throws` Values

The core idea of P0709 was elegant: keep the `throw`/`catch` syntax developers know, but compile it as a return value internally. This gives you ergonomic error propagation with deterministic timing.

```cpp
// Proposed syntax (P0709 - NOT adopted):
double safe_sqrt(double x) throws {   // 'throws' = can fail deterministically
    if (x < 0) throw std::error(errc::invalid_argument);
    return std::sqrt(x);
}

// Internally compiled as if:
std::expected<double, std::error> safe_sqrt(double x) {
    if (x < 0) return std::unexpected(std::error(errc::invalid_argument));
    return std::sqrt(x);
}

// But with exception SYNTAX (throw/try/catch) for ergonomics
```

The beauty of this approach is that you get automatic error propagation (like traditional exceptions) while the compiled code behaves exactly like a value return. No stack tables, no unwinding, no RTTI matching - just a discriminated union returned by value.

**Key Properties of P0709:**

| Property | Traditional Exceptions | P0709 `throws` | `std::expected` |
| --- | --- | --- | --- |
| Success path overhead | Zero (Itanium) | Zero (value return) | Small (discriminant check) |
| Error path overhead | ~1-10 us (unwinding) | ~10 ns (value return) | ~10 ns (value return) |
| Code size | +10-15% (.eh_frame) | Minimal | Minimal |
| Deterministic? | No | **Yes** | **Yes** |
| Propagation | Automatic (unwind) | Automatic (proposed) | Manual (check at each level) |
| Existing catch compatibility | Yes | Yes (proposed) | No |

### What Actually Happened: `std::expected<T,E>` (C++23)

P0709 was too ambitious for adoption - it required ABI changes, new keywords, and significant compiler rewrites. The committee could not agree on the details. Instead, they standardized `std::expected<T,E>`, which achieves the **deterministic** goal without changing the language syntax at all:

```cpp
#include <expected>
#include <string>

// The "practical P0709" - deterministic, value-based error handling
std::expected<double, std::string> safe_sqrt(double x) {
    if (x < 0) return std::unexpected("negative input");
    return std::sqrt(x);
}

// Monadic chaining (C++23) - close to the ergonomics Sutter wanted
auto result = safe_sqrt(16.0)
    .transform([](double v) { return v + 1.0; })
    .or_else([](const std::string& err) -> std::expected<double, std::string> {
        return 0.0;  // default on error
    });
```

The monadic operations (`.transform()`, `.and_then()`, `.or_else()`) are the C++23 way to chain operations without writing explicit `if (!result)` checks at every step. It is not as clean as `throws` would have been, but it is real, available code you can use today.

### Important Notes

- P0709 is a **proposal paper**, not a standard feature - no compiler implements it.
- `std::expected<T,E>` is the practical outcome adopted in C++23.
- The ergonomics gap (no automatic propagation like `throw`) remains - `std::expected` requires explicit checking at each call site.
- Rust's `Result<T,E>` with `?` operator is the closest real-world analog to what P0709 aimed for.

---

## Self-Assessment

### Q1: Explain why traditional exceptions have overhead in the success path on some ABI models

This is one of those things where "it depends on your ABI" is the real answer. The Itanium ABI gets true zero-cost on the happy path, but not all ABIs do.

**Detailed Explanation:**

On the **Itanium ABI** (used by GCC, Clang on Linux/macOS), the happy path truly has zero overhead - the compiler generates side tables, and normal execution never touches them:

```cpp
// Itanium ABI (zero-cost on happy path):
//
//   Normal code executes as if
//   no exceptions exist.
//   Side tables (.eh_frame) are
//   stored separately in read-only
//   memory, never accessed unless
//   an exception is thrown.
//
// Cost: 0 instructions on happy path
//       +10-15% binary size (tables)
```

However, some ABIs and scenarios **do** have happy-path overhead.

**1. SJLJ (SetJump/LongJump) ABI - legacy ARM, some older compilers:**

```cpp
// SJLJ ABI (non-zero happy path cost):
//
//   Every try block:
//     setjmp(buf);  // save state     <- ~20-50 ns per try block entry
//     if (returned_from_longjmp)
//       goto catch_handler;
//     normal_code();
//
//   Every throw:
//     longjmp(buf, 1);                <- deterministic ~100 ns
//
// Cost: setjmp called on EVERY try block entry - even when no exception thrown
```

The SJLJ overhead is actually predictable (unlike Itanium error-path overhead), which is why some embedded systems actually prefer it - you pay the same cost every time and can reason about timing.

**2. Windows SEH (Structured Exception Handling):**

```cpp
// Windows x64 SEH:
//
// - Uses table-based unwinding (similar to Itanium)
// - Happy path: near-zero overhead
// - But: exception filters and frame handlers have
//   registration overhead in some configurations
```

**3. Indirect costs on ALL ABIs:**

```cpp
// Even with zero-cost Itanium ABI:
void f() {
    // The presence of try/catch may inhibit optimizations:
    // - Compiler may not inline across try boundaries
    // - Register allocation may be constrained
    // - RAII destructors prevent tail-call optimization
    try {
        hot_loop();  // optimizer has less freedom
    } catch (...) {
        handle();
    }
}
```

This last point is subtle but important. Even when the happy-path instruction count is literally zero, the optimizer has to be conservative around `try` blocks. It cannot make transformations that would be incorrect if an exception were thrown - so some inlining, tail-call, and register-allocation opportunities are lost.

**Summary of ABI costs:**

| ABI | Happy Path | Error Path | Binary Size |
| --- | --- | --- | --- |
| Itanium (table-based) | ~0 ns | ~1-10 us | +10-15% |
| SJLJ (setjmp/longjmp) | ~20-50 ns per `try` | ~100-500 ns | +5% |
| Windows x64 SEH | ~0 ns | ~1-5 us | +5-10% |

---

### Q2: Describe the `std::expected` approach as a deterministic alternative

`std::expected<T,E>` is what you should actually use today. It delivers the determinism P0709 promised, at the cost of more verbose propagation code. Here is a complete example showing both the success and error paths, plus how monadic chaining works:

**`std::expected<T, E>` - The Deterministic Alternative:**

```cpp
#include <expected>
#include <string>
#include <iostream>
#include <cmath>
#include <fstream>
#include <sstream>
#include <system_error>

// Error type - lightweight, no dynamic allocation
enum class MathError {
    negative_input,
    overflow,
    division_by_zero
};

std::string to_string(MathError e) {
    switch (e) {
        case MathError::negative_input:   return "negative input";
        case MathError::overflow:         return "overflow";
        case MathError::division_by_zero: return "division by zero";
    }
    return "unknown";
}

// Deterministic: no stack unwinding, no RTTI, no tables
// Cost is IDENTICAL for success and failure - just a return value
std::expected<double, MathError> safe_sqrt(double x) {
    if (x < 0.0)
        return std::unexpected(MathError::negative_input);
    return std::sqrt(x);
}

std::expected<double, MathError> safe_divide(double a, double b) {
    if (b == 0.0)
        return std::unexpected(MathError::division_by_zero);
    return a / b;
}

// Composing with monadic operations (C++23)
std::expected<double, MathError> compute(double x, double y) {
    return safe_divide(x, y)
        .and_then([](double ratio) { return safe_sqrt(ratio); })
        .transform([](double root) { return root * 2.0; });
}

int main() {
    // Success path
    auto r1 = compute(16.0, 4.0);
    if (r1) std::cout << "Result: " << *r1 << "\n";  // 4.0

    // Error path - SAME performance cost as success
    auto r2 = compute(16.0, 0.0);
    if (!r2) std::cout << "Error: " << to_string(r2.error()) << "\n";

    auto r3 = compute(-4.0, 1.0);
    if (!r3) std::cout << "Error: " << to_string(r3.error()) << "\n";
}
// Expected output:
//   Result: 4
//   Error: division by zero
//   Error: negative input
```

**Why It's Deterministic:**

```cpp
// std::expected<double, MathError> internally:
//
//   union {
//     double value_;             // 8 bytes
//     MathError error_;          // 4 bytes
//   };
//   bool has_value_;             // 1 byte + padding
//
// Total: 16 bytes, returned in registers on x86-64
//
// Cost: always the same - write discriminant + value, return.
// No stack unwinding, no table lookup, no RTTI, no allocation.
```

The performance story is simple: success and failure both pay the same cost. You get deterministic behavior at both the function boundary and in the calling code.

---

### Q3: List the trade-offs: error ergonomics, zero-cost on success, and caller discipline

There is no free lunch here. Each approach makes different trade-offs. The table below shows them all, but the most important one is the ergonomics gap:

**Complete Trade-off Comparison:**

| Factor | Exceptions (`throw`) | `std::expected<T,E>` | Error Codes (`int`) |
| --- | --- | --- | --- |
| **Ergonomics** | Excellent - automatic propagation | Good - monadic chaining (C++23) | Poor - must check at every call |
| **Happy path cost** | Excellent - zero (Itanium) | Good - near-zero (discriminant) | Good - near-zero (branch) |
| **Error path cost** | Poor - ~1-10 us (unwinding) | Excellent - ~10 ns (return value) | Excellent - ~10 ns (return value) |
| **Determinism** | Poor - non-deterministic | Excellent - fully deterministic | Excellent - fully deterministic |
| **Binary size** | Poor - +10-15% tables | Excellent - minimal | Excellent - minimal |
| **Caller discipline** | Excellent - no discipline needed | Good - must check `.has_value()` | Poor - easy to ignore return value |
| **Composability** | Good - try/catch blocks | Good - monadic ops | Poor - manual at every level |
| **Type safety** | Good - any type throwable | Excellent - typed error `E` | Poor - magic numbers |
| **Embedded/RT** | Not applicable - often disabled | Works everywhere | Works everywhere |
| **Cross-language** | Not applicable - C++ only | C++ only (but trivial to convert) | C-compatible |

**The Ergonomics Gap (The Biggest Trade-off):**

This is where `std::expected` feels most painful compared to traditional exceptions. With exceptions, errors propagate automatically through as many layers as needed. With `std::expected`, every intermediate layer must explicitly forward the error:

```cpp
// Exceptions: error propagates automatically through N layers
void layer_3() { might_throw(); }
void layer_2() { layer_3(); }      // no error handling code needed
void layer_1() { layer_2(); }      // no error handling code needed
void caller() {
    try { layer_1(); }
    catch (const Error& e) { handle(e); }  // caught here
}

// std::expected: EVERY layer must explicitly propagate
auto layer_3() -> std::expected<int, Error> { return might_fail(); }
auto layer_2() -> std::expected<int, Error> {
    auto r = layer_3();
    if (!r) return std::unexpected(r.error());  // must propagate!
    return *r + 1;
}
// This is the price of determinism - verbose but predictable.
// Rust solves this with the ? operator; C++ has no equivalent (yet).
```

The reason this verbosity exists is that `std::expected` does not change the language. Every function must declare its return type and explicitly return error values. Rust solves this with the `?` operator, which expands to exactly the check-and-early-return code you see above. C++ may eventually get something similar, but for now you write it by hand or use monadic chains.

**Recommendation by Use Case:**

| Use Case | Recommended Mechanism |
| --- | --- |
| Embedded / real-time | `std::expected` or error codes |
| `-fno-exceptions` codebase | `std::expected` or error codes |
| Library boundary (rare errors) | Exceptions (easy for callers) |
| Common/expected failures | `std::expected` |
| Performance-critical hot path | `std::expected` (deterministic) |
| Legacy C interop | Error codes |
| New C++23 codebase | `std::expected` with monadic ops |

---

## Notes

- **P0709 status:** Not adopted; Sutter presented at CppCon 2018/2019. Ideas influenced `std::expected` and `std::error` proposals.
- **Boost.LEAF:** A lightweight error-handling library by Emil Dotchevski that implements deterministic exception-like semantics without dynamic allocation. Worth investigating as an alternative.
- **Rust's `Result<T,E>` + `?`:** The closest real-world implementation of what P0709 tried to achieve - deterministic, ergonomic, and zero-cost.
- **`std::expected` in practice:** Use `std::error_code` as the `E` type for system-level errors, or define your own `enum class` for domain errors.
- **Future directions:** C++ may eventually get a `try` expression or propagation operator (like Rust's `?`) to close the ergonomics gap.
