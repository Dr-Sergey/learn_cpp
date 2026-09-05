# Understand What Changes Break ABI in C++ Libraries

**Category:** ABI and Binary Compatibility  
**Standard:** C++11, C++14, C++17, C++20 (ABI is implementation-defined)  
**Reference:** https://community.kde.org/Policies/Binary_Compatibility_Issues_With_C++  

---

## Topic Overview

ABI (Application Binary Interface) compatibility determines whether a new version of a shared library can replace an old one without recompiling dependent applications. Breaking ABI means that existing binaries will malfunction - crashing, corrupting memory, or producing wrong results - even though the source code compiles cleanly. Every C++ library maintainer must know exactly which code changes are ABI-safe and which are not.

The fundamental principle is this: anything that changes the **size, layout, or mangled name** of a type, function, or vtable is an ABI break. This includes surprisingly innocent-looking changes like adding a private data member (changes `sizeof`), adding a virtual function (changes vtable layout), or changing a `constexpr` function to non-`constexpr` (may change calling convention). The reason this trips people up is that "private" in C++ means hidden from the user's *source* - it does not mean hidden from the *binary layout*. If you add a private field, the compiler has to make the object bigger, and every piece of code that was compiled against the old header will now be using the wrong `sizeof`.

### ABI-Breaking Changes - Complete Matrix

If the table feels like a lot to absorb, focus on the pattern: anything that touches size, field positions, vtable order, or name encoding is a break. Adding new things that don't disturb existing positions is generally safe.

| Change                                        | Breaks ABI? | Why                                          |
| --- | :---: | --- |
| Add public data member                        | **YES**     | Changes sizeof and member offsets             |
| Add private data member                       | **YES**     | Changes sizeof and member offsets             |
| Remove any data member                        | **YES**     | Changes sizeof and member offsets             |
| Reorder data members                          | **YES**     | Changes member offsets                        |
| Change data member type                       | **YES**     | Changes sizeof and alignment                  |
| Add virtual function                          | **YES**     | Changes vtable layout                         |
| Remove virtual function                       | **YES**     | Changes vtable layout                         |
| Reorder virtual functions                     | **YES**     | Changes vtable slot indices                   |
| Add first virtual function                    | **YES**     | Adds hidden vptr member, changes sizeof       |
| Add non-virtual public base class             | **YES**     | Changes layout and possibly sizeof            |
| Add virtual base class                        | **YES**     | Changes layout dramatically                   |
| Remove base class                             | **YES**     | Changes layout and vtable                     |
| Change enum underlying type                   | **YES**     | Changes sizeof of enum and ABI of functions   |
| Change function return type                   | **YES**     | Changes mangled name and calling convention   |
| Change function parameter types               | **YES**     | Changes mangled name                          |
| Change template parameter                     | **YES**     | Changes mangled name (different instantiation)|
| Add new enum value (at end)                   | No          | Existing values unchanged                     |
| Add new non-virtual function                  | No          | No layout change, new symbol only             |
| Add new static member                         | No          | No layout change                              |
| Add new overload                              | No          | New mangled name, existing unchanged          |
| Add default argument                          | No          | Resolved at compile time, not in binary       |
| Change function implementation                | No          | Same symbol, different code                   |
| Add new class/struct                          | No          | New symbols only                              |

Changes that are **conditionally ABI-breaking** require special attention: making a previously non-inline function inline (removes the out-of-line definition), changing exception specification (affects mangling in C++17), or modifying alignment attributes.

---

## Self-Assessment

### Q1: Identify all ABI-breaking changes in a proposed library update and classify their severity

Here we compare an imaginary imaging library at v1.0 against a proposed v1.1 update. Read both class definitions carefully and try to spot each break before looking at the analysis table below.

```cpp
// === Library v1.0 (currently shipped) ===

namespace imglib {

class Image {
public:
    Image(int w, int h);
    virtual ~Image();

    virtual void render() const;
    virtual void resize(int w, int h);

    int width() const;
    int height() const;
    const unsigned char* data() const;

private:
    int width_;
    int height_;
    unsigned char* data_;
    // sizeof(Image) on 64-bit: 8 (vptr) + 4 + 4 + 8 = 24
};

enum class PixelFormat {
    RGB,
    RGBA,
    Grayscale
};

void save_image(const Image& img, const char* path, PixelFormat fmt);

}  // namespace imglib


// === Proposed Library v1.1 - REVIEW FOR ABI BREAKS ===

namespace imglib {

class Image {
public:
    Image(int w, int h);
    virtual ~Image();

    virtual void render() const;
    virtual void rotate(double angle);     // [1] NEW virtual - ABI BREAK!
    virtual void resize(int w, int h);     // shifted vtable slot!

    int width() const;
    int height() const;
    const unsigned char* data() const;
    std::size_t data_size() const;         // [2] OK - new non-virtual

private:
    int width_;
    int height_;
    unsigned char* data_;
    std::size_t data_size_;                // [3] NEW member - ABI BREAK!
    PixelFormat format_;                   // [4] NEW member - ABI BREAK!
    // sizeof(Image) changed: 24 -> 40
};

enum class PixelFormat : uint32_t {        // [5] specified underlying type
    RGB,                                   //     ABI BREAK if previously defaulted!
    RGBA,
    Grayscale,
    BGR,                                   // [6] New value at end - OK
    BGRA                                   // [7] New value at end - OK
};

void save_image(const Image& img, const char* path, PixelFormat fmt);
// [8] OK - same signature, same mangled name

bool load_image(const char* path, Image& img);
// [9] OK - entirely new function

}
```

The virtual function insertion at [1] is especially dangerous because it doesn't just add a new slot - it shunts `resize` down by one position. Any derived class compiled against v1.0 that overrides `resize` will now override the wrong slot at runtime.

**Analysis:**

```cpp
| # | Change                         | ABI-Breaking? | Impact Level |
| --- | --- | :---: | :---: |
| 1 | Added virtual `rotate()`       | YES           | CRITICAL     |
| 2 | Added non-virtual data_size()  | No            | Safe         |
| 3 | Added member `data_size_`      | YES           | CRITICAL     |
| 4 | Added member `format_`         | YES           | CRITICAL     |
| 5 | Changed enum underlying type   | YES           | HIGH         |
| 6 | New enum value BGR             | No            | Safe         |
| 7 | New enum value BGRA            | No            | Safe         |
| 8 | Unchanged function             | No            | Safe         |
| 9 | New function                   | No            | Safe         |
```

### Q2: Design a class that can safely evolve without ABI breaks using the Pimpl pattern and reserved vtable slots

The Pimpl pattern (pointer to implementation) is the classic solution here. By hiding all data members behind a pointer to an opaque type, `sizeof(Image)` stays constant no matter how much the implementation changes. The pointer is always pointer-sized, and that's the only thing the ABI sees. Virtual reserved slots let you plan for future polymorphism without committing to names yet.

```cpp
#include <memory>
#include <cstdio>

namespace imglib {

// Forward declaration only - implementation hidden
struct ImageImpl;

class Image {
public:
    Image(int w, int h);
    ~Image();

    // Non-virtual public API - adding new methods is ABI-safe
    int width() const;
    int height() const;
    const unsigned char* data() const;
    std::size_t data_size() const;

    // Future methods can be added here without ABI break
    // because Pimpl hides the actual data layout

    // Reserved virtual methods for future extension
    // When you MUST have virtual functions, reserve slots:
    virtual void render() const;
    virtual void resize(int w, int h);
    virtual void reserved_virtual_1();  // placeholder
    virtual void reserved_virtual_2();  // placeholder
    virtual void reserved_virtual_3();  // placeholder
    // Adding virtuals AFTER these is still an ABI break,
    // but you can repurpose reserved slots safely

protected:
    virtual ~Image() = 0;  // prevent direct deletion through base

private:
    // Pimpl hides all data members
    // sizeof(Image) = sizeof(vptr) + sizeof(unique_ptr) = 16 on 64-bit
    // This NEVER changes regardless of internal data evolution
    std::unique_ptr<ImageImpl> impl_;
};

// === Implementation file (image.cpp) ===
struct ImageImpl {
    int width;
    int height;
    unsigned char* data;
    std::size_t data_size;
    // v1.1: can freely add more fields!
    // PixelFormat format;     // no ABI break
    // std::string metadata;   // no ABI break
    // int quality;            // no ABI break
};

Image::Image(int w, int h) : impl_(std::make_unique<ImageImpl>()) {
    impl_->width = w;
    impl_->height = h;
    impl_->data_size = static_cast<std::size_t>(w) * h * 4;
    impl_->data = new unsigned char[impl_->data_size]();
}

Image::~Image() {
    delete[] impl_->data;
}

int Image::width() const { return impl_->width; }
int Image::height() const { return impl_->height; }
const unsigned char* Image::data() const { return impl_->data; }
std::size_t Image::data_size() const { return impl_->data_size; }
void Image::render() const { std::printf("rendering %dx%d\n", width(), height()); }
void Image::resize(int w, int h) { /* ... */ }
void Image::reserved_virtual_1() {}
void Image::reserved_virtual_2() {}
void Image::reserved_virtual_3() {}

}  // namespace imglib
```

Notice that `ImageImpl` can grow without limit in future versions and none of that growth is visible to the caller - from the binary perspective, `Image` is always just a vptr plus a pointer.

### Q3: Write a test that detects accidental ABI breaks by verifying class layout invariants at compile time

`static_assert` with `sizeof`, `alignof`, and `offsetof` is the simplest and most effective ABI tripwire you can add to a library. These assertions compile to zero runtime cost and fire the moment someone accidentally changes the layout. Think of them as a contract: "if any of these asserts fail, someone needs to bump the soname and update the changelog."

```cpp
#include <cstddef>
#include <cstdint>
#include <type_traits>

namespace mylib {

struct NetworkPacket {
    std::uint32_t magic;        // offset 0
    std::uint16_t version;      // offset 4
    std::uint16_t flags;        // offset 6
    std::uint64_t timestamp;    // offset 8
    std::uint32_t payload_len;  // offset 16
    std::uint32_t checksum;     // offset 20
    char payload[];             // flexible array member (C99-style, offset 24)
};

// === Compile-time ABI checks ===
// These static_asserts fire instantly if someone changes the layout

// Size check - catches added/removed members
static_assert(sizeof(NetworkPacket) == 24,
    "ABI BREAK: NetworkPacket size changed! Update version.");

// Alignment check
static_assert(alignof(NetworkPacket) == 8,
    "ABI BREAK: NetworkPacket alignment changed!");

// Individual member offset checks - catches reordering
static_assert(offsetof(NetworkPacket, magic) == 0,
    "ABI BREAK: 'magic' offset changed!");
static_assert(offsetof(NetworkPacket, version) == 4,
    "ABI BREAK: 'version' offset changed!");
static_assert(offsetof(NetworkPacket, flags) == 6,
    "ABI BREAK: 'flags' offset changed!");
static_assert(offsetof(NetworkPacket, timestamp) == 8,
    "ABI BREAK: 'timestamp' offset changed!");
static_assert(offsetof(NetworkPacket, payload_len) == 16,
    "ABI BREAK: 'payload_len' offset changed!");
static_assert(offsetof(NetworkPacket, checksum) == 20,
    "ABI BREAK: 'checksum' offset changed!");

// Triviality checks - ABI-relevant for parameter passing
static_assert(std::is_trivially_copyable_v<NetworkPacket>,
    "ABI BREAK: NetworkPacket must be trivially copyable for memcpy/send!");
static_assert(std::is_standard_layout_v<NetworkPacket>,
    "ABI BREAK: NetworkPacket must be standard layout for C interop!");

// Macro for convenient per-class ABI lock-down
#define ASSERT_ABI_STABLE(Type, expected_size, expected_align) \
    static_assert(sizeof(Type) == expected_size,               \
        "ABI BREAK: " #Type " size changed from " #expected_size); \
    static_assert(alignof(Type) == expected_align,             \
        "ABI BREAK: " #Type " alignment changed from " #expected_align)

struct Config {
    std::uint32_t timeout_ms;
    std::uint32_t max_retries;
    std::uint64_t max_payload;
    bool use_compression;
    // padding: 7 bytes
};

ASSERT_ABI_STABLE(Config, 24, 8);

}  // namespace mylib

int main() {
    // Layout checks passed - binary is ABI-compatible
    mylib::NetworkPacket pkt{};
    pkt.magic = 0xFEEDFACE;
    return 0;
}
```

The `ASSERT_ABI_STABLE` macro at the bottom is a nice convenience - you can drop it on any type you want to protect and get both a size and alignment check with one line.

---

## Notes

- **Adding a private data member** is an ABI break - even though it's invisible to users, it changes `sizeof` and all offset calculations.
- **Adding a virtual function at the end** shifts secondary vtable entries for derived classes compiled against the old layout.
- The safest library interface uses **Pimpl + non-virtual functions + extern "C"** - this combination is almost immune to ABI breaks.
- Use `static_assert` on `sizeof`, `alignof`, and `offsetof` to create compile-time **ABI tripwires** that catch accidental breaks immediately.
- When you must break ABI: **bump the soname** (e.g., `libfoo.so.2`), update inline namespace version, and document the break in changelog.
- KDE and Qt maintain detailed **ABI compatibility policies** that serve as excellent references for library authors.
- Tools like `abi-compliance-checker` and `abidiff` (from libabigail) can automatically detect ABI breaks between two versions of a shared library.
