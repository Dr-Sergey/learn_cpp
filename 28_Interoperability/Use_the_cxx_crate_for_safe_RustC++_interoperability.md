# Use the cxx crate for safe Rust/C++ interoperability

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://cxx.rs>  

---

## Topic Overview

The **cxx** crate provides a **safe, zero-cost** FFI bridge between Rust and C++. Unlike raw `extern "C"` FFI where you manually manage pointers, layouts, and lifetimes, cxx generates both the C++ and Rust sides of the bridge from a single `#[cxx::bridge]` declaration, with compile-time safety checks on both sides.

The reason this matters is that raw FFI is one of the most dangerous things you can do in Rust - it takes a language that eliminates entire classes of bugs and asks you to manually handle all the memory, null, and lifetime bookkeeping that Rust normally does for you. cxx exists to restore those safety guarantees while still calling across the Rust/C++ boundary.

### How cxx Works

The central idea is that you write one bridge declaration in Rust, and cxx generates the glue code for both sides at build time. You never write the interop boilerplate by hand.

```cpp
     Rust side                    C++ side
  ┌──────────────┐           ┌──────────────┐
  │ #[cxx::bridge]│           │  Generated   │
  │ mod ffi {    │ ────────► │  .h + .cc    │
  │   extern "C++│           │  (by build.rs│
  │   extern "Rust           │   + cxx-build│
  │ }            │           │              │
  └──────────────┘           └──────────────┘
         │ cxx verifies:              │
         │ - Type compatibility       │
         │ - No raw pointer misuse    │
         │ - Lifetime correctness     │
         └────────────────────────────┘
```

### cxx Type Mapping Table

Types do not cross the boundary as-is - cxx has specific wrapper types for each direction. This table is worth memorizing because the type you choose on the Rust side dictates the C++ type on the other end, and vice versa.

| Rust type | C++ type | Behavior |
| --- | --- | --- |
| `String` | `rust::String` | Owned UTF-8 string |
| `&str` | `rust::Str` | Borrowed UTF-8 view |
| `CxxString` | `std::string` | C++ owned string |
| `&CxxString` | `const std::string&` | Borrowed C++ string |
| `Vec<T>` | `rust::Vec<T>` | Owned Rust vector |
| `&[T]` | `rust::Slice<const T>` | Borrowed slice |
| `Box<T>` | `rust::Box<T>` | Owned heap allocation |
| `UniquePtr<T>` | `std::unique_ptr<T>` | C++ unique ownership |
| `SharedPtr<T>` | `std::shared_ptr<T>` | Shared ownership |
| `i32`, `f64`, etc. | `int32_t`, `double` | Direct mapping |

---

## Self-Assessment

### Q1: Define a C++/Rust bridge using cxx::bridge and call a Rust function from C++

**Answer:**

Here is a complete working example of a two-way bridge. The `build.rs` script is what actually generates the glue code - `cxx_build::bridge` reads the bridge declaration in `lib.rs` and emits the C++ header and source that both sides need.

Start with the `Cargo.toml` to understand the dependencies:

```toml
# Cargo.toml
[package]
name = "audio-engine"
version = "0.1.0"
edition = "2021"

[dependencies]
cxx = "1.0"

[build-dependencies]
cxx-build = "1.0"
```

The bridge declaration is the heart of the design. `extern "Rust"` lists Rust functions that C++ can call; `unsafe extern "C++"` lists C++ functions that Rust can call. cxx verifies that the types on both sides are compatible.

```rust
// src/lib.rs - Rust side
use cxx::CxxString;

#[cxx::bridge]
mod ffi {
    // Rust functions callable from C++
    extern "Rust" {
        fn compress_audio(samples: &[f32], ratio: f32) -> Vec<f32>;
        fn compute_rms(samples: &[f32]) -> f32;
    }

    // C++ functions callable from Rust
    unsafe extern "C++" {
        include!("audio-engine/include/mixer.h");

        type AudioMixer;

        fn new_mixer(sample_rate: u32, channels: u32) -> UniquePtr<AudioMixer>;
        fn mix_tracks(mixer: &AudioMixer, track_a: &[f32], track_b: &[f32]) -> Vec<f32>;
        fn get_sample_rate(mixer: &AudioMixer) -> u32;
    }
}

// Implement the Rust functions declared in the bridge
fn compress_audio(samples: &[f32], ratio: f32) -> Vec<f32> {
    samples.iter()
        .map(|&s| {
            let sign = s.signum();
            let abs = s.abs();
            sign * abs.powf(1.0 / ratio)
        })
        .collect()
}

fn compute_rms(samples: &[f32]) -> f32 {
    let sum: f32 = samples.iter().map(|s| s * s).sum();
    (sum / samples.len() as f32).sqrt()
}
```

The C++ header and implementation are conventional C++, with the only difference being that they use `rust::Vec<float>` and `rust::Slice<const float>` where they cross the boundary.

```cpp
// include/mixer.h - C++ header
#pragma once
#include <cstdint>
#include <memory>
#include "rust/cxx.h"

class AudioMixer {
    uint32_t sample_rate_;
    uint32_t channels_;

public:
    AudioMixer(uint32_t sample_rate, uint32_t channels)
        : sample_rate_(sample_rate), channels_(channels) {}

    uint32_t get_sample_rate() const { return sample_rate_; }
};

std::unique_ptr<AudioMixer> new_mixer(uint32_t sample_rate, uint32_t channels);
rust::Vec<float> mix_tracks(const AudioMixer& mixer,
                            rust::Slice<const float> track_a,
                            rust::Slice<const float> track_b);
uint32_t get_sample_rate(const AudioMixer& mixer);
```

```cpp
// src/mixer.cc - C++ implementation
#include "audio-engine/include/mixer.h"
#include <algorithm>

std::unique_ptr<AudioMixer> new_mixer(uint32_t sample_rate, uint32_t channels) {
    return std::make_unique<AudioMixer>(sample_rate, channels);
}

rust::Vec<float> mix_tracks(const AudioMixer& mixer,
                            rust::Slice<const float> track_a,
                            rust::Slice<const float> track_b) {
    rust::Vec<float> result;
    size_t len = std::min(track_a.size(), track_b.size());
    result.reserve(len);
    for (size_t i = 0; i < len; ++i) {
        result.push_back((track_a[i] + track_b[i]) * 0.5f);
    }
    return result;
}

uint32_t get_sample_rate(const AudioMixer& mixer) {
    return mixer.get_sample_rate();
}
```

The build script ties everything together. It reads the bridge declaration, compiles the C++ file, and links both halves.

```rust
// build.rs - Build script
fn main() {
    cxx_build::bridge("src/lib.rs")
        .file("src/mixer.cc")
        .flag_if_supported("-std=c++17")
        .compile("audio-engine");

    println!("cargo:rerun-if-changed=src/lib.rs");
    println!("cargo:rerun-if-changed=src/mixer.cc");
    println!("cargo:rerun-if-changed=include/mixer.h");
}
```

### Q2: Pass std::string to Rust and rust::String to C++ using cxx type mappings

**Answer:**

String passing is the area where cxx's type system does the most visible work. The reason there are so many string types is that Rust `String` is always valid UTF-8, while `std::string` is not - cxx enforces that validation at the boundary rather than letting garbage through silently.

The cheat sheet: use `&str` when Rust is lending a string view to C++ (zero-copy, must be UTF-8). Use `&CxxString` when C++ is lending a string to Rust (also zero-copy, validated at access). Use owned types (`String`, `CxxString`) when ownership needs to transfer.

```rust
// src/lib.rs - String passing patterns
#[cxx::bridge]
mod ffi {
    extern "Rust" {
        // C++ calls this, passing a C++ string -> Rust receives &CxxString
        fn process_name(name: &CxxString) -> String;

        // Rust returns String -> C++ receives rust::String
        fn generate_greeting(first: &CxxString, last: &CxxString) -> String;
    }

    unsafe extern "C++" {
        include!("mylib/include/logger.h");

        type Logger;
        fn create_logger(name: &str) -> UniquePtr<Logger>;

        // Rust passes &str -> C++ receives rust::Str
        fn log_message(logger: &Logger, msg: &str);

        // C++ returns std::string -> Rust receives String
        fn get_log_path(logger: &Logger) -> String;
    }
}

fn process_name(name: &cxx::CxxString) -> String {
    // CxxString -> Rust String: must handle potential non-UTF-8
    let rust_str = name.to_str().unwrap_or("invalid-utf8");
    format!("Processed: {}", rust_str.to_uppercase())
}

fn generate_greeting(first: &cxx::CxxString, last: &cxx::CxxString) -> String {
    format!("Hello, {} {}!",
        first.to_str().unwrap_or("?"),
        last.to_str().unwrap_or("?"))
}
```

On the C++ side, `rust::Str` behaves like a non-owning string view (similar to `std::string_view`), and converting it to `std::string` requires an explicit copy.

```cpp
// include/logger.h - C++ side string handling
#pragma once
#include <string>
#include <memory>
#include "rust/cxx.h"

class Logger {
    std::string name_;
    std::string log_path_;

public:
    explicit Logger(rust::Str name)
        : name_(std::string(name)),         // rust::Str -> std::string
          log_path_("/var/log/" + name_) {}

    void log_message(rust::Str msg) const {
        // rust::Str provides .data() and .size()
        std::string message(msg.data(), msg.size());
        // ... write to file
    }

    // Return std::string -> cxx converts to rust::String
    rust::String get_log_path() const {
        return rust::String(log_path_.data(), log_path_.size());
    }
};

std::unique_ptr<Logger> create_logger(rust::Str name);
```

Here is a quick reference for which string type to use in each situation:

```cpp
Direction                  Type on boundary        Notes
─────────────────────────────────────────────────────────────
Rust -> C++ (borrowed)    &str -> rust::Str         Zero-copy, must be UTF-8
Rust -> C++ (owned)       String -> rust::String    Moved, heap allocated
C++ -> Rust (borrowed)    const std::string& ->     Must call .to_str()
                         &CxxString               (may fail if non-UTF-8)
C++ -> Rust (owned)       std::string -> String     Copied & validated UTF-8
```

### Q3: Explain the safety guarantees cxx provides compared to raw extern "C" FFI

**Answer:**

This is the core of why cxx exists. Raw `extern "C"` FFI forces you into a world of manual everything - null checks, lifetime tracking, memory deallocation, UTF-8 validation, double-free avoidance. The compiler cannot help you because it does not know anything about the invariants of the C++ code on the other side of the boundary.

cxx restores most of Rust's safety story even across the language boundary, by having the compiler verify the bridge declaration and generate checked wrapper types on both sides.

| Safety aspect | Raw extern "C" FFI | cxx crate |
| --- | --- | --- |
| **Type checking** | Manual: you declare types on both sides | **Verified**: cxx codegen ensures both sides agree |
| **Memory safety** | None: raw pointers everywhere | **Enforced**: `UniquePtr`, `Box`, `SharedPtr` |
| **String encoding** | No validation | **UTF-8 checked** at boundary |
| **Null pointers** | Silent UB on deref | **Compile error**: references can't be null |
| **Exception safety** | C++ exceptions cross FFI = UB | **Caught**: C++ exceptions -> Rust Result |
| **ABI mismatch** | Silent corruption | **Compile-time error** |
| **Data races** | No protection | Rust's `Send`/`Sync` enforced |

Here is the contrast side by side. The raw FFI version is a minefield:

```rust
// RAW FFI - Unsafe, error-prone
extern "C" {
    fn create_widget() -> *mut Widget;      // Raw pointer: could be null
    fn widget_name(w: *const Widget) -> *const c_char;  // Dangling?
    fn destroy_widget(w: *mut Widget);      // Double-free?
}

// Caller must manually:
unsafe {
    let w = create_widget();        // Might be null!
    if w.is_null() { panic!(); }    // Manual null check
    let name = widget_name(w);      // Might dangle!
    let s = CStr::from_ptr(name);   // Might not be UTF-8!
    destroy_widget(w);              // Must not forget!
    // destroy_widget(w);           // Double free - UB!
}

// CXX - Safe, checked
#[cxx::bridge]
mod ffi {
    unsafe extern "C++" {
        type Widget;
        fn create_widget() -> UniquePtr<Widget>;  // Never null (or compile error)
        fn widget_name(w: &Widget) -> &CxxString; // Borrow checked
        // No destroy needed - UniquePtr drops automatically
    }
}

fn use_widget() {
    let w = ffi::create_widget();           // UniquePtr - auto-dropped
    let name = ffi::widget_name(&w);        // Borrow - lifetime tied to w
    println!("{}", name.to_str().unwrap());  // UTF-8 validated
    // w dropped here - destructor called automatically
}
```

Here is a list of what cxx catches before your code even links:

```cpp
// Passing a Rust String where C++ expects std::string
//   -> compile error: use &CxxString or convert explicitly
// Returning a dangling reference from C++
//   -> compile error: lifetime not satisfiable
// Forgetting to free C++ objects
//   -> UniquePtr/SharedPtr drop automatically
// Calling a C++ function that doesn't exist
//   -> link error caught by cxx-build
// Struct layout mismatch
//   -> cxx generates layout assertions at compile time
```

---

## Notes

- cxx is **not** a general FFI replacement - it covers the safe subset of C++/Rust interop and intentionally leaves out raw pointers, `void*`, and function pointers.
- For opaque types that cannot be passed by value, use `UniquePtr<T>` or `Box<T>`.
- Shared structs (types you want to pass by value across the boundary) must be declared inside `#[cxx::bridge]` with all their fields explicitly listed.
- cxx does NOT support: raw pointers, `void*`, function pointers, or variadic functions. These are excluded by design for safety reasons.
- Use `Result<T>` return types in the bridge declaration to have C++ exceptions automatically converted into Rust `Err` values.
- If you need to auto-generate bindings from large C++ headers rather than writing the bridge by hand, combine cxx with `autocxx`.
