# Manage Filesystem Path Differences Across Windows and POSIX

**Category:** Cross Platform Development  
**Standard:** C++17, C++20  
**Reference:** https://en.cppreference.com/w/cpp/filesystem/path  

---

## Topic Overview

`std::filesystem::path` (C++17) is the primary abstraction for portable path manipulation, but it papers over - rather than eliminates - fundamental differences between Windows and POSIX file systems. You really do need to understand these differences, because they cause subtle bugs that only show up in deployment on the other platform.

| Aspect | Windows (NTFS) | POSIX (ext4, APFS, etc.) |
| --- | --- | --- |
| Separator | `\` (preferred), `/` accepted | `/` only |
| `preferred_separator` | `L'\\'` | `'/'` |
| Root name | `C:`, `\\server\share` | (empty) |
| Max path | 260 by default; 32,767 with `\\?\` prefix | 4096 (`PATH_MAX`) |
| Case sensitivity | Case-insensitive (preserving) | Case-sensitive |
| Native string type | `std::wstring` (UTF-16) | `std::string` (bytes, usually UTF-8) |
| Forbidden chars | `< > : " \| ? *` and NUL | NUL and `/` |
| Trailing dot/space | Silently stripped by Win32 API | Valid characters |

The `path::native()` method returns a reference to the platform-native string (`std::wstring` on Windows, `std::string` on POSIX). One trap to watch out for: using `.string()` on Windows performs a lossy conversion if the path contains characters outside the current code page. Prefer `.u8string()` or `.wstring()` for lossless round-trips.

Long path support on Windows requires either a manifest entry (`longPathAware = true` in the application manifest plus a registry setting) or the `\\?\` prefix for raw Win32 calls. `std::filesystem` implementations on MSVC transparently handle the prefix in recent versions when the OS setting is enabled.

Here is a quick reference showing how `std::filesystem::path` breaks a Windows path into its components:

```cpp
Path decomposition:  C:\Users\dev\project\src\main.cpp

  root_name()      -> "C:"
  root_directory()  -> "\"
  root_path()       -> "C:\"
  relative_path()   -> "Users\dev\project\src\main.cpp"
  parent_path()     -> "C:\Users\dev\project\src"
  filename()        -> "main.cpp"
  stem()            -> "main"
  extension()       -> ".cpp"
```

---

## Self-Assessment

### Q1: Write a function that normalizes a path for cross-platform use, handling separator differences, trailing separators, and relative resolution

The goal here is to take a messy, potentially relative path with redundant components and turn it into a clean, canonical form. Notice how `lexically_normal()` handles the `..` and `.` parts without ever touching the filesystem, and `make_preferred()` applies the OS-native separator as a final step.

```cpp
#include <filesystem>
#include <iostream>
#include <string>

namespace fs = std::filesystem;

// Normalize: resolve ".." and ".", make preferred separators, remove trailing sep
fs::path normalize_path(const fs::path& input, const fs::path& base = fs::current_path()) {
    fs::path absolute;

    if (input.is_relative()) {
        absolute = base / input;
    } else {
        absolute = input;
    }

    // lexically_normal resolves "." and ".." without touching the filesystem
    fs::path normalized = absolute.lexically_normal();

    // make_preferred converts to platform-native separators
    normalized.make_preferred();

    return normalized;
}

// Convert to a generic (POSIX-style) string for config files / serialization
std::string to_generic(const fs::path& p) {
    return p.generic_string(); // Always uses '/' separators
}

// Cross-platform relative path computation
fs::path make_relative(const fs::path& target, const fs::path& base) {
    return target.lexically_relative(base);
}

int main() {
    fs::path messy = "src/../src/./core//engine.cpp";
    auto clean = normalize_path(messy, "/home/dev/project");
    std::cout << "Normalized:  " << clean << "\n";
    std::cout << "Generic:     " << to_generic(clean) << "\n";

    auto rel = make_relative("/home/dev/project/src/core/engine.cpp",
                             "/home/dev/project");
    std::cout << "Relative:    " << rel << "\n";

    // Demonstrate preferred_separator
    std::cout << "Preferred separator: '"
              << fs::path::preferred_separator << "'\n";
    return 0;
}
```

The output you get from `generic_string()` will always use forward slashes regardless of the platform - that is the format you want when writing paths to config files or JSON, since the consuming code can always handle `/` even on Windows.

### Q2: Demonstrate safe path handling that accounts for Windows long paths, encoding differences, and case sensitivity

This is where the platform differences really bite you. The `PortablePath` wrapper below centralizes three concerns that are easy to get wrong separately: long-path retries on Windows, case-insensitive comparison on platforms that need it, and the UTF-8 round-trip gotcha introduced in C++20.

```cpp
#include <filesystem>
#include <iostream>
#include <algorithm>
#include <string>
#include <locale>

namespace fs = std::filesystem;

class PortablePath {
    fs::path path_;
public:
    explicit PortablePath(const fs::path& p) : path_(p.lexically_normal()) {}

    // Platform-safe existence check that handles long paths
    bool safe_exists() const {
        std::error_code ec;
        bool result = fs::exists(path_, ec);
        if (ec) {
            #ifdef _WIN32
            // Retry with \\?\ prefix for long paths on Windows
            auto native = path_.native();
            if (native.size() > 260 && native.substr(0, 4) != L"\\\\?\\") {
                fs::path extended = L"\\\\?\\" + native;
                return fs::exists(extended, ec);
            }
            #endif
            return false;
        }
        return result;
    }

    // Case-insensitive comparison (needed on Windows, optional on macOS HFS+)
    static bool paths_equivalent_ci(const fs::path& a, const fs::path& b) {
        auto sa = a.lexically_normal().generic_string();
        auto sb = b.lexically_normal().generic_string();
        #if defined(_WIN32) || defined(__APPLE__)
        // Case-insensitive on Windows and default macOS
        std::transform(sa.begin(), sa.end(), sa.begin(), ::tolower);
        std::transform(sb.begin(), sb.end(), sb.begin(), ::tolower);
        #endif
        return sa == sb;
    }

    // Safe UTF-8 round-trip
    std::string to_utf8() const {
        #if __cplusplus >= 202002L
        auto u8 = path_.u8string();
        return std::string(u8.begin(), u8.end()); // char8_t -> char
        #else
        return path_.u8string();
        #endif
    }

    static PortablePath from_utf8(const std::string& s) {
        #if __cplusplus >= 202002L
        std::u8string u8s(s.begin(), s.end());
        return PortablePath(fs::path(u8s));
        #else
        return PortablePath(fs::u8path(s));
        #endif
    }

    const fs::path& get() const { return path_; }
};

int main() {
    PortablePath p = PortablePath::from_utf8("/tmp/données/fichier.txt");
    std::cout << "UTF-8: " << p.to_utf8() << "\n";
    std::cout << "Exists: " << std::boolalpha << p.safe_exists() << "\n";

    bool equiv = PortablePath::paths_equivalent_ci(
        "C:/Users/Dev/Project",
        "c:/users/dev/project"
    );
    std::cout << "Case-insensitive equal: " << equiv << "\n";
    return 0;
}
```

The reason the `to_utf8()` code has a C++20 branch is that `u8string()` changed its return type from `std::string` to `std::u8string` in C++20, which silently breaks code that compiled fine in C++17. The explicit cast through `begin()`/`end()` is the currently accepted workaround.

### Q3: Build a path sanitizer that rejects invalid characters per platform, validates length limits, and produces safe filenames

When you accept a filename from user input or a network protocol, you cannot trust it to be valid on the current platform. This sanitizer encodes the platform rules in one place so the rest of your code does not have to think about them.

```cpp
#include <filesystem>
#include <string>
#include <string_view>
#include <algorithm>
#include <stdexcept>
#include <iostream>
#include <array>

namespace fs = std::filesystem;

struct PathLimits {
    static constexpr std::size_t max_filename =
        #ifdef _WIN32
        255;       // NTFS limit per component
        #else
        255;       // NAME_MAX on most POSIX systems
        #endif

    static constexpr std::size_t max_path =
        #ifdef _WIN32
        32'767;    // With \\?\ prefix; 260 without
        #else
        4'096;     // PATH_MAX
        #endif
};

class PathSanitizer {
    static constexpr std::string_view win_forbidden = R"(<>:"|?*)";
    static constexpr std::array<std::string_view, 22> win_reserved = {
        "CON","PRN","AUX","NUL",
        "COM1","COM2","COM3","COM4","COM5","COM6","COM7","COM8","COM9",
        "LPT1","LPT2","LPT3","LPT4","LPT5","LPT6","LPT7","LPT8","LPT9"
    };

public:
    // Strip or replace characters invalid on the target platform
    static std::string sanitize_filename(std::string_view input,
                                         char replacement = '_') {
        std::string result;
        result.reserve(input.size());

        for (char c : input) {
            if (c == '\0' || c == '/') {
                result += replacement;
            }
            #ifdef _WIN32
            else if (c == '\\' || win_forbidden.find(c) != std::string_view::npos) {
                result += replacement;
            }
            #endif
            else if (static_cast<unsigned char>(c) < 32) {
                result += replacement; // Control characters
            } else {
                result += c;
            }
        }

        // Windows: strip trailing dots and spaces
        #ifdef _WIN32
        while (!result.empty() && (result.back() == '.' || result.back() == ' '))
            result.pop_back();
        #endif

        if (result.empty()) result = "unnamed";

        // Truncate to max filename length (UTF-8 byte count as proxy)
        if (result.size() > PathLimits::max_filename)
            result.resize(PathLimits::max_filename);

        return result;
    }

    // Validate a full path
    struct ValidationResult {
        bool   valid;
        std::string reason;
    };

    static ValidationResult validate_path(const fs::path& p) {
        auto str = p.generic_string();
        if (str.size() > PathLimits::max_path)
            return {false, "Path exceeds maximum length"};

        for (const auto& component : p) {
            auto name = component.string();
            if (name == "/" || name == "\\" || name.find(':') != std::string::npos)
                continue; // root components
            if (name.size() > PathLimits::max_filename)
                return {false, "Component '" + name + "' exceeds max filename length"};
        }
        return {true, {}};
    }
};

int main() {
    std::string dangerous = "CON.txt";
    std::cout << "Sanitized: " << PathSanitizer::sanitize_filename(dangerous) << "\n";

    std::string bad_chars = "file<name>:v2|test?.cpp";
    std::cout << "Sanitized: " << PathSanitizer::sanitize_filename(bad_chars) << "\n";

    auto [valid, reason] = PathSanitizer::validate_path("/normal/path/file.txt");
    std::cout << "Valid: " << std::boolalpha << valid << "\n";

    return 0;
}
```

Notice that `CON.txt` on Windows is a reserved device name even with an extension - the Win32 API will redirect it to the console device, not a file. Your sanitizer needs to know about all 22 of these reserved names, not just `NUL`.

---

## Notes

- Always use the `std::error_code` overloads of `std::filesystem` functions in production - the throwing variants produce cryptic messages and are hard to recover from gracefully.
- `path::u8string()` returns `std::u8string` in C++20 (not `std::string`), which breaks code upgrading from C++17. The cast through `std::string(u8.begin(), u8.end())` is the current workaround.
- `fs::equivalent()` is the only reliable way to compare two paths that may be symlinks or junctions - lexical comparison fails for aliased paths.
- On Windows, forward slashes work in most Win32 APIs, but not in the `\\?\` long-path prefix or in some shell operations.
- `lexically_normal()` does not touch the filesystem; `canonical()` does (and throws if the path does not exist). Use `weakly_canonical()` for partial resolution when the path may not yet exist.
- macOS HFS+/APFS uses Unicode normalization form D for filenames; a file created as "café" may not be found by searching for the NFC-normalized version.
