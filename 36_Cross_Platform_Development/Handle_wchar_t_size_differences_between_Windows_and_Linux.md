# Handle wchar_t size differences between Windows and Linux

**Category:** Cross Platform Development  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/types>  

---

## Topic Overview

### The Problem

`wchar_t` has a **different size on different platforms**, and that size difference has real consequences for any string you store in it:

| Platform | `sizeof(wchar_t)` | Encoding |
| --- | --- | --- |
| Windows | 2 bytes | UTF-16 (surrogate pairs for > U+FFFF) |
| Linux/macOS | 4 bytes | UTF-32 (one code point per `wchar_t`) |

This means `wchar_t` strings are **not portable**. The same emoji requires two `wchar_t` elements on Windows (a UTF-16 surrogate pair) but only one on Linux (a UTF-32 code point). If you serialize a `wchar_t` array and read it back on the other platform, you get corrupt text:

```cpp
#include <cwchar>

static_assert(sizeof(wchar_t) == 2);  // Passes on Windows, fails on Linux
static_assert(sizeof(wchar_t) == 4);  // Passes on Linux, fails on Windows

// This string has different binary representations on each platform
const wchar_t* greeting = L"Hello 🌍";
// Windows: {0x0048, 0x0065, 0x006C, 0x006C, 0x006F, 0x0020, 0xD83C, 0xDF0D, 0x0000}
//          (emoji is a surrogate pair: 2 wchar_t)
// Linux:   {0x00000048, 0x00000065, 0x0000006C, ..., 0x0001F30D, 0x00000000}
//          (emoji is a single wchar_t)
```

### Why Windows Uses 16-bit `wchar_t`

Windows adopted Unicode early (Windows NT, 1993) when Unicode was defined as a 16-bit encoding. The "wide" Win32 APIs (`CreateFileW`, `MessageBoxW`) all use UTF-16. When Unicode expanded beyond U+FFFF to accommodate the full range of scripts and emoji, Windows was already committed to 2-byte `wchar_t` and had to introduce surrogate pairs to handle the extension.

Linux took a different path: it adopted UTF-8 as the primary encoding and made `wchar_t` 32-bit so each element can represent any Unicode code point directly, without surrogate complications.

### The Cross-Platform Solution: Use UTF-8 Internally

The right approach is to avoid `wchar_t` in your internal data model entirely and use UTF-8 (`std::string`) everywhere. Convert to the platform's native wide string only at the OS API boundary. This `CrossPlatformString` class shows the pattern - UTF-8 lives inside the object, and the conversion to `wchar_t` happens only when you actually need to call a Win32 API:

```cpp
#include <string>
#include <string_view>

// Use std::string with UTF-8 encoding as the universal internal format
class CrossPlatformString {
    std::string utf8_;  // UTF-8 encoded

public:
    explicit CrossPlatformString(std::string_view utf8) : utf8_(utf8) {}

    // Convert to platform-native wide string only at system boundaries
#ifdef _WIN32
    std::wstring to_wide() const {
        if (utf8_.empty()) return {};
        int len = MultiByteToWideChar(CP_UTF8, 0,
                                       utf8_.data(),
                                       static_cast<int>(utf8_.size()),
                                       nullptr, 0);
        std::wstring result(len, L'\0');
        MultiByteToWideChar(CP_UTF8, 0,
                            utf8_.data(),
                            static_cast<int>(utf8_.size()),
                            result.data(), len);
        return result;
    }

    static CrossPlatformString from_wide(std::wstring_view wide) {
        if (wide.empty()) return CrossPlatformString{""};
        int len = WideCharToMultiByte(CP_UTF8, 0,
                                       wide.data(),
                                       static_cast<int>(wide.size()),
                                       nullptr, 0, nullptr, nullptr);
        std::string utf8(len, '\0');
        WideCharToMultiByte(CP_UTF8, 0,
                            wide.data(),
                            static_cast<int>(wide.size()),
                            utf8.data(), len, nullptr, nullptr);
        return CrossPlatformString{utf8};
    }
#endif

    const std::string& str() const { return utf8_; }
    const char* c_str() const { return utf8_.c_str(); }
};
```

On Linux, `to_wide` and `from_wide` are not compiled at all - there is no conversion needed because POSIX APIs already accept UTF-8.

### C++20/23 `char8_t` and `std::u8string`

C++20 introduced `char8_t` to distinguish UTF-8 from plain `char`. If you want a type that unambiguously means UTF-8 regardless of platform or locale settings, `char8_t` is the right choice. The key insight is that `char16_t` and `char32_t` are genuinely portable - their sizes are guaranteed by the standard, unlike `wchar_t`:

```cpp
// C++20: explicit UTF-8 type
const char8_t* utf8_lit = u8"Hello 🌍";  // Type: const char8_t[]
std::u8string  utf8_str = u8"Hello 🌍";  // UTF-8 std::basic_string

// Fixed-width character types (always portable):
const char16_t* utf16 = u"Hello 🌍";  // Always 2-byte code units
const char32_t* utf32 = U"Hello 🌍";  // Always 4-byte code units

static_assert(sizeof(char16_t) == 2);  // Guaranteed on ALL platforms
static_assert(sizeof(char32_t) == 4);  // Guaranteed on ALL platforms
```

### Platform Boundary Pattern

`std::filesystem::path` is your best friend here. It stores paths in the native encoding internally and lets you construct it from UTF-8. On Windows it holds a `std::wstring` internally and passes it to the Win32 file APIs. On Linux it holds a `std::string` and passes it directly to POSIX. Your application code never needs to know which is which:

```cpp
#include <filesystem>
#include <string>

// Internal: always UTF-8
void process_file(const std::string& utf8_path) {
    // std::filesystem::path handles the conversion automatically
    std::filesystem::path p = std::filesystem::u8path(utf8_path);

    // On Windows: p.native() returns std::wstring (UTF-16)
    // On Linux: p.native() returns std::string (UTF-8)

    // fstream with path - works on both platforms
    std::ifstream file(p);
}

// Windows API wrapper
#ifdef _WIN32
void show_message(const std::string& utf8_text) {
    // Convert at the boundary
    int len = MultiByteToWideChar(CP_UTF8, 0,
                                   utf8_text.data(),
                                   static_cast<int>(utf8_text.size()),
                                   nullptr, 0);
    std::wstring wide(len, L'\0');
    MultiByteToWideChar(CP_UTF8, 0,
                        utf8_text.data(),
                        static_cast<int>(utf8_text.size()),
                        wide.data(), len);
    MessageBoxW(nullptr, wide.c_str(), L"Info", MB_OK);
}
#endif
```

### Avoid `wchar_t` in Portable Code

Here is a direct before/after showing the problem. The "bad" version uses `std::wstring` and `L""` literals, which differ in meaning between platforms. The "good" version uses UTF-8 throughout, which has identical byte sequences on every platform:

```cpp
// BAD - non-portable
void bad_portable() {
    std::wstring name = L"file.txt";            // Different encoding on each platform
    std::wofstream file(name);                   // Works differently on each platform
    file << L"Data: " << L"日本語" << std::endl; // Encoding depends on locale
}

// GOOD - portable
void good_portable() {
    std::string name = "file.txt";               // UTF-8 everywhere
    std::ofstream file(name);                    // Works on both (C++17 with filesystem)
    file << "Data: " << "日本語" << std::endl;   // Source file saved as UTF-8
}
```

The good version also happens to be simpler code. The `std::wstring` version is both wrong and more verbose.

---

## Self-Assessment

### Q1: Why can't you just use `std::wstring` everywhere for portability

Because `wchar_t` is 2 bytes on Windows (UTF-16) and 4 bytes on Linux (UTF-32). A `std::wstring` containing emoji or supplementary characters has a different number of elements on each platform. Serializing or transmitting `wchar_t` data between platforms produces corrupt text. Using UTF-8 (`std::string` or `std::u8string`) gives identical byte sequences on all platforms.

### Q2: What is the recommended approach for Windows API calls in cross-platform code

Use UTF-8 (`std::string`) as your internal representation. At the Windows API boundary, convert to `std::wstring` using `MultiByteToWideChar(CP_UTF8, ...)`. Wrap this in a helper function. On Linux, no conversion is needed - the system APIs already accept UTF-8.

### Q3: How does `std::filesystem::path` help

`std::filesystem::path` handles encoding automatically. You construct it from UTF-8 (via `u8path` or directly in C++20) and it stores the native representation internally. On Windows, `path::native()` returns `std::wstring`; on Linux, it returns `std::string`. File I/O operations that accept `path` work correctly on both platforms without manual conversion.

---

## Notes

- Always save source files as UTF-8 - this ensures string literals are portable.
- `std::filesystem::path` is the best abstraction for cross-platform file paths.
- On Windows, call `SetConsoleOutputCP(CP_UTF8)` for UTF-8 console output.
- Consider using the `{fmt}` library or `std::format` (C++20) for locale-independent formatting.
- `char16_t` and `char32_t` are genuinely portable - consider them for network protocols.
