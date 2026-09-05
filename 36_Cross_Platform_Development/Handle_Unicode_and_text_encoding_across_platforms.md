# Handle Unicode and Text Encoding Across Platforms

**Category:** Cross Platform Development  
**Standard:** C++20, C++26  
**Reference:** <https://en.cppreference.com/w/cpp/language/string_literal>  

---

## Topic Overview

Unicode handling is one of the most treacherous areas of cross-platform C++ development. The fundamental problem: Windows uses UTF-16 (`wchar_t` = 16-bit) for its native API, while POSIX systems use UTF-8 byte strings (`char`). The C++ standard has evolved to bridge this gap, but significant pitfalls remain.

| Type | Size | Encoding | Platform Usage |
| --- | --- | --- | --- |
| `char` | 8 bits | Locale-dependent (often UTF-8 on Linux) | POSIX APIs, narrow strings |
| `wchar_t` | 16 bits (Win) / 32 bits (POSIX) | UTF-16 (Win) / UTF-32 (POSIX) | Win32 API (`W` functions) |
| `char8_t` (C++20) | 8 bits | UTF-8 (guaranteed) | Type-safe UTF-8 literals |
| `char16_t` (C++11) | 16 bits | UTF-16 | Cross-platform UTF-16 |
| `char32_t` (C++11) | 32 bits | UTF-32 | Single code point operations |

C++20 introduced `char8_t` and `u8string` for type-safe UTF-8. However, this created a breaking change: `u8"hello"` returns `const char8_t*` instead of `const char*`, breaking existing code that passed UTF-8 literals to legacy APIs. C++26 proposes `std::text_encoding` for runtime encoding detection.

Here is a quick map of the string type ecosystem - knowing which type lives where saves a lot of confusion at API boundaries:

```cpp
String type ecosystem:

  std::string      <- char,     locale-dependent (or UTF-8 if set)
  std::wstring     <- wchar_t,  UTF-16 on Windows, UTF-32 on Linux
  std::u8string    <- char8_t,  guaranteed UTF-8 (C++20)
  std::u16string   <- char16_t, guaranteed UTF-16
  std::u32string   <- char32_t, guaranteed UTF-32

  Win32 API: LPCWSTR (wchar_t*)  -- always UTF-16
  POSIX API: const char*         -- usually UTF-8 (locale-dependent)
```

The recommended strategy: use UTF-8 (`std::string` or `std::u8string`) as the internal representation, convert to the platform's native encoding only at API boundaries. On Windows, this means calling `MultiByteToWideChar`/`WideCharToMultiByte` or using `std::filesystem::path` (which handles the conversion internally).

---

## Self-Assessment

### Q1: Implement a UTF-8 <-> UTF-16 converter for Windows API interop without external dependencies

On Windows, the function you reach for is `MultiByteToWideChar` (UTF-8 to UTF-16) and `WideCharToMultiByte` (UTF-16 to UTF-8). Each is called twice: once with a null output buffer to get the required length, and again with the allocated buffer to do the actual conversion. On POSIX, `wchar_t` is 32-bit UTF-32, so the conversion is a manual UTF-8 decode loop. The portable `count_codepoints` function works on both platforms since it only touches the UTF-8 bytes:

```cpp
#include <string>
#include <string_view>
#include <stdexcept>
#include <iostream>
#include <cstdint>
#include <cstddef>

#ifdef _WIN32
    #define WIN32_LEAN_AND_MEAN
    #include <windows.h>
#endif

namespace unicode {

#ifdef _WIN32
// UTF-8 -> UTF-16 (Windows native)
std::wstring to_utf16(std::string_view utf8) {
    if (utf8.empty()) return {};

    int len = ::MultiByteToWideChar(
        CP_UTF8, MB_ERR_INVALID_CHARS,
        utf8.data(), static_cast<int>(utf8.size()),
        nullptr, 0
    );
    if (len <= 0) throw std::runtime_error("Invalid UTF-8 input");

    std::wstring result(static_cast<std::size_t>(len), L'\0');
    ::MultiByteToWideChar(
        CP_UTF8, MB_ERR_INVALID_CHARS,
        utf8.data(), static_cast<int>(utf8.size()),
        result.data(), len
    );
    return result;
}

// UTF-16 -> UTF-8
std::string to_utf8(std::wstring_view utf16) {
    if (utf16.empty()) return {};

    int len = ::WideCharToMultiByte(
        CP_UTF8, WC_ERR_INVALID_CHARS,
        utf16.data(), static_cast<int>(utf16.size()),
        nullptr, 0, nullptr, nullptr
    );
    if (len <= 0) throw std::runtime_error("Invalid UTF-16 input");

    std::string result(static_cast<std::size_t>(len), '\0');
    ::WideCharToMultiByte(
        CP_UTF8, WC_ERR_INVALID_CHARS,
        utf16.data(), static_cast<int>(utf16.size()),
        result.data(), len, nullptr, nullptr
    );
    return result;
}

#else
// POSIX: wchar_t is UTF-32, use standard C conversion or manual
std::wstring to_utf32(std::string_view utf8) {
    std::wstring result;
    result.reserve(utf8.size());

    std::size_t i = 0;
    while (i < utf8.size()) {
        char32_t cp;
        auto c = static_cast<unsigned char>(utf8[i]);

        int bytes;
        if      (c < 0x80)  { cp = c;          bytes = 1; }
        else if (c < 0xC0)  { throw std::runtime_error("Invalid UTF-8 continuation"); }
        else if (c < 0xE0)  { cp = c & 0x1F;   bytes = 2; }
        else if (c < 0xF0)  { cp = c & 0x0F;   bytes = 3; }
        else if (c < 0xF8)  { cp = c & 0x07;   bytes = 4; }
        else { throw std::runtime_error("Invalid UTF-8 lead byte"); }

        for (int j = 1; j < bytes; ++j) {
            if (i + j >= utf8.size())
                throw std::runtime_error("Truncated UTF-8 sequence");
            cp = (cp << 6) | (static_cast<unsigned char>(utf8[i + j]) & 0x3F);
        }
        i += bytes;
        result += static_cast<wchar_t>(cp);  // wchar_t is 32-bit on POSIX
    }
    return result;
}
#endif

// Portable: always available
// Count Unicode code points in a UTF-8 string
std::size_t count_codepoints(std::string_view utf8) {
    std::size_t count = 0;
    for (std::size_t i = 0; i < utf8.size(); ++count) {
        auto c = static_cast<unsigned char>(utf8[i]);
        if      (c < 0x80)  i += 1;
        else if (c < 0xE0)  i += 2;
        else if (c < 0xF0)  i += 3;
        else                 i += 4;
    }
    return count;
}

} // namespace unicode

int main() {
    std::string utf8_str = u8"Hello, 世界! 🌍";
    std::cout << "UTF-8 bytes: " << utf8_str.size() << "\n";
    std::cout << "Code points: " << unicode::count_codepoints(utf8_str) << "\n";

    #ifdef _WIN32
    auto wide = unicode::to_utf16(utf8_str);
    auto back = unicode::to_utf8(wide);
    std::cout << "Round-trip: " << back << "\n";
    std::cout << "UTF-16 units: " << wide.size() << "\n";
    #else
    auto wide = unicode::to_utf32(utf8_str);
    std::cout << "UTF-32 units: " << wide.size() << "\n";
    #endif

    return 0;
}
```

The byte count and code point count will differ for any string containing multi-byte characters - that difference is the whole reason careful encoding matters.

### Q2: Demonstrate the C++20 `char8_t` type and the interop challenges it creates with legacy APIs

This is an area where C++20 did the right thing for type safety but created real friction in the process. In C++17, `u8"hello"` had type `const char*`, which made it easy to pass to any API accepting a `char*`. In C++20, that same literal has type `const char8_t*`, and `char8_t` is a distinct type - you can no longer pass it directly to `std::cout` or to legacy `char*` APIs without an explicit conversion. The thin compatibility layer below bridges that gap:

```cpp
#include <string>
#include <string_view>
#include <iostream>
#include <filesystem>
#include <type_traits>

// The char8_t problem:
// In C++17: u8"hello" -> const char*       (convenient but lies about type)
// In C++20: u8"hello" -> const char8_t*    (correct but breaks everything)

// Conversion utilities for the C++17/C++20 boundary
namespace u8compat {

// char8_t string -> char string (same bytes, different type)
inline std::string from_u8string(const std::u8string& s) {
    return std::string(s.begin(), s.end());
}

inline std::string from_u8string(std::u8string_view sv) {
    return std::string(sv.begin(), sv.end());
}

// char string -> char8_t string (assumes valid UTF-8)
inline std::u8string to_u8string(std::string_view s) {
    return std::u8string(s.begin(), s.end());
}

// Create filesystem::path from UTF-8 char string (C++20-safe)
inline std::filesystem::path u8path(std::string_view utf8) {
    #if __cplusplus >= 202002L
    // C++20: u8string constructor
    std::u8string u8s(utf8.begin(), utf8.end());
    return std::filesystem::path(u8s);
    #else
    // C++17: deprecated but functional
    return std::filesystem::u8path(utf8);
    #endif
}

// Get UTF-8 string from path (C++20-safe)
inline std::string path_to_utf8(const std::filesystem::path& p) {
    #if __cplusplus >= 202002L
    auto u8 = p.u8string();  // Returns std::u8string in C++20
    return std::string(u8.begin(), u8.end());
    #else
    return p.u8string();     // Returns std::string in C++17
    #endif
}

} // namespace u8compat

// Safe output wrapper
void print_utf8(std::u8string_view text) {
    // std::cout cannot directly accept char8_t in C++20
    auto s = u8compat::from_u8string(text);
    std::cout << s;
}

int main() {
    // C++20 char8_t literals
    constexpr auto greeting = u8"Héllo, wörld! 🎉";
    static_assert(std::is_same_v<decltype(greeting), const char8_t(&)[22]>);

    // Cannot pass to legacy APIs directly:
    // std::cout << greeting;           // ERROR in C++20
    // std::string s = greeting;        // ERROR in C++20

    // Must convert explicitly
    print_utf8(greeting);
    std::cout << "\n";

    // Filesystem interop
    auto path = u8compat::u8path("/tmp/données/résultat.txt");
    std::cout << "Path: " << u8compat::path_to_utf8(path) << "\n";

    // u8string concatenation works as expected
    std::u8string base = u8"prefix_";
    std::u8string full = base + u8"données.txt";
    std::cout << "Filename: " << u8compat::from_u8string(full) << "\n";

    return 0;
}
```

The conversion is zero-cost at runtime since UTF-8 and `char8_t` use identical byte representations - you are just reinterpreting the type.

### Q3: Build a locale-aware text processing utility that handles normalization differences across platforms

This example shows the level of care required when processing real Unicode text without an external library. The `CodePointIterator` decodes UTF-8 byte sequences into `char32_t` code points one at a time. Notice that `count_visible_chars` skips combining diacritical marks - this matters because the same glyph can be represented in two ways (a precomposed character like `é`, or a base character `e` followed by a combining accent), and those two representations are not byte-equal even though they look identical:

```cpp
#include <string>
#include <string_view>
#include <algorithm>
#include <iostream>
#include <vector>
#include <cstdint>
#include <array>
#include <cstddef>

// UTF-8 iteration without external libraries
namespace utf8 {

struct CodePointIterator {
    const char* ptr;
    const char* end;

    char32_t operator*() const {
        auto c = static_cast<unsigned char>(*ptr);
        char32_t cp;
        int len;
        if      (c < 0x80)  { cp = c;        len = 1; }
        else if (c < 0xE0)  { cp = c & 0x1F; len = 2; }
        else if (c < 0xF0)  { cp = c & 0x0F; len = 3; }
        else                 { cp = c & 0x07; len = 4; }

        for (int i = 1; i < len && ptr + i < end; ++i)
            cp = (cp << 6) | (static_cast<unsigned char>(ptr[i]) & 0x3F);
        return cp;
    }

    CodePointIterator& operator++() {
        auto c = static_cast<unsigned char>(*ptr);
        if      (c < 0x80) ptr += 1;
        else if (c < 0xE0) ptr += 2;
        else if (c < 0xF0) ptr += 3;
        else               ptr += 4;
        if (ptr > end) ptr = end;
        return *this;
    }

    bool operator!=(const CodePointIterator& o) const { return ptr != o.ptr; }
};

struct CodePointRange {
    std::string_view data;
    CodePointIterator begin() const { return {data.data(), data.data() + data.size()}; }
    CodePointIterator end() const { return {data.data() + data.size(), data.data() + data.size()}; }
};

// Encode a single code point to UTF-8
std::string encode(char32_t cp) {
    std::string result;
    if (cp < 0x80) {
        result += static_cast<char>(cp);
    } else if (cp < 0x800) {
        result += static_cast<char>(0xC0 | (cp >> 6));
        result += static_cast<char>(0x80 | (cp & 0x3F));
    } else if (cp < 0x10000) {
        result += static_cast<char>(0xE0 | (cp >> 12));
        result += static_cast<char>(0x80 | ((cp >> 6) & 0x3F));
        result += static_cast<char>(0x80 | (cp & 0x3F));
    } else {
        result += static_cast<char>(0xF0 | (cp >> 18));
        result += static_cast<char>(0x80 | ((cp >> 12) & 0x3F));
        result += static_cast<char>(0x80 | ((cp >> 6) & 0x3F));
        result += static_cast<char>(0x80 | (cp & 0x3F));
    }
    return result;
}

} // namespace utf8

// ASCII case folding (full Unicode requires ICU)
namespace text {

std::string ascii_to_lower(std::string_view input) {
    std::string result;
    result.reserve(input.size());
    for (char32_t cp : utf8::CodePointRange{input}) {
        if (cp >= 'A' && cp <= 'Z')
            result += utf8::encode(cp + 32);
        else
            result += utf8::encode(cp);
    }
    return result;
}

// Validate UTF-8 encoding
bool is_valid_utf8(std::string_view s) {
    std::size_t i = 0;
    while (i < s.size()) {
        auto c = static_cast<unsigned char>(s[i]);
        int expected;
        if      (c < 0x80)  expected = 1;
        else if (c < 0xC2)  return false;  // Overlong or continuation
        else if (c < 0xE0)  expected = 2;
        else if (c < 0xF0)  expected = 3;
        else if (c < 0xF5)  expected = 4;
        else return false;

        if (i + expected > s.size()) return false;
        for (int j = 1; j < expected; ++j)
            if ((static_cast<unsigned char>(s[i + j]) & 0xC0) != 0x80) return false;
        i += expected;
    }
    return true;
}

// Count grapheme clusters (simplified - real impl needs UAX #29)
std::size_t count_visible_chars(std::string_view utf8) {
    std::size_t count = 0;
    for (char32_t cp : utf8::CodePointRange{utf8}) {
        // Skip combining marks (simplified: Unicode category M)
        if (cp >= 0x0300 && cp <= 0x036F) continue;  // Combining diacriticals
        if (cp >= 0x20D0 && cp <= 0x20FF) continue;  // Combining for symbols
        ++count;
    }
    return count;
}

} // namespace text

int main() {
    std::string s = u8"Ïñtérñàtiônàlizàtiôn";

    std::cout << "Valid UTF-8: " << std::boolalpha << text::is_valid_utf8(s) << "\n";
    std::cout << "Byte length: " << s.size() << "\n";
    std::cout << "Visible chars: " << text::count_visible_chars(s) << "\n";
    std::cout << "Lowercased: " << text::ascii_to_lower(s) << "\n";

    // Combining character example: é = e + U+0301 (combining acute)
    std::string combined = "e\xCC\x81";  // e + combining acute
    std::string precomposed = "\xC3\xA9"; // é precomposed
    std::cout << "Combined form:    '" << combined << "' (" << combined.size() << " bytes)\n";
    std::cout << "Precomposed form: '" << precomposed << "' (" << precomposed.size() << " bytes)\n";
    std::cout << "Byte-equal: " << (combined == precomposed) << "\n";
    // These look identical but are NOT byte-equal - normalization needed (NFC/NFD)

    return 0;
}
```

The last three lines are the most important takeaway: two strings that look identical to a human are not byte-equal because one uses NFC (precomposed) and the other NFD (decomposed with a combining mark). This is a real cross-platform hazard: macOS stores filenames in NFD, which means a file you created with an NFC name may not match a directory listing on macOS. Always normalize before comparing.

---

## Notes

- **UTF-8 everywhere**: Use UTF-8 as internal representation. Convert to UTF-16 only when calling Windows APIs. On Windows, call `SetConsoleOutputCP(CP_UTF8)` at startup for correct console output.
- `char8_t` (C++20) provides type safety but creates friction with existing APIs. Use a thin compatibility layer (`from_u8string`/`to_u8string`) during migration.
- `wchar_t` is unreliable for cross-platform code: it's 16-bit on Windows (UTF-16, which needs surrogate pairs) and 32-bit on POSIX (UTF-32). Avoid it in portable interfaces.
- String comparison, sorting, and case folding are locale-sensitive. For anything beyond ASCII, use ICU (`icu::UnicodeString`) or the platform's native API.
- macOS filenames are stored in NFD (decomposed) Unicode normalization. A file named "café" (NFC) may not match a directory listing that returns "café" (NFD) - always normalize before comparison.
- `std::text_encoding` (C++26) will finally provide runtime encoding detection, replacing the fragile `std::locale` approach.
