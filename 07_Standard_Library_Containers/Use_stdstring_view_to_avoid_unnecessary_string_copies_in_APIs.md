# Use std::string_view to avoid unnecessary string copies in APIs

**Category:** Standard Library Containers  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string_view>  

---

## Topic Overview

`std::string_view` (C++17) is a non-owning, lightweight reference to a contiguous sequence of characters. It has the same read-only interface as `std::string` but **never allocates memory**. It is made up of just a pointer and a size - 16 bytes on 64-bit systems. You use it as a function parameter type any time the function only needs to read characters, not own them.

### Why string_view

The problem with `const std::string&` is that it only handles `std::string` efficiently. Pass a string literal and you pay for a heap allocation to construct a temporary. `std::string_view` accepts everything without allocating:

```text
                       const std::string&          std::string_view
                       -----------------           ----------------
Accepts std::string    Yes                         Yes
Accepts "literal"      Yes (constructs temp!)      Yes (no copy, no alloc)
Accepts string_view    No (needs .data() hack)     Yes
Accepts substring      No (needs substr -> alloc)  Yes (just adjust ptr+len)
Memory allocation      Potentially                  Never
```

### Core API

Here is the essential usage: passing different string types to a single function, and creating substrings without allocation:

```cpp
#include <string_view>
#include <string>
#include <iostream>

// === Use string_view in function parameters ===
void print_name(std::string_view name) {    // no copy, no alloc
    std::cout << "Name: " << name << "\n";
    std::cout << "Length: " << name.size() << "\n";
    std::cout << "First: " << name.front() << "\n";
}

int main() {
    std::string s = "Alice";
    const char* c = "Bob";
    std::string_view sv = "Charlie";

    print_name(s);        // no copy (implicit conversion)
    print_name(c);        // no copy (implicit conversion)
    print_name(sv);       // no copy
    print_name("David");  // no copy (no temp string created!)

    // === Substring without allocation ===
    std::string_view full = "Hello, World!";
    std::string_view hello = full.substr(0, 5);     // just ptr+len, no alloc
    std::string_view world = full.substr(7, 5);     // just ptr+len, no alloc
    // Compare: std::string::substr ALWAYS allocates a new string

    // === Trimming operations (no allocation) ===
    std::string_view padded = "   trimmed   ";
    padded.remove_prefix(3);   // modifies the VIEW, not the data
    padded.remove_suffix(3);
    std::cout << "[" << padded << "]\n";  // [trimmed]

    return 0;
}
```

`remove_prefix` and `remove_suffix` just adjust the internal pointer and size - no bytes are moved or copied.

### Key Rules

| Rule | Details |
| --- | --- |
| **Non-owning** | Does not manage lifetime of underlying data |
| **Read-only** | No `operator[]` modification, no `push_back`, etc. |
| **Not null-terminated** | `data()` may not be null-terminated (e.g., after `remove_suffix`) |
| **Implicit from string** | `std::string` -> `string_view` is implicit |
| **Explicit to string** | `string_view` -> `std::string` requires explicit constructor |

---

## Self-Assessment

### Q1: Rewrite a function taking const std::string& to take std::string_view and verify no allocation occurs

The biggest win from `string_view` shows up when callers pass literals, substrings, or other non-`std::string` types. This example tracks allocations so you can see the difference directly:

```cpp
#include <iostream>
#include <string>
#include <string_view>

// === BEFORE: const std::string& ===
// This forces allocation when called with a string literal:
//   greet_old("World");  // constructs a temporary std::string -> heap allocation!
std::string greet_old(const std::string& name) {
    return "Hello, " + name + "!";
}

// === AFTER: std::string_view ===
// No temporary string created for ANY caller type
std::string greet_new(std::string_view name) {
    std::string result;
    result.reserve(7 + name.size() + 1);  // "Hello, " + name + "!"
    result += "Hello, ";
    result += name;     // string_view appends to string without allocation
    result += "!";
    return result;
}

// === Counting allocations (simplified) ===
struct AllocCounter {
    static inline int count = 0;
};

void* operator new(std::size_t sz) {
    AllocCounter::count++;
    return std::malloc(sz);
}

void operator delete(void* ptr) noexcept { std::free(ptr); }
void operator delete(void* ptr, std::size_t) noexcept { std::free(ptr); }

int main() {
    // Test with string literal
    AllocCounter::count = 0;
    auto r1 = greet_old("World");
    int old_allocs = AllocCounter::count;

    AllocCounter::count = 0;
    auto r2 = greet_new("World");
    int new_allocs = AllocCounter::count;

    std::cout << "greet_old(\"World\"): " << old_allocs << " allocations\n";
    std::cout << "greet_new(\"World\"): " << new_allocs << " allocations\n";
    // The old version needs an extra allocation for the temporary std::string parameter
    // The new version avoids it entirely

    // Test with std::string
    std::string name = "Alice";
    AllocCounter::count = 0;
    auto r3 = greet_old(name);
    old_allocs = AllocCounter::count;

    AllocCounter::count = 0;
    auto r4 = greet_new(name);
    new_allocs = AllocCounter::count;

    std::cout << "greet_old(string):  " << old_allocs << " allocations\n";
    std::cout << "greet_new(string):  " << new_allocs << " allocations\n";
    // With std::string argument, both are similar - no temporary needed

    // Test with substring (biggest win)
    std::string full = "Hello World!!!!";
    std::string_view sv = std::string_view(full).substr(6, 5); // "World" - no alloc

    AllocCounter::count = 0;
    auto r5 = greet_new(sv);
    std::cout << "greet_new(substr_view): " << AllocCounter::count << " allocations\n";
    // Only allocates for the result string, never for the parameter

    return 0;
}
```

The rule of thumb that falls out of this: use `string_view` for any parameter where the function only reads characters. Use `const string&` only if the function needs to store or own the string.

### Q2: Show the danger of storing a string_view beyond the lifetime of its source string

This is where `string_view` bites people. Because it is non-owning, it is your responsibility to ensure the source data outlives the view. The compiler will not warn you when it doesn't:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <vector>

// === DANGEROUS: storing string_view as a member ===
class Logger {
    std::string_view name_;  // WARNING: non-owning!
public:
    Logger(std::string_view name) : name_(name) {}
    void log(std::string_view msg) {
        std::cout << "[" << name_ << "] " << msg << "\n";
    }
};

std::string_view dangerous_function() {
    std::string local = "temporary";
    return local;  // WARNING: local destroyed, view points to freed memory
}

std::string_view also_dangerous() {
    return std::string("also temporary");  // WARNING: temporary destroyed at semicolon
}

int main() {
    // === Bug 1: dangling from local variable ===
    std::string_view sv = dangerous_function();
    // sv.data() points to destroyed stack memory!
    // std::cout << sv << "\n";  // UNDEFINED BEHAVIOR

    // === Bug 2: dangling from temporary ===
    std::string_view sv2 = also_dangerous();
    // std::cout << sv2 << "\n";  // UNDEFINED BEHAVIOR

    // === Bug 3: dangling from container reallocation ===
    std::vector<std::string> names = {"Alice"};
    std::string_view first = names[0];  // points to names[0]'s buffer
    names.push_back("Bob");             // may reallocate -> names[0] moves!
    // first now dangles if reallocation happened
    // std::cout << first << "\n";  // UNDEFINED BEHAVIOR (maybe)

    // === Bug 4: storing string_view in a class ===
    Logger* logger;
    {
        std::string service_name = "AuthService";
        logger = new Logger(service_name);
    }  // service_name destroyed here
    // logger->log("user login");  // UNDEFINED BEHAVIOR: name_ dangles

    // === SAFE patterns ===
    // 1. Use string_view only for parameters (transient use):
    auto print = [](std::string_view s) { std::cout << s << "\n"; };
    print("safe");  // string literal lives for program duration

    // 2. Store std::string in classes, accept string_view in constructors:
    // class SafeLogger {
    //     std::string name_;  // OWNING
    //     SafeLogger(std::string_view name) : name_(name) {}  // copies into string
    // };

    // 3. string_view from string literals is always safe:
    std::string_view safe_sv = "literals live forever";  // OK

    // 4. Convert to string when you need ownership:
    std::string owned(sv);  // explicit conversion copies the data

    delete logger;
    return 0;
}
```

The reason this trips people up is that `string_view` looks and acts exactly like a string, so it is tempting to store it in a struct or return it from a function the same way. The safe mental model: treat `string_view` like a raw pointer with a size - it can dangle, and the compiler will not catch it. String literals are the exception: they have static lifetime and are always safe to hold a `string_view` into.

### Q3: Explain why string_view::data() is not necessarily null-terminated

This is an important C-interop trap. `std::string` always maintains a null terminator. `std::string_view` does not, and cannot:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <cstring>

void c_api_function(const char* str) {
    // Expects null-terminated C-string
    std::cout << "C API: " << str << " (len=" << std::strlen(str) << ")\n";
}

int main() {
    // === Case 1: string_view from string literal - IS null-terminated ===
    std::string_view sv1 = "Hello, World!";
    std::cout << "sv1.data(): " << sv1.data() << "\n";  // OK: underlying literal is null-terminated
    c_api_function(sv1.data());  // Happens to work, but not guaranteed!

    // === Case 2: string_view after remove_suffix - NOT null-terminated ===
    std::string_view sv2 = "Hello, World!";
    sv2.remove_suffix(8);  // sv2 is now "Hello" (5 chars)
    std::cout << "sv2: " << sv2 << "\n";       // prints "Hello" (operator<< respects size)
    std::cout << "sv2.data(): " << sv2.data() << "\n";  // prints "Hello, World!" <- STILL reads past the view!
    std::cout << "sv2.size(): " << sv2.size() << "\n";  // 5

    // c_api_function(sv2.data());  // WRONG: C function reads past the 5-char view

    // === Case 3: string_view from substr - NOT null-terminated ===
    std::string_view sv3 = std::string_view("Hello, World!").substr(0, 5);
    // sv3.data() points to 'H', but there's no '\0' after 'o'
    // The 'o' is followed by ',' in memory
    std::cout << "sv3.data() raw: " << sv3.data() << "\n";  // "Hello, World!" - reads too far!

    // === Case 4: string_view from arbitrary memory ===
    char buffer[] = {'A', 'B', 'C'};  // no null terminator!
    std::string_view sv4(buffer, 3);
    std::cout << "sv4: " << sv4 << "\n";  // "ABC" (safe: operator<< uses size)
    // c_api_function(sv4.data());  // UNDEFINED BEHAVIOR: no null terminator

    // === Safe way to pass string_view to C APIs ===
    // Convert to std::string (which IS null-terminated)
    std::string safe_copy(sv2);  // copies data + adds '\0'
    c_api_function(safe_copy.c_str());  // OK - safe

    // === Why this happens ===
    // string_view is {pointer, size}. It has NO control over what comes after size bytes.
    // remove_prefix/remove_suffix only adjust the pointer/size - they don't insert '\0'.
    // substr creates a new view with different ptr/size - no null terminator inserted.
    //
    // std::string ALWAYS maintains a null terminator.
    // string_view NEVER guarantees one.
    //
    // This is the fundamental trade-off: string_view avoids allocation at the cost of
    // not being compatible with C APIs that expect null-terminated strings.

    // === Summary ===
    std::cout << "\n=== Rules ===\n";
    std::cout << "1. NEVER pass string_view::data() to C APIs unless you know it's null-terminated\n";
    std::cout << "2. Use string_view with size-aware APIs (cout, string ctor, algorithms)\n";
    std::cout << "3. Convert to std::string when you need null termination\n";
    std::cout << "4. string_view from a literal IS null-terminated (but don't rely on it after trimming)\n";

    return 0;
}
```

The reason `string_view` can never guarantee null termination is structural: it is just `{const char* data, size_t size}` and has no control over what byte sits immediately after `data + size`. When created from a literal or a `std::string`, the memory happens to have a null there - but `remove_suffix` or `substr` only update the size field. Nothing moves in memory, so no null terminator is inserted. C++ APIs that use `size()` (streams, string constructors, algorithms) are safe; C APIs that read until `'\0'` are not.

---

## Notes

- **Use `string_view` for parameters**, `std::string` for data members and return values that outlive the function.
- **`std::string_view` literal:** `"hello"sv` (C++17, `using namespace std::string_view_literals`).
- **`constexpr`-friendly:** `string_view` operations are `constexpr` - usable in compile-time contexts.
- **C++20 additions:** `starts_with()`, `ends_with()` added to `string_view`. C++23 adds `contains()`.
- **Performance:** `string_view` is typically passed by value (16 bytes on 64-bit) - cheaper than `const string&` (which is a pointer plus indirection to heap).
- **Pitfall with `operator+`:** You can't do `sv + " suffix"` because `string_view` has no `operator+`. Convert first: `std::string(sv) + " suffix"`.
- **`std::string::operator string_view()`** is implicit. The reverse is explicit (`std::string(sv)`).
