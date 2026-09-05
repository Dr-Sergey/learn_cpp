# Interoperate with Rust via a C ABI boundary using the cxx crate

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://cxx.rs/>  

---

## Topic Overview

The **cxx** crate provides **safe** Rust/C++ interop by generating type-safe FFI bindings from a `#[cxx::bridge]` declaration. Instead of writing raw `extern "C"` functions with `unsafe`, cxx generates glue code that preserves type safety on both sides. The key idea is that you declare the interface once in a `#[cxx::bridge]` block and cxx generates both the C++ header and the Rust bindings from it - so both sides are always in sync.

### Architecture

The bridge macro acts as a shared contract between the two languages. Each side declares what the other side provides, and cxx generates the necessary glue:

```cpp
┌──────────────────────────────────────────────────────┐
│               cxx::bridge macro                        │
│  ┌────────────────────┐  ┌─────────────────────────┐ │
│  │ extern "Rust" {    │  │ extern "C++" {          │ │
│  │   fn rust_func()   │  │   include!("lib.h")     │ │
│  │   type RustType    │  │   fn cpp_func()         │ │
│  │ }                  │  │   type CppType          │ │
│  └────────────────────┘  └─────────────────────────┘ │
│            │                          │               │
│            ▼                          ▼               │
│  Generated C++ header        Generated Rust bindings  │
│  (callable from C++)         (callable from Rust)     │
└──────────────────────────────────────────────────────┘
```

### cxx Type Mappings

cxx maintains its own set of bridge types that sit between Rust's standard types and C++'s standard types. Some of these are zero-copy borrows, others are owned conversions - the "Notes" column tells you which:

| Rust Type | C++ Type | Notes |
| --- | --- | --- |
| `String` | `rust::String` | UTF-8, owned |
| `&str` | `rust::Str` | Borrowed slice |
| `&[T]` | `rust::Slice<const T>` | Borrowed contiguous |
| `Vec<T>` | `rust::Vec<T>` | Owned dynamic array |
| `Box<T>` | `rust::Box<T>` | Rust heap-owned |
| `UniquePtr<T>` | `std::unique_ptr<T>` | C++ heap-owned |
| `CxxString` | `std::string` | C++ string |
| `CxxVector<T>` | `std::vector<T>` | C++ vector |

---

## Self-Assessment

### Q1: Define a shared header with cxx::bridge and call a Rust function from C++ via FFI

**Answer:**

The bridge declaration lives in Rust and is the single source of truth for both sides. `extern "Rust"` blocks declare functions implemented in Rust that C++ can call; `extern "C++"` blocks declare C++ functions that Rust can call. Here's a complete bidirectional example:

**Rust side (src/lib.rs):**

```rust
// src/lib.rs
#[cxx::bridge]
mod ffi {
    // Shared types - usable on both sides
    struct Point {
        x: f64,
        y: f64,
    }

    // Functions implemented in Rust, callable from C++
    extern "Rust" {
        fn distance(a: &Point, b: &Point) -> f64;
        fn greet(name: &str) -> String;
        fn fibonacci(n: u32) -> u64;
    }

    // Functions implemented in C++, callable from Rust
    extern "C++" {
        include!("myproject/cpp_funcs.h");
        fn log_message(level: i32, msg: &CxxString);
    }
}

fn distance(a: &ffi::Point, b: &ffi::Point) -> f64 {
    ((a.x - b.x).powi(2) + (a.y - b.y).powi(2)).sqrt()
}

fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

fn fibonacci(n: u32) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => {
            let (mut a, mut b) = (0u64, 1u64);
            for _ in 2..=n { let t = a + b; a = b; b = t; }
            b
        }
    }
}
```

The C++ side includes the cxx-generated header and the header for the C++ functions it implements. From C++, calling into Rust feels like calling any other function:

**C++ side (cpp_funcs.h / cpp_funcs.cpp):**

```cpp
// cpp_funcs.h
#pragma once
#include "rust/cxx.h"  // cxx-generated header

void log_message(int32_t level, const rust::String& msg);
```

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// cpp_funcs.cpp
#include "cpp_funcs.h"
#include "myproject/src/lib.rs.h"  // cxx-generated Rust bindings
#include <cstdio>

void log_message(int32_t level, const rust::String& msg) {
    printf("[%d] %s\n", level, std::string(msg).c_str());
}

int main() {
    // Call Rust functions from C++!
    auto p1 = Point{0.0, 0.0};
    auto p2 = Point{3.0, 4.0};
    printf("Distance: %f\n", distance(p1, p2));  // 5.0

    auto greeting = greet("C++");
    printf("%s\n", std::string(greeting).c_str());  // "Hello, C++!"

    printf("fib(20) = %llu\n", fibonacci(20));  // 6765
    return 0;
}
```

### Q2: Pass a std::string across the C++/Rust boundary using cxx's safe string bridge

**Answer:**

String passing is probably the most common thing you'll do across this boundary, and cxx has specific types for each direction. The trick is knowing which type to use depending on whether you're borrowing or transferring ownership, and whether the string originates in Rust or C++:

```rust
// src/lib.rs
#[cxx::bridge]
mod ffi {
    extern "Rust" {
        // Takes C++ string by reference, returns Rust String
        fn process_name(input: &CxxString) -> String;
        // Takes Rust str slice
        fn validate_email(email: &str) -> bool;
    }

    extern "C++" {
        include!("myproject/strings.h");
        // C++ function receives Rust string
        fn format_output(prefix: &str, value: &CxxString) -> UniquePtr<CxxString>;
    }
}

fn process_name(input: &CxxString) -> String {
    // CxxString -> Rust &str (zero-copy borrow)
    let s: &str = input.to_str().unwrap_or("invalid-utf8");
    // Transform and return as Rust String
    s.trim().to_uppercase()
}

fn validate_email(email: &str) -> bool {
    email.contains('@') && email.contains('.')
}
```

```cpp
// strings.h + strings.cpp
#include "rust/cxx.h"
#include <string>
#include <memory>

// C++ receives rust::Str (borrowed Rust string slice)
std::unique_ptr<std::string> format_output(
    rust::Str prefix,        // Borrowed from Rust - no copy
    const std::string& value // Regular C++ string
) {
    // rust::Str -> std::string conversion (copies)
    std::string result = std::string(prefix) + ": " + value;
    return std::make_unique<std::string>(result);
}

// Usage from C++:
void demo() {
    std::string name = "  john doe  ";

    // C++ std::string -> Rust: automatic via &CxxString
    rust::String processed = process_name(name);
    // processed = "JOHN DOE"

    // Rust &str -> C++: automatic via rust::Str
    bool valid = validate_email("user@example.com");
    // valid = true
}
```

Here's a cheat sheet for the string crossing directions:

**String type cheat sheet:**

| Direction | Source Type | Bridge Type | Copy? |
| --- | --- | --- | --- |
| C++ -> Rust | `const std::string&` | `&CxxString` | No (borrow) |
| C++ -> Rust | `std::string` (owned) | `CxxString` | Moved |
| Rust -> C++ | `&str` | `rust::Str` | No (borrow) |
| Rust -> C++ | `String` | `rust::String` | Moved |

### Q3: Explain why direct C++ to Rust interop is unsafe and why the C ABI bridge is safer

**Answer:**

The reason you can't just call C++ code directly from Rust is that the two languages have completely different ideas about how code works at the binary level. It's not just naming - it's exceptions vs panics, vtable layout, memory aliasing rules, and the fact that `std::string` and Rust's `String` have entirely different internal representations. A direct call without a defined boundary would mean undefined behavior in several of these areas simultaneously.

**Why direct interop is unsafe:**

| Problem | Details |
| --- | --- |
| **No shared ABI** | C++ and Rust have different name mangling, vtable layouts, exception mechanisms |
| **Memory model mismatch** | C++ allows aliasing; Rust's borrow checker forbids it |
| **Exceptions vs panics** | C++ exception unwinding through Rust frames = undefined behavior |
| **std::string vs String** | Different allocation strategies, size fields, SSO |
| **Inheritance** | C++ has virtual inheritance; Rust has no inheritance |
| **Templates vs generics** | C++ templates monomorphize differently than Rust generics |

The solution is to use C as the stable middle ground. C has a well-defined, minimal ABI that both languages know how to speak. POD types and simple function signatures travel across that boundary safely:

**Why C ABI is the safe bridge:**

```cpp
Direct C++<->Rust:  <- UNSAFE
┌─────────┐          ┌──────┐
│  C++    │ <--?-->  │ Rust │
│ vtables │          │ traits│
│ RTTI    │          │ enums │
│ std::   │          │ std:: │
└─────────┘          └──────┘
  Different ABIs, different layouts, UB

C ABI bridge:  <- SAFE
┌─────────┐    ┌──────┐    ┌──────┐
│  C++    │<-->│ C ABI│<-->│ Rust │
│ std::   │    │ POD  │    │ repr │
│ objects │    │ types│    │  (C) │
└─────────┘    └──────┘    └──────┘
  C ABI is stable, well-defined, and minimal
```

What cxx adds on top of raw `extern "C"` is compile-time safety checking and ownership tracking - you get the stability of the C ABI without having to write unsafe Rust by hand:

**What cxx does better than raw `extern "C"`:**

1. **Compile-time safety**: cxx generates bindings that the Rust AND C++ compilers both check.
2. **No unsafe Rust**: function signatures in `extern "Rust"` don't require `unsafe` blocks.
3. **Ownership tracking**: `Box<T>` and `UniquePtr<T>` make ownership explicit and enforceable.
4. **String safety**: cxx validates UTF-8 when converting `CxxString` to Rust `&str`.
5. **Error handling**: Rust `Result` maps to C++ exceptions (caught, not UB).

---

## Notes

- cxx generates header files at build time via `build.rs` - add `cxx-build` to build-dependencies.
- Shared structs must be `#[repr(C)]` compatible (no enums with data, no generics).
- `UniquePtr<T>` is the primary way to pass C++ objects to Rust by value.
- `Box<T>` is the primary way to pass Rust objects to C++ by value.
- cxx does NOT support raw pointers - forcing safe patterns by design.
- For async: use the `cxx-async` crate to bridge C++ coroutines with Rust futures.
