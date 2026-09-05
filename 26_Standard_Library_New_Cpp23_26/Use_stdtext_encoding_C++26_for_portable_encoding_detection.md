# Use std::text_encoding (C++26) for portable encoding detection

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/locale/text_encoding>  

---

## Topic Overview

This file focuses on the **compile-time vs runtime encoding distinction** and building a portable UTF-8 input handler. (See also file #762 for core `std::text_encoding` API and Windows-specific examples.)

### Two Encodings in Every Program

This is a concept that trips up a lot of developers: your program actually has *two* encodings to worry about, and they can be different from each other. One is baked in at compile time (what encoding are string literals stored in the binary?), and the other is determined at runtime (what encoding does the current system expect when you write to the console or a file?).

| Encoding                          | When decided  | API                                     |
| --------------------------------- | ------------- | --------------------------------------- |
| **Literal encoding**              | Compile time  | `std::text_encoding::literal()`         |
| **System/environment encoding**   | Runtime       | `std::text_encoding::environment()`     |

When these two disagree, you get mojibake - garbled output where the bytes you wrote don't match what the terminal expected to receive. The diagram shows exactly how this happens:

```cpp
Source code: const char* s = "hello café";
                                    │
Compiler flag (-fexec-charset=utf-8)│
                                    ▼
Binary contains: UTF-8 bytes [68 65 6c 6c 6f 20 63 61 66 c3 a9]
                                    │
Runtime: console uses windows-1252  │
                                    ▼
Display: garbled "cafÃ©" unless converted!
```

`std::text_encoding` gives you a standard way to query both sides of this at runtime and at compile time, so you can detect mismatches and act accordingly.

---

## Self-Assessment

### Q1: Use std::text_encoding::environment() to detect the system's narrow character encoding at runtime

**Answer:**

`environment()` returns a `text_encoding` object that describes the encoding the current system uses for narrow strings - essentially, what the OS locale or Windows code page resolves to. You can query it by MIB enum value, by name, or compare it directly to a named encoding:

<!-- compile: needs `text_encoding` (not in the CI standard library yet) -->
```cpp
#include <text_encoding>  // C++26
#include <iostream>
#include <print>

int main() {
    // Detect system encoding
    auto sys = std::text_encoding::environment();
    std::println("System encoding: {} (MIB: {})",
                 sys.name(), static_cast<int>(sys.mib()));

    // Common MIB checks
    switch (sys.mib()) {
        case std::text_encoding::id::UTF8:
            std::println("System uses UTF-8 - no conversion needed");
            break;
        case std::text_encoding::id::ISO8859_1:
            std::println("System uses ISO-8859-1 (Latin-1)");
            break;
        case std::text_encoding::id::unknown:
            std::println("Unknown encoding - proceed with caution");
            break;
        default:
            std::println("System uses: {}", sys.name());
            break;
    }

    // Check specific encoding by name
    auto expected = std::text_encoding("UTF-8");
    if (sys == expected) {
        std::println("Confirmed: system is UTF-8");
    }

    // aliases() - all known names for this encoding
    std::println("Known aliases:");
    for (auto alias : sys.aliases()) {
        std::println("  - {}", alias);
    }
    // e.g., for UTF-8: "UTF-8", "utf-8", "csUTF8"
}
```

The MIB values come from the IANA character set registry. `UTF8`, `ISO8859_1`, and `unknown` cover the most common cases you'll encounter in practice. The `aliases()` method is useful when you need to pass the encoding name to a third-party library like iconv that may expect a specific alias.

---

### Q2: Explain the difference between literal encoding (compile time) and system encoding (runtime)

**Answer:**

The reason this distinction matters is that the same program binary can be deployed to machines with different locale settings. The bytes in your string literals are fixed at compile time, but whether those bytes make sense to the receiving system depends on what encoding that system expects at runtime.

<!-- compile: needs `text_encoding` (not in the CI standard library yet) -->
```cpp
#include <text_encoding>  // C++26
#include <iostream>
#include <print>

int main() {
    // Literal encoding: decided at COMPILE TIME
    auto lit = std::text_encoding::literal();
    std::println("Literal encoding: {}", lit.name());
    // Determined by:
    //   MSVC:  /utf-8 or /source-charset:utf-8 /execution-charset:utf-8
    //   GCC:   -fexec-charset=utf-8 (default is UTF-8 since GCC 10)
    //   Clang: -fexec-charset=utf-8 (default is UTF-8)

    // This affects how string literals are encoded in the binary:
    const char* greeting = "café";
    // With -fexec-charset=utf-8:  bytes = 63 61 66 c3 a9 (5 bytes, UTF-8)
    // With -fexec-charset=latin1: bytes = 63 61 66 e9    (4 bytes, Latin-1)

    // System encoding: decided at RUNTIME
    auto sys = std::text_encoding::environment();
    std::println("System encoding:  {}", sys.name());
    // Determined by:
    //   Linux:   LC_CTYPE locale (usually UTF-8)
    //   Windows: GetACP() -> codepage (often 1252 or 65001)
    //   macOS:   UTF-8 (always)

    // The mismatch problem
    if (lit != sys) {
        std::println("MISMATCH: literal={} vs system={}",
                     lit.name(), sys.name());
        std::println("  String literals will display incorrectly without conversion!");

        // Example scenario:
        // Compiled on Linux (UTF-8 literals)
        // Running on Windows (system = windows-1252)
        // -> "café" stored as UTF-8 in binary
        // -> Console expects windows-1252
        // -> "cafÃ©" displayed (mojibake!)

        // Solution: detect at startup and configure conversion
    } else {
        std::println("Match: both literal and system use {}", lit.name());
    }

    /*
    Summary:
    ┌─────────────────┬───────────────────┬──────────────────────────┐
    │                 │ Literal encoding  │ System encoding          │
    ├─────────────────┼───────────────────┼──────────────────────────┤
    │ When decided?   │ Compile time      │ Runtime                  │
    │ Changed by?     │ Compiler flags    │ OS locale / env vars     │
    │ Affects?        │ String literals   │ Console, file I/O, APIs  │
    │ Can differ?     │ YES - this causes │ encoding mismatch bugs!  │
    │ Queried via?    │ literal()         │ environment()            │
    └─────────────────┴───────────────────┴──────────────────────────┘
    */
}
```

The reason this trips people up is that on Linux and macOS everything is almost always UTF-8 end-to-end, so the mismatch never manifests. On Windows, especially in legacy projects, the system codepage is often 1252 even when the compiler is generating UTF-8 literals. `std::text_encoding` lets you detect and respond to this at runtime rather than relying on documentation or tribal knowledge.

---

### Q3: Show a portable UTF-8 input handler that converts from the system encoding using text_encoding

**Answer:**

The key limitation to know upfront: `std::text_encoding` *detects* encodings but does *not convert* between them. The conversion itself still requires a platform API (Win32's `MultiByteToWideChar`/`WideCharToMultiByte`) or a library like iconv. What `text_encoding` gives you is a portable way to *decide* whether conversion is needed:

<!-- compile: needs `text_encoding` (not in the CI standard library yet) -->
```cpp
#include <text_encoding>  // C++26
#include <iostream>
#include <string>
#include <print>
#include <stdexcept>

// Portable UTF-8 input handler
// Reads input from stdin, converts to UTF-8 if system encoding differs

class Utf8InputHandler {
    bool needs_conversion_;
    std::string sys_encoding_;

public:
    Utf8InputHandler() {
        auto sys = std::text_encoding::environment();
        auto target = std::text_encoding("UTF-8");
        needs_conversion_ = (sys != target);
        sys_encoding_ = sys.name();

        if (needs_conversion_) {
            std::println(std::cerr, "Note: system encoding is {} - will convert to UTF-8",
                         sys_encoding_);
        }
    }

    // Read a line and return it as UTF-8
    std::string read_line_utf8() {
        std::string line;
        if (!std::getline(std::cin, line))
            return {};

        if (!needs_conversion_)
            return line;  // Already UTF-8, no conversion needed

        return convert_to_utf8(line);
    }

private:
    std::string convert_to_utf8(const std::string& input) {
        // Platform-specific conversion
        // In production, use ICU, iconv, or OS APIs
#ifdef _WIN32
        // Windows: MultiByteToWideChar(CP_ACP) -> WideCharToMultiByte(CP_UTF8)
        // Pseudocode:
        // int wide_len = MultiByteToWideChar(CP_ACP, 0, input.c_str(), -1, nullptr, 0);
        // std::wstring wide(wide_len, 0);
        // MultiByteToWideChar(CP_ACP, 0, input.c_str(), -1, wide.data(), wide_len);
        // int utf8_len = WideCharToMultiByte(CP_UTF8, 0, wide.c_str(), -1, nullptr, 0, ...);
        // std::string utf8(utf8_len, 0);
        // WideCharToMultiByte(CP_UTF8, 0, wide.c_str(), -1, utf8.data(), utf8_len, ...);
        // return utf8;
        return input;  // Simplified
#else
        // POSIX: use iconv
        // iconv_t cd = iconv_open("UTF-8", sys_encoding_.c_str());
        // ... iconv(cd, &inbuf, &inlen, &outbuf, &outlen) ...
        // iconv_close(cd);
        return input;  // On Linux/macOS, usually already UTF-8
#endif
    }
};

int main() {
    Utf8InputHandler handler;

    std::println("Enter text (will be read as UTF-8):");
    std::string line = handler.read_line_utf8();
    std::println("UTF-8 input: {}", line);
    std::println("Byte length: {}", line.size());

    // Verify UTF-8: check for valid byte sequences
    bool valid = true;
    for (size_t i = 0; i < line.size(); ++i) {
        unsigned char c = static_cast<unsigned char>(line[i]);
        if (c >= 0x80) {
            // Multi-byte UTF-8: verify continuation bytes
            int expected = (c >= 0xF0) ? 3 : (c >= 0xE0) ? 2 : (c >= 0xC0) ? 1 : -1;
            if (expected < 0) { valid = false; break; }
            for (int j = 0; j < expected && i + 1 < line.size(); ++j) {
                if ((static_cast<unsigned char>(line[++i]) & 0xC0) != 0x80) {
                    valid = false; break;
                }
            }
        }
    }
    std::println("Valid UTF-8: {}", valid ? "yes" : "no");
}
```

The `needs_conversion_` flag set in the constructor is the portable part. The actual conversion in `convert_to_utf8` is necessarily platform-specific - `std::text_encoding` can't change that. What it does change is that the *detection* logic no longer needs `#ifdef` blocks or platform-specific API calls.

---

## Notes

- `std::text_encoding` detects encodings but does not convert between them - conversion still requires ICU, iconv, or OS APIs.
- Always compile with `/utf-8` (MSVC) or `-fexec-charset=utf-8` (GCC/Clang) to ensure the literal encoding is UTF-8 and eliminate one source of mismatch.
- `text_encoding::environment()` may return different results based on `LC_CTYPE`, `LANG`, or the Windows active code page.
- C++26 also provides `std::text_encoding::wide_literal()` for querying the wide string literal encoding.
- On modern Linux and macOS, `environment()` almost always returns UTF-8; Windows is the main edge case to defend against.
