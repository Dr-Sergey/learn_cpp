# Understand abi_tag Attribute for Symbol Versioning

**Category:** ABI and Binary Compatibility  
**Standard:** GCC extension , de facto standard on Linux (`[[gnu::abi_tag]]`)  
**Reference:** https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Attributes.html#index-abi_005ftag-type-attribute  

---

## Topic Overview

The `[[gnu::abi_tag]]` attribute (also written as `__attribute__((abi_tag(...)))`) is GCC's mechanism for changing the mangled name of a symbol without changing its source-level name. It was introduced specifically to solve the `std::string` ABI transition problem in GCC 5, where the implementation changed from COW (copy-on-write) to SSO (small string optimization). Symbols tagged with `abi_tag("cxx11")` get a different mangled name than untagged symbols, causing link failures when mixing old and new binaries - the desired behavior. A loud linker error is infinitely better than silent memory corruption at runtime.

The attribute can be applied to classes, functions, and variables. When applied to a class, it affects the mangling of every symbol that uses that class in its signature. This is how GCC automatically detects when code compiled with the old `std::string` is mixed with code compiled with the new `std::string` - they have different abi_tags and thus different mangled names.

If the table feels like a lot, the core message is simple: adding `abi_tag("cxx11")` turns a short mangled name into a long one that includes `__cxx11`, and that difference makes the linker reject mismatched objects instead of silently proceeding.

| Element                      | Without abi_tag             | With `abi_tag("cxx11")`     |
| --- | --- | --- |
| `std::string`                | `Ss` (abbreviation)         | `NSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE` |
| `f(std::string)`             | `_Z1fSs`                   | `_Z1fB5cxx11NSt7...`        |
| Runtime behavior             | COW semantics               | SSO semantics               |
| Mixing old+new objects       | Silent corruption           | Linker error                |

The snippet below shows exactly how the tag propagates through the mangled name - notice that the `__cxx11` qualifier shows up not just on the class itself but on every function that takes it as a parameter.

```cpp
How abi_tag propagates through mangling:

  namespace std {
    inline namespace __cxx11 {
      class __attribute__((abi_tag("cxx11"))) basic_string { ... };
    }
  }

  void process(std::string s);
  // Mangled: _Z7processNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
  //                    ^^^^^^^^^^^^^^^^
  //                    abi_tag causes __cxx11 to appear in mangling

  // Without abi_tag:
  // _Z7processSs  (abbreviated form for std::string)
```

The abi_tag mechanism is distinct from inline namespaces, though they are often used together. Inline namespaces change mangling by adding a namespace qualifier, while abi_tag adds a special `B<tag>` markup to the mangled name that can apply to individual entities without nesting in a namespace.

---

## Self-Assessment

### Q1: Apply abi_tag to your own library types to create tagged symbol versions that detect ABI mismatches at link time

Here we build a `SmallString` class whose layout changed between versions (v1 was heap-only, v2 adds an SSO buffer). Applying `[[gnu::abi_tag("sso")]]` to the class makes every function that mentions `SmallString` in its signature carry the tag in its mangled name. If anyone compiles against the v1 header and tries to link with the v2 library, the linker will refuse rather than silently corrupt memory.

```cpp
#include <cstdio>
#include <cstring>
#include <cstddef>

// === Library header with abi_tag for version 2 of SmallString ===

namespace mylib {

// Version 2: SSO implementation with abi_tag
// Any function using SmallString in its signature will get
// the "sso" tag appended to its mangled name
class
#if __has_cpp_attribute(gnu::abi_tag)
    [[gnu::abi_tag("sso")]]
#endif
SmallString {
public:
    static constexpr std::size_t SSO_CAP = 22;  // fits in-place (+ null)

    SmallString() : size_(0) {
        buf_.local[0] = '\0';
        is_local_ = true;
    }

    explicit SmallString(const char* s) : SmallString() {
        std::size_t len = std::strlen(s);
        if (len <= SSO_CAP) {
            std::memcpy(buf_.local, s, len + 1);
            size_ = len;
            is_local_ = true;
        } else {
            buf_.heap = new char[len + 1];
            std::memcpy(buf_.heap, s, len + 1);
            size_ = len;
            is_local_ = false;
        }
    }

    SmallString(const SmallString& o) : SmallString() {
        *this = SmallString(o.c_str());
    }

    ~SmallString() {
        if (!is_local_) delete[] buf_.heap;
    }

    const char* c_str() const {
        return is_local_ ? buf_.local : buf_.heap;
    }

    std::size_t size() const { return size_; }
    bool is_sso() const { return is_local_; }

private:
    union Storage {
        char  local[SSO_CAP + 1];
        char* heap;
    } buf_;
    std::size_t size_;
    bool is_local_;
    // sizeof(SmallString) in v2 is different from v1
    // abi_tag ensures link-time detection of mismatch
};

// These functions get "sso" in their mangled names because
// SmallString has [[gnu::abi_tag("sso")]]

void print_string(const SmallString& s) {
    std::printf("[%s] (sso=%d, len=%zu)\n",
                s.c_str(), s.is_sso(), s.size());
}

SmallString concat(const SmallString& a, const SmallString& b) {
    char buf[256];
    std::snprintf(buf, sizeof(buf), "%s%s", a.c_str(), b.c_str());
    return SmallString(buf);
}

// Mangled names:
// _ZN5mylib12print_stringB3ssoERKNS_11SmallStringB3ssoE
//                         ^^^^                    ^^^^
// _ZN5mylib6concatB3ssoERKNS_11SmallStringB3ssoES3_
//                 ^^^^                    ^^^^

}  // namespace mylib

int main() {
    mylib::SmallString short_str("hello");     // SSO
    mylib::SmallString long_str("this is a longer string that exceeds SSO");

    mylib::print_string(short_str);
    mylib::print_string(long_str);

    auto combined = mylib::concat(short_str, long_str);
    mylib::print_string(combined);

    return 0;
}
```

Notice the `^^^^` markers in the comments - the `B3sso` tag appears on every function that touches the tagged type, not just on the class itself. That's the transitivity rule in action.

### Q2: Understand how GCC's cxx11 abi_tag works with std::string and diagnose common dual-ABI issues

This is the scenario most Linux developers actually encounter in practice. You have a pre-GCC-5 library sitting on the system, and you're compiling with a modern GCC. If your code passes `std::string` across that library boundary, the linker will complain about an unresolved symbol because the two sides are looking for different mangled names. The code below shows how to check which ABI is active and how to reason about the size difference, which is a dead giveaway when you need to diagnose this at runtime.

```cpp
#include <string>
#include <cstdio>

// GCC 5+ supports dual ABI via _GLIBCXX_USE_CXX11_ABI macro
// Default: _GLIBCXX_USE_CXX11_ABI=1 (new SSO string)
// Override: _GLIBCXX_USE_CXX11_ABI=0 (old COW string)

// === Demonstration of the dual ABI problem ===

// This function uses std::string in its signature
std::string transform(const std::string& input) {
    return "[" + input + "]";
}

// When compiled with _GLIBCXX_USE_CXX11_ABI=1 (default):
// Mangled: _Z9transformB5cxx11RKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
//                      ^^^^^^^^^
//                      abi_tag "cxx11" injected

// When compiled with _GLIBCXX_USE_CXX11_ABI=0:
// Mangled: _Z9transformRKSs
//          (no abi_tag, abbreviated std::string)


// === Common dual-ABI scenario ===
// Library A: compiled with GCC 4.9 (old ABI)
// Library B: compiled with GCC 12 (new ABI, abi_tag "cxx11")
//
// If B calls A's function that takes std::string:
//   B looks for: _Z4funcB5cxx11RKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
//   A exports:   _Z4funcRKSs
//   -> LINKER ERROR (safe failure)
//
// Workarounds:
// 1. Recompile A with new GCC (best solution)
// 2. Compile B with -D_GLIBCXX_USE_CXX11_ABI=0 (use old ABI)
// 3. Use extern "C" at the boundary (avoid std::string in interface)


// === Displaying which ABI is active ===
void check_abi() {
    #if _GLIBCXX_USE_CXX11_ABI
        std::printf("Using NEW ABI (cxx11, SSO string)\n");
    #else
        std::printf("Using OLD ABI (COW string)\n");
    #endif

    std::printf("sizeof(std::string) = %zu\n", sizeof(std::string));
    // New ABI (SSO): typically 32 bytes (8 ptr + 8 size + 16 SSO buffer)
    // Old ABI (COW): typically 8 bytes (just a pointer to shared data)

    std::string s = "hello";
    std::printf("String at: %p, data at: %p, diff: %td\n",
                static_cast<const void*>(&s),
                static_cast<const void*>(s.data()),
                reinterpret_cast<const char*>(s.data()) -
                reinterpret_cast<const char*>(&s));
    // SSO: data pointer points INSIDE the string object (diff is small)
    // COW: data pointer points to heap (diff is large)
}


// === Detecting abi_tag issues with nm ===
// $ nm -C libold.so | grep "transform"
// T transform(std::string const&)
//
// $ nm -C libnew.so | grep "transform"
// T transform[abi:cxx11](std::__cxx11::basic_string<...> const&)
//
// Note the [abi:cxx11] tag in the demangled output!


// === Multiple abi_tags ===

#if __has_cpp_attribute(gnu::abi_tag)

// You can apply multiple tags to a single entity
class [[gnu::abi_tag("v2", "experimental")]] Processor {
public:
    void run() { std::printf("Processor::run\n"); }
};

// Mangled name includes both tags:
// _ZN9ProcessorB2v2B12experimental3runEv
//              ^^^^^^^^^^^^^^^^^^^^

void use_processor(Processor& p) {
    p.run();
}
// _Z13use_processorB2v2B12experimentalRN9ProcessorB2v2B12experimentalE

#endif

int main() {
    check_abi();

    std::string result = transform("world");
    std::printf("Result: %s\n", result.c_str());

    return 0;
}
```

The `sizeof(std::string)` check is the simplest sanity test you can do: 32 bytes means new ABI, 8 bytes means old. A 4x size difference makes it obvious why naively mixing the two would corrupt memory immediately.

### Q3: Build a library that uses abi_tag to provide both old and new implementations simultaneously

The idea here is that during a major library transition, you can keep the old implementation alive and accessible via a non-inline namespace, while the new tagged implementation becomes the default. The `abi_tag` on the v2 `Encoder` ensures that any code compiled against the new header will link only to the new symbols, while old pre-existing binaries continue to find the old untagged symbols. Both live in the same `.so`.

```cpp
#include <cstdio>
#include <cstring>
#include <cstdlib>

namespace codec {

// === Version 1: Simple codec (no abi_tag) ===
namespace v1_impl {

class Encoder {
public:
    Encoder() : buffer_(nullptr), capacity_(0), size_(0) {}
    ~Encoder() { std::free(buffer_); }

    void encode(const char* data, int len) {
        ensure_capacity(size_ + len);
        std::memcpy(buffer_ + size_, data, len);
        size_ += len;
    }

    const char* data() const { return buffer_; }
    int size() const { return size_; }

private:
    void ensure_capacity(int needed) {
        if (needed <= capacity_) return;
        int new_cap = capacity_ == 0 ? 64 : capacity_ * 2;
        while (new_cap < needed) new_cap *= 2;
        buffer_ = static_cast<char*>(std::realloc(buffer_, new_cap));
        capacity_ = new_cap;
    }

    char* buffer_;
    int capacity_;
    int size_;
};

}  // namespace v1_impl


// === Version 2: Encoder with SIMD-friendly alignment (ABI break) ===
namespace v2_impl {

class
#if __has_cpp_attribute(gnu::abi_tag)
    [[gnu::abi_tag("simd")]]
#endif
Encoder {
public:
    static constexpr int ALIGNMENT = 64;  // Cache-line aligned

    Encoder() : buffer_(nullptr), capacity_(0), size_(0), flags_(0) {}
    ~Encoder() { aligned_free(buffer_); }

    void encode(const char* data, int len) {
        ensure_capacity(size_ + len);
        std::memcpy(buffer_ + size_, data, len);
        size_ += len;
    }

    // New in v2: SIMD-optimized batch encode
    void encode_batch(const char** chunks, const int* lengths, int count) {
        int total = 0;
        for (int i = 0; i < count; ++i) total += lengths[i];
        ensure_capacity(size_ + total);
        for (int i = 0; i < count; ++i) {
            std::memcpy(buffer_ + size_, chunks[i], lengths[i]);
            size_ += lengths[i];
        }
    }

    const char* data() const { return buffer_; }
    int size() const { return size_; }
    int alignment() const { return ALIGNMENT; }

private:
    static void* aligned_alloc_impl(int alignment, int size) {
        #ifdef _WIN32
            return _aligned_malloc(size, alignment);
        #else
            void* ptr = nullptr;
            posix_memalign(&ptr, alignment, size);
            return ptr;
        #endif
    }

    static void aligned_free(void* ptr) {
        #ifdef _WIN32
            _aligned_free(ptr);
        #else
            std::free(ptr);
        #endif
    }

    void ensure_capacity(int needed) {
        if (needed <= capacity_) return;
        int new_cap = capacity_ == 0 ? 256 : capacity_ * 2;
        while (new_cap < needed) new_cap *= 2;
        char* new_buf = static_cast<char*>(
            aligned_alloc_impl(ALIGNMENT, new_cap));
        if (buffer_) {
            std::memcpy(new_buf, buffer_, size_);
            aligned_free(buffer_);
        }
        buffer_ = new_buf;
        capacity_ = new_cap;
    }

    char* buffer_;
    int capacity_;
    int size_;
    int flags_;       // new field - sizeof changed!
    // Layout and alignment are different from v1 -> ABI break
};

}  // namespace v2_impl


// === Public API: uses inline namespace to select active version ===
// The abi_tag on v2's Encoder ensures link-time detection

inline namespace current {
    using Encoder = v2_impl::Encoder;
}

// v1 remains accessible for migration
namespace legacy {
    using Encoder = v1_impl::Encoder;
}

// Function signatures include the abi_tag through the type
void process_data(Encoder& enc, const char* input) {
    // If Encoder has [[gnu::abi_tag("simd")]], this function's
    // mangled name includes the tag -> incompatible with v1 callers
    enc.encode(input, static_cast<int>(std::strlen(input)));
    std::printf("Encoded %d bytes (aligned=%d)\n",
                enc.size(), enc.alignment());
}

}  // namespace codec


int main() {
    // Current version (v2 with abi_tag "simd")
    codec::Encoder enc;
    codec::process_data(enc, "hello world");

    // Legacy version (no abi_tag)
    codec::legacy::Encoder legacy_enc;
    legacy_enc.encode("hello", 5);
    std::printf("Legacy encoded %d bytes\n", legacy_enc.size());

    return 0;
}
```

The `codec::legacy::Encoder` at the bottom is the migration path: old code that was already compiled against v1 keeps working, while new code gets the tagged v2 symbols. Only when you drop the `legacy` namespace entirely do you cut support for the old ABI.

---

## Notes

- `[[gnu::abi_tag("tag")]]` modifies the **mangled symbol name** by inserting `B<len><tag>` - any function using the tagged type gets the tag transitively.
- The `_GLIBCXX_USE_CXX11_ABI` macro controls whether libstdc++ uses the `abi_tag("cxx11")`-tagged implementations - set to 0 to link with pre-GCC5 libraries.
- abi_tag is **transitive**: if a function's parameter or return type has an abi_tag, the function itself gets tagged in its mangled name.
- Multiple abi_tags can be applied to a single entity: `[[gnu::abi_tag("v2", "experimental")]]`.
- abi_tag is a **GCC/Clang extension** - MSVC does not support it. Cross-platform code should guard with `#if __has_cpp_attribute(gnu::abi_tag)`.
- The demangled form shows tags as `[abi:tagname]`, e.g., `mylib::SmallString[abi:sso]` - look for these in `nm -C` output to diagnose link errors.
- `sizeof(std::string)` is 32 bytes with cxx11 ABI (SSO) vs 8 bytes with old ABI (COW) - a dramatic difference that makes mixing catastrophic.
