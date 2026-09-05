# Use std::system_error and its integration with std::error_code

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/error/system_error>  

---

## Topic Overview

`std::system_error` is an **exception class** that wraps a `std::error_code`, bridging the gap between error-code-based APIs (C, POSIX, Win32) and C++ exception handling. When a system call fails, you throw a `system_error` carrying the error code and a human-readable description.

### Class Hierarchy

It helps to know where `system_error` sits in the exception hierarchy, because you can catch it as a `std::runtime_error` if you do not need the error code details:

```cpp
std::exception
  +-- std::runtime_error
        +-- std::system_error          <- wraps error_code
              +-- std::ios_base::failure (C++11)
              +-- std::filesystem::filesystem_error (C++17)
```

### Construction

There are three common ways to construct a `system_error`. The most important rule is to capture `errno` immediately after the failing call - any subsequent function call (even an innocent one) may overwrite it:

```cpp
#include <system_error>
#include <cerrno>

// From errno + system_category
throw std::system_error(errno, std::system_category(), "open() failed");

// From errc (portable)
throw std::system_error(
    std::make_error_code(std::errc::no_such_file_or_directory),
    "config file missing"
);

// From error_code directly
std::error_code ec(EACCES, std::generic_category());
throw std::system_error(ec, "cannot access resource");
```

### Catching and Inspecting

When you catch a `system_error`, you get access to both a human-readable message and the structured error code. The portable comparison through `std::errc` is particularly useful when you want platform-independent error handling:

```cpp
try {
    // ... code that throws system_error ...
} catch (const std::system_error& e) {
    std::cout << "what():    " << e.what()            << "\n";
    std::cout << "code():    " << e.code().value()    << "\n";
    std::cout << "message(): " << e.code().message()  << "\n";
    std::cout << "category:  " << e.code().category().name() << "\n";

    // Portable comparison via errc:
    if (e.code() == std::errc::permission_denied)
        std::cout << "-> permission denied\n";
}
```

### `system_error` vs `error_code` - When to Use Which

| Use Case | Mechanism | Rationale |
| --- | --- | --- |
| Single syscall in a small wrapper | `throw system_error` | Exception propagates naturally up the call stack |
| Batch file operations | `error_code` parameter | Avoid throw/catch overhead per iteration |
| Library API (dual interface) | Provide both overloads | Let the caller choose (like `<filesystem>`) |
| Performance-critical inner loop | `error_code` only | Zero overhead from exceptions |

---

## Self-Assessment

### Q1: Throw a `std::system_error` from a failed POSIX call and catch it to extract the error message

**Solution - Wrapping `open()` and `read()`:**

The pattern here is straightforward: call the C function, check the return value, and if it failed, throw immediately while `errno` still holds the right value. Notice how `read()` saves `errno` before calling `close()` - that extra call would overwrite it:

```cpp
#include <iostream>
#include <system_error>
#include <cerrno>
#include <cstring>
#include <string>

// POSIX headers (Linux/macOS)
#ifdef _WIN32
  #include <io.h>
  #include <fcntl.h>
#else
  #include <unistd.h>
  #include <fcntl.h>
#endif

void read_file(const std::string& path) {
    int fd = ::open(path.c_str(), O_RDONLY);
    if (fd == -1) {
        // errno is set by open() - capture it IMMEDIATELY
        throw std::system_error(errno, std::system_category(),
                                "open(\"" + path + "\")");
    }

    char buf[256];
    ssize_t n = ::read(fd, buf, sizeof(buf));
    if (n == -1) {
        int saved_errno = errno;
        ::close(fd);
        throw std::system_error(saved_errno, std::system_category(),
                                "read(\"" + path + "\")");
    }

    ::close(fd);
    std::cout << "Read " << n << " bytes from " << path << "\n";
}

int main() {
    try {
        read_file("/nonexistent/path/file.txt");
    } catch (const std::system_error& e) {
        // Extract rich error information:
        std::cout << "what():     " << e.what() << "\n";
        std::cout << "value():    " << e.code().value() << "\n";
        std::cout << "message():  " << e.code().message() << "\n";
        std::cout << "category(): " << e.code().category().name() << "\n";

        // Portable check - works across OSes:
        if (e.code() == std::errc::no_such_file_or_directory)
            std::cout << "-> File does not exist (portable check).\n";
    }
}
// Expected output (Linux):
//   what():     open("/nonexistent/path/file.txt"): No such file or directory
//   value():    2
//   message():  No such file or directory
//   category(): system
//   -> File does not exist (portable check).
```

Notice that `what()` automatically concatenates your context message with the system's error description. That combined string is what makes `system_error` so useful in log output - you get the full picture from a single call.

---

### Q2: Show how `system_error::code()` returns an `error_code` comparable with `std::errc` values

**Solution - `code()` and Portable Comparison:**

The comparison `ec == std::errc::permission_denied` works across platforms because the category's `equivalent()` method maps the platform-specific integer to a portable `errc` condition. You write one comparison and it works on Linux, macOS, and Windows:

```cpp
#include <iostream>
#include <system_error>
#include <cerrno>
#include <fstream>

void demonstrate_code_comparison() {
    // Simulate various system errors and compare portably
    std::error_code codes[] = {
        {EACCES,  std::system_category()},   // Permission denied
        {ENOENT,  std::system_category()},   // No such file
        {EEXIST,  std::system_category()},   // File exists
        {ENOMEM,  std::system_category()},   // Out of memory
    };

    for (const auto& ec : codes) {
        std::cout << "Code " << ec.value()
                  << " [" << ec.category().name() << "]: "
                  << ec.message() << "\n";

        // Portable comparison using std::errc:
        if (ec == std::errc::permission_denied)
            std::cout << "  -> PORTABLE: permission denied\n";
        else if (ec == std::errc::no_such_file_or_directory)
            std::cout << "  -> PORTABLE: file not found\n";
        else if (ec == std::errc::file_exists)
            std::cout << "  -> PORTABLE: file already exists\n";
        else if (ec == std::errc::not_enough_memory)
            std::cout << "  -> PORTABLE: out of memory\n";
    }
}

void catch_and_switch() {
    try {
        throw std::system_error(EACCES, std::system_category(), "resource access");
    } catch (const std::system_error& e) {
        // Switch on the portable error condition:
        const auto& ec = e.code();

        if (ec == std::errc::permission_denied) {
            std::cout << "Handle: ask user for elevated privileges\n";
        } else if (ec == std::errc::no_such_file_or_directory) {
            std::cout << "Handle: create the file\n";
        } else if (ec == std::errc::connection_refused) {
            std::cout << "Handle: retry with backoff\n";
        } else {
            std::cout << "Handle: unknown system error: " << ec.message() << "\n";
        }
    }
}

int main() {
    demonstrate_code_comparison();
    std::cout << "\n";
    catch_and_switch();
}
// Expected output (Linux):
//   Code 13 [system]: Permission denied
//     -> PORTABLE: permission denied
//   Code 2 [system]: No such file or directory
//     -> PORTABLE: file not found
//   Code 17 [system]: File exists
//     -> PORTABLE: file already exists
//   Code 12 [system]: Cannot allocate memory
//     -> PORTABLE: out of memory
//
//   Handle: ask user for elevated privileges
```

**How the comparison works:**

```cpp
ec == std::errc::permission_denied
  |
ec.category().equivalent(ec.value(), errc::permission_denied)
  |
system_category maps OS error code to generic_category
  |
On Linux: EACCES(13) maps to errc::permission_denied -> true
On Windows: ERROR_ACCESS_DENIED(5) maps to errc::permission_denied -> true
```

---

### Q3: Implement a C++ wrapper around a C file API that converts `errno` to `system_error` on failure

**Solution - RAII File Wrapper with `system_error`:**

This is what wrapping a C API in idiomatic C++ looks like. The constructor either succeeds or throws - callers never hold a half-constructed `File` object. Every subsequent operation follows the same pattern: check the return value, save `errno`, throw if needed:

```cpp
#include <iostream>
#include <system_error>
#include <string>
#include <vector>
#include <cerrno>
#include <cstdio>
#include <cstring>

class File {
    std::FILE* fp_ = nullptr;
    std::string path_;

public:
    // Open file - throws system_error on failure
    explicit File(const std::string& path, const char* mode = "r")
        : path_(path)
    {
        fp_ = std::fopen(path.c_str(), mode);
        if (!fp_)
            throw std::system_error(errno, std::system_category(),
                                    "fopen(\"" + path + "\", \"" + mode + "\")");
    }

    // RAII: close on destruction
    ~File() {
        if (fp_) std::fclose(fp_);
    }

    // Non-copyable, movable
    File(const File&) = delete;
    File& operator=(const File&) = delete;
    File(File&& other) noexcept : fp_(other.fp_), path_(std::move(other.path_)) {
        other.fp_ = nullptr;
    }

    // Read entire file into a string
    std::string read_all() {
        if (std::fseek(fp_, 0, SEEK_END) != 0)
            throw std::system_error(errno, std::system_category(),
                                    "fseek(\"" + path_ + "\")");

        long size = std::ftell(fp_);
        if (size < 0)
            throw std::system_error(errno, std::system_category(),
                                    "ftell(\"" + path_ + "\")");

        std::rewind(fp_);
        std::string content(static_cast<size_t>(size), '\0');
        size_t n = std::fread(content.data(), 1, content.size(), fp_);
        if (n != content.size() && std::ferror(fp_))
            throw std::system_error(errno, std::system_category(),
                                    "fread(\"" + path_ + "\")");

        content.resize(n);
        return content;
    }

    // Write data - throws on failure
    void write(const std::string& data) {
        size_t n = std::fwrite(data.data(), 1, data.size(), fp_);
        if (n != data.size())
            throw std::system_error(errno, std::system_category(),
                                    "fwrite(\"" + path_ + "\")");
    }

    const std::string& path() const { return path_; }
};

int main() {
    // === Write a file ===
    try {
        File out("test_output.txt", "w");
        out.write("Hello from RAII wrapper!\n");
        std::cout << "Written successfully.\n";
    } catch (const std::system_error& e) {
        std::cout << "Write error: " << e.what() << "\n";
    }

    // === Read the file back ===
    try {
        File in("test_output.txt", "r");
        std::string content = in.read_all();
        std::cout << "Read: " << content;
    } catch (const std::system_error& e) {
        std::cout << "Read error: " << e.what() << "\n";
    }

    // === Error case: open nonexistent file ===
    try {
        File missing("/no/such/path.txt");
    } catch (const std::system_error& e) {
        std::cout << "Error: " << e.what() << "\n";

        if (e.code() == std::errc::no_such_file_or_directory)
            std::cout << "-> File not found (portable)\n";
    }

    // Cleanup
    std::remove("test_output.txt");
}
// Expected output:
//   Written successfully.
//   Read: Hello from RAII wrapper!
//   Error: fopen("/no/such/path.txt", "r"): No such file or directory
//   -> File not found (portable)
```

**Pattern: errno -> system_error:**

```cpp
C function call -> check return value -> if error:
  int saved = errno;           // Save IMMEDIATELY (next call may change it)
  throw std::system_error(     // Throw with:
      saved,                   //   errno value
      std::system_category(),  //   OS error category
      "context message"        //   what happened
  );
```

---

## Notes

- **Save `errno` immediately** - any intervening function call (including `close()`, `std::string` allocation) may overwrite it.
- **`std::system_category()`** maps to OS errors: `errno` values on POSIX, `GetLastError()` values on Windows.
- **`std::generic_category()`** maps to portable `std::errc` values - use for cross-platform error creation.
- **`filesystem_error`** derives from `system_error` - catching `system_error` also catches filesystem errors.
- **Dual API pattern:** Provide a throwing version and an `error_code` overload, like the standard library does for `<filesystem>`.
- **`what()` includes both your context message and the system's error description** - e.g., `"open(\"/foo\"): No such file or directory"`.
