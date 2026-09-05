# Understand how global module fragments work for macro dependencies

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/modules>  

---

## Topic Overview

This topic focuses on **practical macro dependency scenarios** - wrapping C APIs, conditional compilation, and platform-specific includes that require macros inside module code.

The core problem is simple: macros are a preprocessor-level concept, and the module system lives at the compiler level. You can't mix them freely. The **global module fragment** is the designated escape hatch that lets you `#include` legacy C headers (with all their macros) before the module declaration takes effect.

### Common Macro Dependencies That Require Global Module Fragments

| Dependency | Header | Macros needed |
| --- | --- | --- |
| POSIX API | `<unistd.h>`, `<fcntl.h>` | `O_RDONLY`, `S_IRUSR`, etc. |
| Windows API | `<windows.h>` | `NOMINMAX`, `WIN32_LEAN_AND_MEAN` |
| OpenGL | `<GL/gl.h>` | `GL_TRIANGLES`, `GL_FLOAT` |
| SQLite | `<sqlite3.h>` | `SQLITE_OK`, `SQLITE_ROW` |
| errno | `<cerrno>` | `errno`, `ENOENT`, `EACCES` |
| Configuration | `"config.h"` | `#define`-based feature flags |

### Why Macros Can't Live After `export module`

Here's the situation that catches people off guard. If you put a `#include` of a C header after your `export module` declaration, the macros that header defines may not behave as expected:

```cpp
export module wrapper;

// This is implementation-defined and likely won't work as expected:
#include <cerrno>       // BAD: macros may not be available after module declaration

export int get_errno() {
    return errno;       // errno is a macro - may fail!
}
```

The reason this trips people up is that the module purview (everything after `export module`) is under the module system's control. Including headers there is technically allowed by some compilers, but macro availability is implementation-defined and not reliable.

The correct approach is to use the global module fragment, which is the region before the module declaration. Start the file with a bare `module;` line, include your C headers there, then declare the module:

```cpp
module;                 // <- global module fragment begins here
#include <cerrno>       // GOOD: macros are guaranteed available

export module wrapper;  // global module fragment ends here

export int get_errno() {
    return errno;       // GOOD: macro was included in fragment
}
```

Everything between the bare `module;` and the `export module` declaration is the global module fragment. Preprocessor directives there work exactly as they always have.

---

## Self-Assessment

### Q1: Use `module; ... export module foo;` to include legacy headers whose macros are needed

Here's a realistic example: a module that wraps POSIX file operations. All the C macros (`errno`, `ENOENT`, `EACCES`, platform path separator) go in the global fragment, while the exported interface exposes only clean, type-safe C++ abstractions:

```cpp
// file_wrapper.cppm - wraps POSIX file operations as a C++20 module
module;

// Global module fragment: include C headers with macros
#include <cstdio>      // FILE*, fopen, fclose, fread, etc.
#include <cerrno>      // errno macro
#include <cstring>     // strerror

// Platform detection (preprocessor in global fragment)
#ifdef _WIN32
#  include <io.h>
#  define PATH_SEP '\\'
#else
#  define PATH_SEP '/'
#endif

export module file_wrapper;  // ends the global module fragment

export enum class FileError {
    None,
    NotFound,
    PermissionDenied,
    Unknown
};

export struct FileResult {
    bool ok;
    FileError error;
    int error_code;  // raw errno value
};

export class FileReader {
    std::FILE* fp_ = nullptr;
public:
    FileReader() = default;
    ~FileReader() { close(); }

    // Non-copyable, movable
    FileReader(const FileReader&) = delete;
    FileReader& operator=(const FileReader&) = delete;
    FileReader(FileReader&& other) noexcept : fp_(other.fp_) { other.fp_ = nullptr; }

    FileResult open(const char* path) {
        close();
        fp_ = std::fopen(path, "r");  // C API from <cstdio>
        if (!fp_) {
            int err = errno;          // errno macro from <cerrno>
            FileError fe = FileError::Unknown;
            if (err == ENOENT)  fe = FileError::NotFound;       // macro
            if (err == EACCES)  fe = FileError::PermissionDenied; // macro
            return {false, fe, err};
        }
        return {true, FileError::None, 0};
    }

    void close() {
        if (fp_) { std::fclose(fp_); fp_ = nullptr; }
    }

    bool is_open() const { return fp_ != nullptr; }

    export char path_separator() { return PATH_SEP; } // uses platform macro
};
```

Notice that `errno`, `ENOENT`, `EACCES`, and `PATH_SEP` are all used inside the module implementation but never make it out to importers. The consumer only sees the `FileReader`, `FileError`, and `FileResult` types:

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
import file_wrapper;
#include <iostream>

int main() {
    FileReader reader;
    auto result = reader.open("nonexistent.txt");

    if (!result.ok) {
        std::cout << "Error: ";
        switch (result.error) {
        case FileError::NotFound:
            std::cout << "File not found"; break;
        case FileError::PermissionDenied:
            std::cout << "Permission denied"; break;
        default:
            std::cout << "Unknown error"; break;
        }
        std::cout << " (errno=" << result.error_code << ")\n";
    }

    // ENOENT, errno, PATH_SEP are NOT visible here
    // Only the exported types and functions are accessible
}
// Expected output (typical):
// Error: File not found (errno=2)
```

The global module fragment is doing exactly its job: C API macros go in, clean C++ types come out. Importers never deal with the raw macro world.

### Q2: Explain why macros defined after the module declaration are not exported

This is one of the most important conceptual points about modules. Let's see it concretely:

```cpp
module;
#define BEFORE_DECL 1      // Available in the module unit
export module example;

#define AFTER_DECL 2        // Also available in the module unit

export int use_macros() {
    return BEFORE_DECL + AFTER_DECL;  // Both work inside the module: returns 3
}
```

Both macros work fine *inside* the module. The function compiles and returns 3. But the reason macros don't export - even when defined after the module declaration - comes down to how the module system works at a fundamental level:

1. **Macros are preprocessor concepts.** Modules operate at the **compiler level**, after preprocessing. The compiler's module system doesn't know about macros.

2. **The BMI (Binary Module Interface) stores declarations, not preprocessor state.** When `import example;` reads the BMI, it gets:
   - Function signature: `int use_macros()`
   - But NOT: `#define BEFORE_DECL 1` or `#define AFTER_DECL 2`

3. **This is intentional.** The entire point of modules is to eliminate macro leakage. Think about the processing model:

```cpp
Header model:     source text --preprocessor--> expanded text --compiler--> object
                       ^
                  macros leak through #include

Module model:     module source --> BMI (declarations only) --> importer
                                     ^
                                no macro state stored
```

You can verify this by importing the module and checking whether the macros are visible:

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
import example;
#include <iostream>

int main() {
    std::cout << use_macros() << '\n';  // 3 - function works

    #ifdef BEFORE_DECL
        std::cout << "BEFORE_DECL visible\n";  // NOT printed
    #endif
    #ifdef AFTER_DECL
        std::cout << "AFTER_DECL visible\n";   // NOT printed
    #endif

    std::cout << "No macros leaked\n";
}
// Expected output:
// 3
// No macros leaked
```

The function works perfectly, but the macros are invisible. This is the clean separation modules give you.

### Q3: Show a case where a global module fragment is required to make a C API usable from a module

Wrapping SQLite3 is a perfect illustration because SQLite uses integer macros everywhere (`SQLITE_OK`, `SQLITE_ROW`, `SQLITE_DONE`) for its error-checking protocol. Without a global module fragment, those macros simply won't exist when you try to use them:

```cpp
// sqlite_wrapper.cppm
module;

// SQLite3 is a C library that uses macros extensively
#include <sqlite3.h>   // SQLITE_OK, SQLITE_ROW, SQLITE_DONE, etc.
#include <cstring>     // std::strlen

export module sqlite_wrapper;

import <string>;       // or #include <string> in global fragment
import <stdexcept>;

export class Database {
    sqlite3* db_ = nullptr;

public:
    Database() = default;
    ~Database() { if (db_) sqlite3_close(db_); }

    void open(const char* path) {
        int rc = sqlite3_open(path, &db_);
        if (rc != SQLITE_OK) {          // SQLITE_OK is a macro from sqlite3.h
            std::string msg = sqlite3_errmsg(db_);
            sqlite3_close(db_);
            db_ = nullptr;
            throw std::runtime_error("Failed to open DB: " + msg);
        }
    }

    void execute(const char* sql) {
        char* err_msg = nullptr;
        int rc = sqlite3_exec(db_, sql, nullptr, nullptr, &err_msg);
        if (rc != SQLITE_OK) {          // macro comparison
            std::string msg(err_msg);
            sqlite3_free(err_msg);
            throw std::runtime_error("SQL error: " + msg);
        }
    }

    bool is_open() const { return db_ != nullptr; }
};

// Without the global module fragment, SQLITE_OK, SQLITE_ROW etc.
// would not be available, making the C API unusable.
```

The difference in practice is stark:

| Without global module fragment | With global module fragment |
| --- | --- |
| `#include <sqlite3.h>` after `export module` | `#include <sqlite3.h>` in `module;` block |
| `SQLITE_OK` may not be defined | `SQLITE_OK` is guaranteed defined |
| Code doesn't compile | Code compiles correctly |
| C API unusable from module | C API fully usable |

The pattern applies to **any C library** that relies on macros: OpenSSL, zlib, POSIX, Win32, Vulkan, etc.

---

## Notes

- The global module fragment can **only** contain preprocessor directives - no actual C++ code.
- Always prefer wrapping C macros in `constexpr` variables or `enum` values in the exported interface.
- Multiple `#include` directives in the global fragment are fine - order matters as with normal `#include`.
- The global fragment is the recommended migration path for codebases heavily dependent on C libraries.
