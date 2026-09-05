# Use Inline Namespace Versioning for ABI Evolution

**Category:** ABI and Binary Compatibility  
**Standard:** C++11 and later  
**Reference:** https://en.cppreference.com/w/cpp/language/namespace#Inline_namespaces  

---

## Topic Overview

Inline namespaces are the primary mechanism in C++ for **ABI versioning without breaking source compatibility**. When a type's implementation changes in a way that alters its binary layout (e.g., `std::string` switching from COW to SSO), you need existing binaries to fail at link time rather than silently using an incompatible layout. Inline namespaces achieve this by changing the mangled symbol names while keeping the unqualified source-level names accessible.

Here's the intuition: an inline namespace is transparent to the user writing `mylib::Widget`, but the compiler encodes the namespace name into the mangled symbol. So `mylib::v1::Widget` and `mylib::v2::Widget` are genuinely different symbols in the object file, even though a user writing `mylib::Widget` gets whichever one is currently inline. If you ship a library update where only `v2` is inline and then someone links old `v1` objects against the new library, the linker cannot find the `v1` symbols - it fails loudly instead of silently reading the wrong memory layout.

The canonical example is libstdc++'s transition of `std::string`. In GCC 5, `std::string` was moved from a COW (copy-on-write) implementation to SSO (small string optimization). The new implementation lives in `std::__cxx11::string`, which is an inline namespace - so source code writing `std::string` transparently uses the new type, but the mangled names differ, causing linker errors if old and new objects are mixed. This is the desired behavior: a loud failure at link time is infinitely better than silent memory corruption at runtime.

| Scenario                               | Without Inline NS        | With Inline NS               |
| --- | --- | --- |
| Source compatibility                   | Maintained               | Maintained                   |
| Binary compatibility with old .so/.lib | Silent corruption risk   | Linker error (safe failure)  |
| Mangled name of `MyLib::Widget`        | `N5MyLib6WidgetE`        | `N5MyLib2v26WidgetE`         |
| User code changes needed               | None                     | None                         |
| Library maintainer effort              | None                     | Create new inline namespace  |

The side-by-side below shows the mangling difference directly. The `v1`/`v2` string ends up embedded in the symbol name, which is exactly what makes the linker error possible.

```cpp
Library version 1:              Library version 2:
namespace mylib {                namespace mylib {
  inline namespace v1 {            inline namespace v2 {
    struct Widget { int x; };        struct Widget { int x; int y; };
  }                                }
}                                }

Mangled: N5mylib2v16WidgetE      Mangled: N5mylib2v26WidgetE
                                 -> Link error if v1 object mixed with v2!
```

The inline namespace technique is used throughout the standard library: libc++ puts everything in `std::__1`, libstdc++ uses `std::__cxx11`, and MSVC STL uses versioned implementation namespaces. When designing your own library, adopting this pattern from day one prevents painful ABI breaks later.

---

## Self-Assessment

### Q1: Implement a library that uses inline namespace versioning to safely evolve a class layout across major versions

The practical way to do this in a real library is to drive the active ABI version from a single macro in a config header, then use that macro everywhere through a pair of `MYLIB_BEGIN_NAMESPACE` / `MYLIB_END_NAMESPACE` macros. That way you never have to manually update each file when you cut a new ABI version - you just change one number.

```cpp
// === mylib/config.h ===
#ifndef MYLIB_CONFIG_H
#define MYLIB_CONFIG_H

// Define which ABI version is active
// Change this when making ABI-breaking changes
#define MYLIB_ABI_VERSION 2

#if MYLIB_ABI_VERSION == 1
    #define MYLIB_ABI_NAMESPACE v1
#elif MYLIB_ABI_VERSION == 2
    #define MYLIB_ABI_NAMESPACE v2
#else
    #error "Unknown MYLIB_ABI_VERSION"
#endif

#define MYLIB_BEGIN_NAMESPACE \
    namespace mylib { inline namespace MYLIB_ABI_NAMESPACE {
#define MYLIB_END_NAMESPACE \
    } }

#endif

// === mylib/buffer.h (version 2) ===
#ifndef MYLIB_BUFFER_H
#define MYLIB_BUFFER_H

#include <cstddef>
#include <cstring>
#include <algorithm>
// #include "config.h"

MYLIB_BEGIN_NAMESPACE

// v1: Simple heap-allocated buffer
// v2: Added SSO (small buffer optimization) - ABI-breaking change!
class Buffer {
public:
    static constexpr std::size_t SSO_CAPACITY = 64;  // new in v2

    Buffer() : size_(0), capacity_(SSO_CAPACITY), data_(sso_buf_) {
        sso_buf_[0] = '\0';
    }

    explicit Buffer(const char* str) : Buffer() {
        std::size_t len = std::strlen(str);
        reserve(len + 1);
        std::memcpy(data_, str, len + 1);
        size_ = len;
    }

    Buffer(const Buffer& other) : Buffer() {
        reserve(other.size_ + 1);
        std::memcpy(data_, other.data_, other.size_ + 1);
        size_ = other.size_;
    }

    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            Buffer tmp(other);
            swap(tmp);
        }
        return *this;
    }

    ~Buffer() {
        if (data_ != sso_buf_) {
            delete[] data_;
        }
    }

    void reserve(std::size_t new_cap) {
        if (new_cap <= capacity_) return;
        char* new_data = new char[new_cap];
        if (size_ > 0) std::memcpy(new_data, data_, size_ + 1);
        if (data_ != sso_buf_) delete[] data_;
        data_ = new_data;
        capacity_ = new_cap;
    }

    void swap(Buffer& other) noexcept {
        // SSO-aware swap - more complex than v1
        // (simplified for illustration)
        std::swap(size_, other.size_);
        std::swap(capacity_, other.capacity_);
        // Full implementation would handle sso_buf_ correctly
    }

    const char* c_str() const { return data_; }
    std::size_t size() const { return size_; }
    bool is_sso() const { return data_ == sso_buf_; }

private:
    std::size_t size_;
    std::size_t capacity_;
    char* data_;
    char sso_buf_[SSO_CAPACITY];  // NOT present in v1 - ABI break!
};

MYLIB_END_NAMESPACE

#endif
```

The `sso_buf_` field at the bottom is the whole reason the version bump is necessary - it makes `sizeof(Buffer)` much larger than in v1, so any old code that allocates or copies a `Buffer` by value would corrupt memory if it used the wrong size.

### Q2: Demonstrate how inline namespaces cause controlled linker failures when object files from different ABI versions are mixed

This example drives the point home concretely. Two translation units compiled with different `MYLIB_ABI_VERSION` definitions will produce incompatible symbol names, and the linker will refuse to combine them. The comments show the exact mangled names so you can see why the symbols don't match.

```cpp
// === Scenario: compile file_a.cpp with ABI v1, file_b.cpp with ABI v2 ===

// --- file_a.cpp (compiled with MYLIB_ABI_VERSION=1) ---
namespace mylib {
    inline namespace v1 {
        struct Message {
            char text[32];
            int priority;
            // sizeof = 36 in v1
        };

        // Mangled: _ZN5mylib2v112send_messageERKNS0_7MessageE
        void send_message(const Message& msg);
    }
}

void file_a_function() {
    mylib::Message msg{"hello", 5};  // resolves to mylib::v1::Message
    mylib::send_message(msg);         // resolves to mylib::v1::send_message
}

// --- file_b.cpp (compiled with MYLIB_ABI_VERSION=2) ---
namespace mylib {
    inline namespace v2 {
        struct Message {
            char text[64];      // expanded - ABI break!
            int priority;
            int flags;          // new field - ABI break!
            // sizeof = 72 in v2
        };

        // Mangled: _ZN5mylib2v212send_messageERKNS0_7MessageE
        //                  ^^                ^^
        // Note: "v2" instead of "v1" - names WILL NOT link!
        void send_message(const Message& msg) {
            // implementation for v2
        }
    }
}

// --- Link attempt ---
// $ g++ -c file_a.cpp -DMYLIB_ABI_VERSION=1
// $ g++ -c file_b.cpp -DMYLIB_ABI_VERSION=2
// $ g++ file_a.o file_b.o
//
// LINKER ERROR:
// undefined reference to `mylib::v1::send_message(mylib::v1::Message const&)'
//
// This is EXACTLY what we want - a clear error instead of:
// - Reading 36 bytes when 72 are expected
// - Stack corruption from size mismatch
// - Silent data truncation

// === Verification: demangling the symbols ===
// $ nm file_a.o | c++filt
// U mylib::v1::send_message(mylib::v1::Message const&)
//
// $ nm file_b.o | c++filt
// T mylib::v2::send_message(mylib::v2::Message const&)
//
// Different symbols -> linker cannot resolve -> safe failure
```

The `nm | c++filt` commands at the bottom are your first diagnostic tool when you suspect an ABI mismatch. If you see `v1` in one file and `v2` in another, you know immediately what went wrong without having to dig into crash dumps.

### Q3: Handle backward compatibility by keeping old ABI versions accessible alongside the current one

In a real library you often can't just drop the old format on day one - you have existing serialized data, network protocols, or third-party integrations that still speak v1. The answer is to keep `v1` as a regular (non-inline) namespace so it remains explicitly accessible, while `v2` is the inline namespace that handles all unqualified code. A migration helper function can bridge the two.

```cpp
#include <cstdio>
#include <cstdint>
#include <cstring>

namespace serialization {

// Old ABI - kept for backward compatibility with existing serialized data
namespace v1 {
    struct Header {
        std::uint32_t magic;
        std::uint32_t version;
        std::uint32_t payload_size;
        // sizeof = 12
    };

    bool parse_header(const char* data, Header& out) {
        std::memcpy(&out, data, sizeof(Header));
        return out.magic == 0xDEADBEEF && out.version == 1;
    }
}

// Current ABI - inline so unqualified names resolve here
inline namespace v2 {
    struct Header {
        std::uint32_t magic;
        std::uint32_t version;
        std::uint64_t payload_size;   // widened to 64-bit - ABI break
        std::uint32_t checksum;       // new field
        std::uint32_t flags;          // new field
        // sizeof = 24
    };

    bool parse_header(const char* data, Header& out) {
        std::memcpy(&out, data, sizeof(Header));
        return out.magic == 0xDEADBEEF && out.version == 2;
    }
}

// Migration helper that handles both versions
struct ParsedData {
    Header header;  // always v2 (inline namespace)
    bool from_legacy;
};

ParsedData parse_any_version(const char* data) {
    // Try current version first
    Header h;
    if (parse_header(data, h)) {
        return {h, false};
    }

    // Fall back to legacy
    v1::Header old_h;
    if (v1::parse_header(data, old_h)) {
        // Convert v1 -> v2
        Header converted{};
        converted.magic = old_h.magic;
        converted.version = 2;
        converted.payload_size = old_h.payload_size;  // implicit widening
        converted.checksum = 0;
        converted.flags = 0;
        return {converted, true};
    }

    std::fprintf(stderr, "Unknown format\n");
    return {{}, false};
}

}  // namespace serialization

int main() {
    // User code doesn't need to know about versioning
    serialization::Header h{};  // This is serialization::v2::Header
    h.magic = 0xDEADBEEF;
    h.version = 2;

    // But can explicitly access old version when needed
    serialization::v1::Header old_h{};
    old_h.magic = 0xDEADBEEF;
    old_h.version = 1;

    std::printf("Current Header size: %zu\n", sizeof(serialization::Header));
    std::printf("Legacy  Header size: %zu\n", sizeof(serialization::v1::Header));

    return 0;
}
```

Notice that user code in `main` just writes `serialization::Header` and gets v2 automatically - no changes needed. But the explicit `serialization::v1::Header` form is still available for migration code or legacy data parsing. That's the whole point of the pattern: seamless for new code, reachable for migration code.

---

## Notes

- Inline namespaces change **mangled symbol names** without changing source-level qualified names - the key insight enabling ABI versioning.
- Always define the inline namespace via a **macro in a single config header** so the entire library uses the same ABI version consistently.
- Keep **old namespace versions non-inline but accessible** for migration code, deserialization of old formats, and interop with legacy systems.
- The standard library itself relies on this pattern: `std::string` is actually `std::__cxx11::basic_string` on libstdc++ (GCC 5+).
- Do NOT make multiple namespace versions inline simultaneously - only **one version** should be inline at compile time.
- The ABI version macro (`MYLIB_ABI_VERSION`) should be set by the **library build system**, not by users - users should never override it.
- This technique protects against ABI mismatch but does **not solve the diamond problem** of depending on two libraries that each depend on different ABI versions of a third library.
