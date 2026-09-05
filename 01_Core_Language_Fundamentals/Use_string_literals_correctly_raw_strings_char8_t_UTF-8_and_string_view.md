# Use string literals correctly: raw strings, char8_t, UTF-8, and string_view

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/string_literal>  

---

## Topic Overview

C++ has a rich set of string literal types. Understanding them prevents encoding bugs, dangling references, and unnecessary copies.

### String Literal Types Summary

| Literal | Type | Standard | Example |
| --- | --- | --- | --- |
| `"hello"` | `const char[6]` | C++98 | Regular string |
| `L"hello"` | `const wchar_t[6]` | C++98 | Wide string |
| `u8"hello"` | `const char8_t[6]` (C++20) | C++11/20 | UTF-8 |
| `u"hello"` | `const char16_t[6]` | C++11 | UTF-16 |
| `U"hello"` | `const char32_t[6]` | C++11 | UTF-32 |
| `R"(...)"` | `const char[N]` | C++11 | Raw string |

### Raw String Literals (C++11)

Raw strings don't process escape sequences - what you see is what you get. This is invaluable for regex patterns, file paths, and embedded multiline content:

```cpp
// Regular string: backslashes and quotes need escaping
std::string path = "C:\\Users\\name\\file.txt";
std::string json = "{\"key\": \"value\"}";

// Raw string: no escaping needed!
std::string path_r = R"(C:\Users\name\file.txt)";
std::string json_r = R"({"key": "value"})";

// Multi-line works naturally:
std::string html = R"(
<html>
  <body>
    <p>Hello, World!</p>
  </body>
</html>
)";

// Custom delimiter for strings that contain ")":
std::string tricky = R"delim(She said "hello" and (waved))delim";
// Without delim, the )" inside would end the literal prematurely
```

### char8_t and UTF-8 (C++20)

C++20 introduced `char8_t` as a distinct type for UTF-8 data. This was a deliberate breaking change - the goal is to make UTF-8 strings type-safe so you can't accidentally mix them with implementation-defined `char` encodings:

```cpp
// Before C++20: u8"..." produces const char[]
// After C++20: u8"..." produces const char8_t[]

auto s1 = u8"hello";  // C++20: const char8_t[6]
// std::string s2 = u8"hello";  // ERROR in C++20: char8_t != char

// To use with std::string, you need a cast:
std::string s3(reinterpret_cast<const char*>(u8"hello"));

// Or use std::u8string:
std::u8string s4 = u8"hello";  // OK

// Regular "hello" is still const char[] - and may or may not be UTF-8
// depending on source file encoding and compiler settings
```

### std::string_view (C++17)

A non-owning view into a string - no allocation, no copy. It accepts string literals, `std::string`, and substrings all through the same parameter type:

```cpp
#include <string_view>

void greet(std::string_view name) {  // accepts string, literal, or view
    std::cout << "Hello, " << name << "\n";
}

std::string s = "Alice";
greet(s);           // OK: string -> string_view (implicit)
greet("Bob");       // OK: literal -> string_view (no copy!)
greet(s.substr(0, 3));  // OK: returns a string (copy)

std::string_view sv = "Charlie";  // Points to static storage - fine
```

**DANGER: string_view does NOT own the data.** If the underlying string goes away, the view becomes a dangling pointer:

```cpp
std::string_view dangerous() {
    std::string local = "hello";
    return local;  // DANGLING: local destroyed, view points to freed memory
}

std::string_view also_bad;
{
    std::string temp = "world";
    also_bad = temp;  // view points to temp's data
}
// also_bad is now dangling - temp is destroyed!
```

**string_view is NOT null-terminated.** This trips people up when passing views to C APIs:

```cpp
std::string full = "Hello, World!";
std::string_view sv = full;
std::string_view sub = sv.substr(0, 5);  // "Hello" - NO null terminator!

// DANGER: passing to C API that expects null-terminated string
// printf("%s", sub.data());  // May print garbage after "Hello"

// Safe: convert to string first
std::string safe(sub);
printf("%s", safe.c_str());  // OK: c_str() is null-terminated
```

---

## Self-Assessment

### Q1: Write a raw string literal containing both backslashes and newlines

Raw strings shine whenever you'd otherwise have a forest of backslashes. The delimiter form (`R"cpp(...)cpp"`) lets you embed even `)` characters safely:

```cpp
#include <iostream>
#include <string>

int main() {
    // 1. Basic raw string with backslashes
    std::string regex = R"(\d{3}-\d{3}-\d{4})";
    std::cout << "Regex: " << regex << "\n";
    // Output: \d{3}-\d{3}-\d{4}  (no escaping needed!)

    // 2. Multi-line raw string
    std::string sql = R"(
SELECT users.name, orders.total
FROM users
JOIN orders ON users.id = orders.user_id
WHERE orders.total > 100.00
ORDER BY orders.total DESC;
)";
    std::cout << "SQL:" << sql << "\n";

    // 3. Raw string with backslashes AND quotes AND parentheses
    std::string code = R"cpp(
#include <iostream>
int main() {
    std::cout << "Path: C:\Windows\System32" << "\n";
    int arr[] = {1, 2, 3};
    auto fn = [](int x) { return x * 2; };
}
)cpp";
    std::cout << "Code:" << code << "\n";

    // 4. Compared with regular string (same content):
    std::string path_regular = "C:\\Users\\name\\Documents\\file.txt";
    std::string path_raw     = R"(C:\Users\name\Documents\file.txt)";
    std::cout << "Equal: " << std::boolalpha << (path_regular == path_raw) << "\n";  // true

    // 5. Raw string with all prefixes
    auto raw_wide = LR"(Wide: C:\path)";     // const wchar_t[]
    auto raw_utf8 = u8R"(UTF8: C:\path)";    // const char8_t[] in C++20
    auto raw_utf16 = uR"(UTF16: C:\path)";   // const char16_t[]
}
```

### Q2: Explain the difference between `u8"hello"` and `"hello"` in C++20

**Answer:**

The key difference is about guarantees, not just spelling. Regular `"..."` may or may not be UTF-8 depending on your compiler flags; `u8"..."` is always UTF-8 by definition. The C++20 type change enforces this at the type-system level:

| Aspect | `"hello"` | `u8"hello"` |
| --- | --- | --- |
| Type (C++17) | `const char[6]` | `const char[6]` |
| Type (C++20) | `const char[6]` | `const char8_t[6]` |
| Encoding | Implementation-defined (usually UTF-8) | **Guaranteed UTF-8** |
| Works with `std::string` | Yes | No (C++20) |
| Works with `std::u8string` | No | Yes |

```cpp
#include <iostream>
#include <string>

int main() {
    // "hello" - type is const char[6]
    // Encoding depends on compiler and flags (-fexec-charset)
    const char* regular = "hello";
    std::string s1 = "hello";  // Always works

    // u8"hello" - type is const char8_t[6] in C++20
    // Encoding is ALWAYS UTF-8, guaranteed by the standard
    // const char8_t* utf8 = u8"hello";
    // std::string s2 = u8"hello";  // ERROR in C++20

    // The char8_t change in C++20 broke a lot of code:
    // Before C++20: u8"..." was const char[], same as "..."
    // After C++20:  u8"..." is const char8_t[], incompatible with char

    // Workaround if you need UTF-8 in std::string:
    std::string s3(reinterpret_cast<const char*>(u8"日本語"));

    // Or use the dedicated type:
    std::u8string s4 = u8"日本語";

    // Why does char8_t exist?
    // To make UTF-8 strings TYPE-SAFE:
    // - You can't accidentally mix char (unknown encoding) with char8_t (UTF-8)
    // - Overload resolution can distinguish: void process(const char*) vs process(const char8_t*)
}
```

### Q3: Show why `std::string_view` does not null-terminate and when that causes bugs

The bug here is subtle: `cout` uses the view's `.size()` to know when to stop, so it prints correctly - but a C API uses `strlen`-style scanning, which reads past the end of the view into adjacent memory:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <cstdio>
#include <cstring>

// Simulated C API that requires null-terminated string
void c_api_log(const char* msg) {
    std::printf("LOG: %s\n", msg);  // %s reads until \0
}

int main() {
    std::string full = "Hello, World!";

    // string_view::substr does NOT copy - returns another view
    std::string_view greeting = std::string_view(full).substr(0, 5);
    // greeting points to "Hello" but the byte after 'o' is ','  NOT '\0'

    std::cout << "View: " << greeting << "\n";  // OK: cout uses .size()
    std::cout << "Size: " << greeting.size() << "\n";  // 5

    // BUG: passing non-null-terminated view to C API
    // c_api_log(greeting.data());
    // Output might be: "LOG: Hello, World!" (reads past the view!)
    // Or worse: reads into unrelated memory

    // FIX 1: Convert to std::string (adds null terminator)
    std::string safe(greeting);
    c_api_log(safe.c_str());  // "LOG: Hello"

    // FIX 2: Use .data() only when view covers a null-terminated string
    std::string_view full_view = full;  // Views entire string including \0
    c_api_log(full_view.data());  // OK because full.c_str() is null-terminated

    // Another bug: dangling string_view
    std::string_view dangling;
    {
        std::string temp = "temporary";
        dangling = temp;
    }
    // dangling.data() points to freed memory - UB to read!
    // std::cout << dangling << "\n";  // UB

    // Safe pattern: string_view for function parameters, not storage
    // void print(std::string_view sv);  // Good: caller keeps string alive
    // std::string_view member_;         // Dangerous: who owns the data?
}
```

---

## Notes

- Use **raw string literals** for regex patterns, file paths, JSON, SQL, and multi-line strings - eliminates escape-sequence bugs.
- `char8_t` (C++20) is a breaking change - existing code using `std::string s = u8"...";` won't compile. Use `-fno-char8_t` as a migration flag.
- `std::string_view` is ideal for **function parameters** (accepts any string type without copying) but dangerous for **storage** (dangling risk).
- `string_view::substr()` returns another view (O(1), no allocation) while `string::substr()` returns a new string (O(n), allocates).
- Always convert `string_view` to `std::string` before passing to C APIs that expect null-terminated `const char*`.
- User-defined literal `"hello"sv` creates a `std::string_view`, while `"hello"s` creates a `std::string` - both require `using namespace std::literals;`.
