# Use std::filesystem for portable file system operations (C++17)

**Category:** Standard Library Utilities  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/filesystem>  

---

## Topic Overview

`std::filesystem` (in `<filesystem>`, namespace `std::filesystem` or `namespace fs = std::filesystem`) provides portable file/directory operations - listing, creating, copying, removing, querying - that work identically on Windows, Linux, and macOS.

Before C++17, portable file I/O meant either wrapping POSIX calls (which do not exist on Windows), using Boost.Filesystem, or writing your own platform-switching code. `std::filesystem` ended that. The same path arithmetic, directory iteration, and file queries now compile and work on every major platform.

### Key Types

| Type | Purpose |
| --- | --- |
| `fs::path` | Represents a file path with platform-aware separators |
| `fs::directory_entry` | An entry in a directory (file, dir, symlink) |
| `fs::directory_iterator` | Non-recursive directory listing |
| `fs::recursive_directory_iterator` | Recursive directory listing |
| `fs::file_status` | Type + permissions of a file |
| `std::error_code` | Non-throwing error reporting |

### Path Arithmetic

`fs::path` overloads `/=` and `/` so you can build paths by concatenating components without worrying about which separator character to use on which OS.

```cpp
namespace fs = std::filesystem;
fs::path p = "/home/user";
p /= "documents";           // p = "/home/user/documents"
p /= "report.pdf";          // p = "/home/user/documents/report.pdf"

p.filename();       // "report.pdf"
p.stem();           // "report"
p.extension();      // ".pdf"
p.parent_path();    // "/home/user/documents"
p.root_path();      // "/"
```

### Core Syntax

Here is a minimal tour: get the current directory, create a nested directory tree, check existence, query file size, and clean up.

```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

int main() {
    fs::path dir = fs::current_path();
    std::cout << "CWD: " << dir << "\n";

    // Create directories
    fs::create_directories("test/sub/deep");

    // Check existence
    if (fs::exists("test")) {
        std::cout << "test/ exists\n";
    }

    // File size
    if (fs::exists("main.cpp")) {
        std::cout << "main.cpp: " << fs::file_size("main.cpp") << " bytes\n";
    }

    // Remove
    fs::remove_all("test");  // recursive delete
}
```

---

## Self-Assessment

### Q1: Iterate a directory recursively and print all .cpp files using std::filesystem::recursive_directory_iterator

**Answer:**

The second loop in the example shows how to handle errors per-entry using `std::error_code`. This is the robust pattern for real code where you might hit permission-denied directories mid-traversal.

```cpp
#include <filesystem>
#include <iostream>
#include <vector>
#include <string>

namespace fs = std::filesystem;

int main() {
    fs::path search_dir = fs::current_path();  // or any path
    std::cout << "Searching in: " << search_dir << "\n\n";

    std::vector<fs::path> cpp_files;

    for (const auto& entry : fs::recursive_directory_iterator(search_dir)) {
        if (entry.is_regular_file() && entry.path().extension() == ".cpp") {
            cpp_files.push_back(entry.path());
            std::cout << entry.path().string() << "  ("
                      << entry.file_size() << " bytes)\n";
        }
    }

    std::cout << "\nTotal .cpp files: " << cpp_files.size() << "\n";

    // With error handling (skip permission-denied directories)
    std::error_code ec;
    for (auto it = fs::recursive_directory_iterator(
             search_dir,
             fs::directory_options::skip_permission_denied,
             ec);
         it != fs::recursive_directory_iterator();
         it.increment(ec))
    {
        if (ec) {
            std::cerr << "Error: " << ec.message() << "\n";
            ec.clear();
            continue;
        }
        if (it->is_regular_file() && it->path().extension() == ".h") {
            std::cout << "Header: " << it->path().filename() << "\n";
        }
    }
}
```

**Key points:**

- `recursive_directory_iterator` traverses all subdirectories automatically.
- `entry.is_regular_file()` filters out directories and symlinks.
- `.extension()` returns the file extension including the dot (`.cpp`).
- Use `directory_options::skip_permission_denied` to avoid exceptions on inaccessible directories.

---

### Q2: Use std::filesystem::path arithmetic to construct cross-platform paths

**Answer:**

Notice the `fs::relative` and `fs::canonical` utilities - these go beyond simple concatenation and let you normalize paths and compute relative relationships between them, which comes up constantly in build tools and project file management.

```cpp
#include <filesystem>
#include <iostream>
#include <string>

namespace fs = std::filesystem;

int main() {
    // --- Operator/ builds paths portably ---
    fs::path base = "/home/user";
    fs::path docs = base / "documents" / "reports";
    std::cout << docs << "\n";
    // Output (Linux): "/home/user/documents/reports"
    // Output (Windows): "/home/user\\documents\\reports"

    // --- Construct from Windows and convert ---
    fs::path win_path(R"(C:\Users\Alice\file.txt)");
    std::cout << "filename:  " << win_path.filename() << "\n";
    std::cout << "stem:      " << win_path.stem() << "\n";
    std::cout << "extension: " << win_path.extension() << "\n";
    std::cout << "parent:    " << win_path.parent_path() << "\n";
    // Output:
    // filename:  "file.txt"
    // stem:      "file"
    // extension: ".txt"
    // parent:    "C:\\Users\\Alice"

    // --- Replace parts ---
    fs::path p = "/project/src/main.cpp";
    p.replace_extension(".o");
    std::cout << "object file: " << p << "\n";
    // Output: "/project/src/main.o"

    p.replace_filename("utils.o");
    std::cout << "utils:       " << p << "\n";
    // Output: "/project/src/utils.o"

    // --- Relative path construction ---
    fs::path from = "/home/user/project/src";
    fs::path to   = "/home/user/project/include/header.h";
    fs::path rel  = fs::relative(to, from);
    std::cout << "relative: " << rel << "\n";
    // Output: "../include/header.h"

    // --- Canonical (resolve symlinks, .., .) ---
    fs::path messy = "/home/user/../user/./documents";
    std::error_code ec;
    fs::path clean = fs::canonical(messy, ec);
    if (!ec)
        std::cout << "canonical: " << clean << "\n";
    // Output: "/home/user/documents"

    // --- Iterate path components ---
    fs::path long_path = "/usr/local/bin/gcc";
    for (const auto& component : long_path) {
        std::cout << "[" << component << "] ";
    }
    std::cout << "\n";
    // Output: ["/"] ["usr"] ["local"] ["bin"] ["gcc"]
}
```

---

### Q3: Handle errors using std::error_code overloads to avoid exceptions in performance-critical paths

**Answer:**

Every `std::filesystem` function that can fail comes in two flavors: one that throws `fs::filesystem_error` and one that takes an `std::error_code` reference and sets it instead. Use the exception version when failure is unexpected; use the `error_code` version in loops or anywhere you expect some operations to fail.

```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

int main() {
    // --- Throwing version (default) ---
    try {
        auto size = fs::file_size("/nonexistent/file.txt");
        std::cout << size << "\n";
    } catch (const fs::filesystem_error& e) {
        std::cerr << "Exception: " << e.what() << "\n";
        std::cerr << "  path1: " << e.path1() << "\n";
        std::cerr << "  code:  " << e.code().message() << "\n";
    }

    // --- Non-throwing version: std::error_code overload ---
    std::error_code ec;
    auto size = fs::file_size("/nonexistent/file.txt", ec);
    if (ec) {
        std::cerr << "Error: " << ec.message() << "\n";
        // No exception thrown - suitable for hot paths
    } else {
        std::cout << "Size: " << size << "\n";
    }

    // --- Pattern: check-then-act with error_code ---
    fs::path target = "/tmp/myapp/data";

    // Create directory (non-throwing)
    fs::create_directories(target, ec);
    if (ec) {
        std::cerr << "Cannot create " << target << ": " << ec.message() << "\n";
        return 1;
    }

    // Copy file (non-throwing)
    fs::path src = "config.ini";
    fs::path dst = target / "config.ini";

    if (fs::exists(src, ec) && !ec) {
        fs::copy_file(src, dst, fs::copy_options::overwrite_existing, ec);
        if (ec) {
            std::cerr << "Copy failed: " << ec.message() << "\n";
        } else {
            std::cout << "Copied " << src << " -> " << dst << "\n";
        }
    }

    // --- Batch processing: error_code avoids try/catch overhead ---
    fs::path scan_dir = ".";
    for (auto it = fs::directory_iterator(scan_dir, ec);
         it != fs::directory_iterator();
         it.increment(ec))
    {
        if (ec) {
            std::cerr << "Iterator error: " << ec.message() << "\n";
            break;
        }
        auto fsize = fs::file_size(it->path(), ec);
        if (!ec) {
            std::cout << it->path().filename().string()
                      << ": " << fsize << " bytes\n";
        }
        // Skip files we can't stat - no exception, no crash
    }
}
```

**When to use which:**

- **Exceptions** (`fs::filesystem_error`): when failure is unexpected and should propagate up.
- **`std::error_code`**: in loops, hot paths, batch processing, or where you expect some operations to fail (e.g., listing directories with mixed permissions).

---

## Notes

- On GCC < 9, link with `-lstdc++fs`. On Clang/libc++ < 9, link with `-lc++fs`.
- `fs::path` uses `wchar_t` on Windows and `char` on POSIX. Use `.string()` for narrow string, `.wstring()` for wide.
- `fs::remove` deletes a single file/empty directory. `fs::remove_all` is recursive - use with caution.
- `fs::last_write_time` returns a `std::filesystem::file_time_type` - convert to `system_clock` via `std::chrono::clock_cast` (C++20).
- `fs::space()` returns disk space info (capacity, free, available).
- Always handle errors when working with the filesystem - permissions, missing files, and race conditions are common.
