# Interoperate with Rust via a C ABI using the cxx crate (Ownership and Error Handling)

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://cxx.rs/>  

---

## Topic Overview

This part focuses on **ownership semantics** across the Rust/C++ boundary and **error handling** - how cxx maps C++ exceptions to Rust `Result` and vice versa. These are the two areas where most cxx-related bugs originate, because both languages have strong opinions about who owns what and what happens when something goes wrong.

### Ownership Model

The ownership picture becomes clearer once you accept that each language manages its own heap. Rust-allocated objects are represented as `Box<T>` on the Rust side and `rust::Box<T>` on the C++ side. C++-allocated objects are `std::unique_ptr<T>` on the C++ side and `cxx::UniquePtr<T>` on the Rust side. The destructor always runs on the side that allocated:

```cpp
Rust-owned:                     C++-owned:
┌───────────────┐               ┌───────────────┐
│ Box<T>        │               │ UniquePtr<T>   │
│ (Rust heap)   │-> C++ sees    │ (C++ heap)    │-> Rust sees
│ Drop on Rust  │   rust::Box   │ ~T() on C++   │   cxx::UniquePtr
│ side          │   <T>         │ side          │   <T>
└───────────────┘               └───────────────┘

Shared references:
  Rust &T      -->  C++ sees const T&   (borrowing, no ownership transfer)
  Rust &mut T  -->  C++ sees T&         (exclusive mutable access)
```

### Error Handling Across the Boundary

The general rule is: Rust `Result::Err` becomes a C++ exception, and a C++ exception becomes a Rust `Result::Err`. But only when you declare the function correctly. If you forget to declare `Result<>` on a C++ function that can throw, an uncaught exception will call `std::terminate` and abort the process:

| Rust Side | Bridge Action | C++ Side |
| --- | :---: | --- |
| `fn foo() -> Result<T>` | `Err` -> thrown | `try { foo(); } catch(rust::Error&)` |
| `fn bar()` (no Result) | Panic -> `std::terminate` | Process aborts |
| - | C++ throws | `Result<T>` in Rust (if declared) |

---

## Self-Assessment

### Q1: Define a CXX bridge that exposes a C++ function to Rust and a Rust function to C++

**Answer:**

Here's a more realistic bidirectional bridge - a Rust database pool exposed to C++, and a C++ logger exposed to Rust. Notice that the C++ functions are declared in an `unsafe extern "C++"` block because cxx can't statically verify their implementation is safe:

```rust
// src/lib.rs - bidirectional bridge
#[cxx::bridge(namespace = "myapp")]
mod ffi {
    // Shared struct - usable on both sides
    #[derive(Debug, Clone)]
    struct Config {
        max_connections: u32,
        timeout_ms: u64,
        host: String,
    }

    // Rust functions callable from C++
    extern "Rust" {
        type DatabasePool;
        fn create_pool(config: &Config) -> Box<DatabasePool>;
        fn query(pool: &DatabasePool, sql: &str) -> Result<Vec<String>>;
        fn pool_size(pool: &DatabasePool) -> usize;
    }

    // C++ functions callable from Rust
    unsafe extern "C++" {
        include!("myapp/logger.h");

        type Logger;
        fn create_logger(path: &CxxString) -> UniquePtr<Logger>;
        fn log(logger: &Logger, level: i32, msg: &str);
        fn flush(logger: &Logger) -> Result<()>;  // C++ may throw
    }
}

pub struct DatabasePool {
    connections: Vec<String>,
    max: u32,
}

fn create_pool(config: &ffi::Config) -> Box<DatabasePool> {
    Box::new(DatabasePool {
        connections: Vec::with_capacity(config.max_connections as usize),
        max: config.max_connections,
    })
}

fn query(pool: &DatabasePool, sql: &str) -> Result<Vec<String>, cxx::Exception> {
    if sql.is_empty() {
        return Err(cxx::Exception::msg("Empty query"));
    }
    Ok(vec![format!("Result for: {}", sql)])
}

fn pool_size(pool: &DatabasePool) -> usize {
    pool.connections.len()
}
```

The C++ `Logger` class can throw from its `flush` method. Because `flush` is declared as `Result<()>` in the bridge, that exception will be caught by cxx and delivered to Rust as an `Err` value rather than crashing the process:

```cpp
// logger.h
#pragma once
#include "rust/cxx.h"
#include <string>
#include <fstream>
#include <memory>
#include <stdexcept>

namespace myapp {

class Logger {
    std::ofstream file_;
public:
    explicit Logger(const std::string& path) : file_(path) {
        if (!file_) throw std::runtime_error("Cannot open log file");
    }

    void log(int32_t level, rust::Str msg) const {
        // rust::Str -> std::string_view-like (no copy for read)
        file_ << "[" << level << "] " << std::string(msg) << "\n";
    }

    void flush() const {
        if (!file_) throw std::runtime_error("File not open");
        // This exception -> Rust Result::Err
    }
};

std::unique_ptr<Logger> create_logger(const std::string& path) {
    return std::make_unique<Logger>(path);
}

}  // namespace myapp
```

### Q2: Explain how cxx handles ownership: `Box<T>` (Rust owned) vs `UniquePtr<T>` (C++ owned)

**Answer:**

The mental model is straightforward: you always free memory on the side that allocated it. cxx enforces this through its type system rather than leaving it to convention. Here's the bridge declaration that makes the ownership contracts visible:

```rust
#[cxx::bridge]
mod ffi {
    extern "Rust" {
        type ImageProcessor;

        // Returns Box<T> - Rust allocates, Rust owns
        fn create_processor(width: u32, height: u32) -> Box<ImageProcessor>;

        // Takes &T - borrows, no ownership transfer
        fn process(proc: &ImageProcessor, data: &[u8]) -> Vec<u8>;

        // Takes Box<T> - ownership transferred TO this function (consumed)
        fn destroy_processor(proc: Box<ImageProcessor>);
    }

    unsafe extern "C++" {
        include!("myapp/renderer.h");
        type Renderer;

        // Returns UniquePtr<T> - C++ allocates, C++ owns
        fn create_renderer() -> UniquePtr<Renderer>;

        // Takes &T - borrows the C++ object
        fn render(renderer: &Renderer, width: u32, height: u32);

        // Takes Pin<&mut T> - mutable borrow of C++ object
        fn set_viewport(renderer: Pin<&mut Renderer>, w: u32, h: u32);
    }
}
```

Here's the full ownership table for reference, because this is easy to get confused:

**Ownership rules:**

| Operation | Rust Side | C++ Side | Who Frees? |
| --- | --- | --- | --- |
| `Box<T>` returned | Created with `Box::new` | Sees `rust::Box<T>` | Rust (when Box dropped) |
| `UniquePtr<T>` returned | Sees `cxx::UniquePtr<T>` | Created with `make_unique` | C++ (when unique_ptr destroyed) |
| `&T` passed | Borrow (lifetime checked) | `const T&` | Nobody (no ownership) |
| `&mut T` passed | Exclusive mutable borrow | `T&` | Nobody (no ownership) |
| `Pin<&mut T>` | Pinned mutable reference | `T&` | Nobody (prevents move) |
| `Vec<T>` returned | Owned Rust vec | `rust::Vec<T>` (can iterate) | Rust (when Vec dropped) |

On the C++ side, working with a Rust-owned `Box` looks like this - it's automatically freed when it goes out of scope, which calls Rust's `Drop`:

```cpp
// C++ side - working with Rust-owned objects
void demo() {
    // Rust allocates, returns Box
    rust::Box<ImageProcessor> proc = create_processor(1920, 1080);

    // Borrow - no ownership transfer
    std::vector<uint8_t> input = load_image();
    rust::Vec<uint8_t> output = process(*proc, {input.data(), input.size()});

    // proc automatically freed when it goes out of scope
    // (calls Rust's Drop)
}
```

### Q3: Show error handling across the boundary: C++ exceptions become Rust Results

**Answer:**

Error handling is the area where forgetting a `Result<>` declaration can crash your process, so it's worth going through carefully. When a Rust function returns `Result::Err`, cxx translates it to a C++ exception of type `rust::Error`. When a C++ function throws and it's declared as `Result<>`, cxx catches the exception and delivers it as a Rust `Err`. Here's both directions in full:

```rust
#[cxx::bridge]
mod ffi {
    extern "Rust" {
        // Rust -> C++: Result becomes catchable exception
        fn parse_config(json: &str) -> Result<String>;
        fn divide(a: f64, b: f64) -> Result<f64>;
    }

    unsafe extern "C++" {
        include!("myapp/io.h");
        // C++ -> Rust: declared as Result = exceptions caught
        fn read_file(path: &CxxString) -> Result<Vec<u8>>;
        fn write_file(path: &CxxString, data: &[u8]) -> Result<()>;
    }
}

fn parse_config(json: &str) -> Result<String, cxx::Exception> {
    if json.is_empty() {
        return Err(cxx::Exception::msg("Empty JSON input"));
        // In C++: throws rust::Error("Empty JSON input")
    }
    Ok(format!("parsed: {}", json))
}

fn divide(a: f64, b: f64) -> Result<f64, cxx::Exception> {
    if b == 0.0 {
        return Err(cxx::Exception::msg("Division by zero"));
    }
    Ok(a / b)
}
```

From the C++ side, Rust errors look like regular C++ exceptions - you catch them as `rust::Error` and call `.what()` to get the message:

```cpp
// C++ calling Rust functions with error handling
#include "myapp/src/lib.rs.h"
#include <iostream>

void demo() {
    // Rust Result::Ok -> returns value
    try {
        rust::String result = parse_config("{}");
        std::cout << std::string(result) << "\n";
    } catch (const rust::Error& e) {
        // Rust Result::Err -> caught as rust::Error
        std::cerr << "Parse error: " << e.what() << "\n";
    }

    // Rust Result::Err -> exception
    try {
        double result = divide(10.0, 0.0);
    } catch (const rust::Error& e) {
        std::cerr << e.what() << "\n";  // "Division by zero"
    }
}
```

From the Rust side, C++ exceptions on `Result<>`-declared functions arrive as `Err` values. The warning at the end is critical: if a C++ function is NOT declared as `Result<>` and it throws, the exception is unhandled and the process aborts via `std::terminate`:

```rust
// Rust calling C++ functions with error handling
fn demo() {
    let path = std::pin::Pin::new(&cxx::CxxString::new("config.txt"));

    // C++ exception -> Rust Result::Err
    match ffi::read_file(&path) {
        Ok(data) => println!("Read {} bytes", data.len()),
        Err(e) => eprintln!("C++ threw: {}", e),
        // e.what() returns the exception message
    }

    // If C++ function NOT declared as Result and it throws:
    // -> std::terminate() - program aborts!
    // ALWAYS declare Result<> for C++ functions that may throw
}
```

---

## Notes

- `Box<T>` crosses the boundary by transferring ownership - Rust's Drop runs when C++ releases it.
- `UniquePtr<T>` crossing to Rust - C++ destructor runs when Rust drops the UniquePtr wrapper.
- `SharedPtr<T>` is also supported for shared ownership across the boundary.
- Never declare a C++ function without `Result<>` if it can throw - unhandled exceptions cause an abort.
- cxx doesn't support passing raw pointers - forcing safe ownership patterns by design.
- Build integration: add `cxx-build` crate to `build-dependencies` in Cargo.toml.
