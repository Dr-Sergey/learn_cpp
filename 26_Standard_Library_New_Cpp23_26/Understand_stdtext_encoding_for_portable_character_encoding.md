# Understand std::text_encoding for portable character encoding

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/locale/text_encoding>  

---

## Topic Overview

`std::text_encoding` (C++26, `<text_encoding>`) provides a **portable way to query and identify character encodings** at runtime. Before this, detecting the system's encoding required platform-specific APIs - `GetACP()` on Windows, `nl_langinfo(CODESET)` on POSIX. With `std::text_encoding`, C++ programs can discover encodings portably and make informed decisions about string conversion.

The reason this matters in practice is that encoding mismatches are one of the most common sources of silent data corruption in cross-platform code. A string that looks fine on Linux (UTF-8 everywhere) can display as garbage on a Windows machine whose console uses windows-1252. `std::text_encoding` gives you the tools to detect this mismatch at runtime and handle it explicitly, rather than hoping for the best.

### Key API

| Expression | Description |
| --- | --- |
| `std::text_encoding::literal()` | Encoding of string literals (set by compiler flags) |
| `std::text_encoding::environment()` | System's runtime encoding (locale-dependent) |
| `enc.name()` | IANA name string ("UTF-8", "windows-1252", etc.) |
| `enc.mib()` | IANA MIB enum identifier |
| `enc == other` | Compare two encodings for equality |

### Common Encoding MIBs

These are the identifiers you will encounter most often when writing cross-platform string-handling code:

```cpp
UTF-8            -> MIB 106
US-ASCII         -> MIB 3
ISO-8859-1       -> MIB 4
windows-1252     -> MIB 2252
Shift_JIS        -> MIB 17
```

---

## Self-Assessment

### Q1: Use std::text_encoding::environment() to query the system's default encoding at runtime

This is the most common starting point: query both what the compiler embedded in your string literals and what the system currently expects, then decide if they match:

<!-- compile: needs `text_encoding` (not in the CI standard library yet) -->
```cpp
// C++26 - requires compiler/library support for <text_encoding>
#include <text_encoding>
#include <iostream>
#include <print>

int main() {
    // Query the system's runtime encoding
    auto sys_enc = std::text_encoding::environment();
    std::println("System encoding: {}", sys_enc.name());
    std::println("MIB identifier:  {}", static_cast<int>(sys_enc.mib()));

    // Query the encoding used for string literals (compile-time decision)
    auto lit_enc = std::text_encoding::literal();
    std::println("Literal encoding: {}", lit_enc.name());

    // Check if they match
    if (sys_enc == lit_enc) {
        std::println("System and literal encodings match - no conversion needed");
    } else {
        std::println("WARNING: System ({}) != Literal ({}) - conversion required!",
                      sys_enc.name(), lit_enc.name());
    }

    // Check for specific encodings
    if (sys_enc.mib() == std::text_encoding::id::UTF8) {
        std::println("System uses UTF-8 - good to go");
    }
}

// Possible output on Linux (UTF-8 locale):
//   System encoding: UTF-8
//   MIB identifier:  106
//   Literal encoding: UTF-8
//   System and literal encodings match - no conversion needed
//   System uses UTF-8 - good to go

// Possible output on Windows (default locale):
//   System encoding: windows-1252
//   MIB identifier:  2252
//   Literal encoding: UTF-8
//   WARNING: System (windows-1252) != Literal (UTF-8) - conversion required!
```

On Linux you almost always see UTF-8 for both, which is why encoding bugs are often "only reproducible on Windows." This API lets you surface that mismatch programmatically.

### Q2: Explain the role of text_encoding in interfacing between UTF-8 internal strings and platform APIs

Modern C++ programs typically use **UTF-8 internally** (especially with `-fexec-charset=utf-8` or `/utf-8`). But platform APIs may expect different encodings. `std::text_encoding` gives you a portable way to make the conversion decision without sprinkling `#ifdef _WIN32` everywhere:

```cpp
// Problem scenario: Your program stores UTF-8 strings internally.
// On Windows (non-UTF-8 locale), file paths go through ANSI APIs expecting windows-1252.

#include <text_encoding>
#include <string>
#include <iostream>

// Portable conversion decision based on runtime encoding detection
enum class ConversionNeeded { None, Utf8ToSystem, SystemToUtf8 };

ConversionNeeded check_conversion_need() {
    auto sys = std::text_encoding::environment();
    auto lit = std::text_encoding::literal();

    if (sys == lit) return ConversionNeeded::None;

    // Literal is UTF-8 but system is something else
    if (lit.mib() == std::text_encoding::id::UTF8)
        return ConversionNeeded::Utf8ToSystem;

    return ConversionNeeded::SystemToUtf8;
}

// Use case: safely writing user-visible strings to console
void safe_print(const std::string& utf8_str) {
    auto need = check_conversion_need();
    switch (need) {
        case ConversionNeeded::None:
            std::cout << utf8_str;  // Direct output is safe
            break;
        case ConversionNeeded::Utf8ToSystem:
            // Convert UTF-8 -> system encoding before output
            // (Use iconv, ICU, or Windows WideCharToMultiByte)
            std::cout << "[converted] " << utf8_str;
            break;
        case ConversionNeeded::SystemToUtf8:
            std::cout << utf8_str;  // Already UTF-8 internally
            break;
    }
}

/*
Flow diagram:
    Internal string (UTF-8)
         |
         v
    text_encoding::environment() == UTF-8?
         |                |
        YES              NO
         |                |
    Output directly   Convert UTF-8 -> system encoding
                      (windows-1252, Shift_JIS, etc.)
         |                |
         +---- Platform API (console, file system, network) ----+
*/
```

The key mental model here: `literal()` tells you what encoding was baked into your source at compile time, and `environment()` tells you what the system expects right now at runtime. When those two differ, you know you need a conversion layer.

### Q3: Show a Windows-specific example where the system encoding is not UTF-8 and text_encoding detects it

This example shows the real-world Windows scenario that trips up many developers: you compiled with `/utf-8` so your string literals are UTF-8, but the Windows console is using an ANSI code page. `std::text_encoding` makes this mismatch explicit:

<!-- compile: needs `text_encoding` (not in the CI standard library yet) -->
```cpp
#include <text_encoding>
#include <iostream>
#include <print>

#ifdef _WIN32
#include <windows.h>  // For SetConsoleOutputCP
#endif

int main() {
    auto sys = std::text_encoding::environment();
    auto lit = std::text_encoding::literal();

    std::println("System encoding: {} (MIB {})", sys.name(),
                 static_cast<int>(sys.mib()));
    std::println("Literal encoding: {} (MIB {})", lit.name(),
                 static_cast<int>(lit.mib()));

#ifdef _WIN32
    // On Windows with default locale, sys is typically windows-1252 (MIB 2252)
    // while /utf-8 flag makes literals UTF-8 (MIB 106)
    if (sys.mib() != std::text_encoding::id::UTF8) {
        std::println("System is NOT UTF-8 - enabling UTF-8 console output");
        SetConsoleOutputCP(CP_UTF8);

        // After: re-query shows system still reports ANSI codepage,
        // but console now accepts UTF-8
        std::println("Console codepage set to UTF-8 (65001)");
    }

    // Demonstrate the encoding mismatch problem:
    // Without conversion, non-ASCII chars display incorrectly
    std::string utf8_str = u8"Nono - cafe";  // UTF-8 encoded
    std::println("UTF-8 string: {}", utf8_str);
    // With SetConsoleOutputCP(CP_UTF8): displays correctly
    // Without: shows garbled characters
#else
    // On Linux/macOS, system encoding is almost always UTF-8
    if (sys.mib() == std::text_encoding::id::UTF8) {
        std::println("System uses UTF-8 - no Windows-style mismatch");
    }
#endif
}

// Output on Windows (default English locale, compiled with /utf-8):
//   System encoding: windows-1252 (MIB 2252)
//   Literal encoding: UTF-8 (MIB 106)
//   System is NOT UTF-8 - enabling UTF-8 console output
//   Console codepage set to UTF-8 (65001)
//   UTF-8 string: Nono - cafe
```

---

## Notes

- `std::text_encoding` is C++26; early support in libc++ and MSVC STL.
- Use `/utf-8` (MSVC) or `-fexec-charset=utf-8` (GCC/Clang) to make `literal()` return UTF-8.
- `text_encoding` identifies encodings but does **not** convert between them - use ICU, iconv, or OS APIs for that.
- On Windows, `SetConsoleOutputCP(CP_UTF8)` combined with wide API (`_wfopen`) is the pragmatic approach until full UTF-8 locale support arrives.
- IANA maintains the canonical registry of encoding names and MIB numbers.
