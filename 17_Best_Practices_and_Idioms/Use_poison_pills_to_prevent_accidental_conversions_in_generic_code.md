# Use poison pills to prevent accidental conversions in generic code

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/function#Deleted_functions>  

---

## Topic Overview

**Poison pills** use `= delete` to close off implicit conversion paths that the compiler would otherwise silently use. Instead of getting a quietly wrong result at runtime, the caller gets a compile error right at the call site - which is exactly where you want to catch the problem.

```cpp
void process(int n);              // intended
void process(double) = delete;    // poison: prevents process(3.14)
void process(bool) = delete;      // poison: prevents process(true)
```

The reason this matters is that C++ will happily convert `true` to `1`, `3.14` to `3`, and a single char to its ASCII value when looking for a matching overload. Deleted overloads participate in overload resolution and lose - so the compiler picks the deleted version, sees it's deleted, and issues an error instead of silently converting.

---

## Self-Assessment

### Q1: Delete a constructor overload to prevent implicit conversion

Here's the pattern applied to constructors. `UserId` wraps an `int`, but you don't want someone accidentally passing `true` or `3.14` and getting a valid-looking but wrong ID. Each deleted constructor blocks one specific implicit conversion path.

```cpp
#include <iostream>
#include <string>
#include <cstddef>
#include <utility>

class UserId {
    int id_;
public:
    explicit UserId(int id) : id_(id) {}

    // Poison pills: prevent accidental conversions
    UserId(double) = delete;        // no UserId(3.14)
    UserId(bool) = delete;          // no UserId(true)
    UserId(char) = delete;           // no UserId('A')
    UserId(const char*) = delete;   // no UserId("123")
    UserId(std::string) = delete;   // no UserId(std::string("5"))

    int value() const { return id_; }
};

class SafeString {
    std::string data_;
public:
    SafeString(const std::string& s) : data_(s) {}
    SafeString(std::string&& s) : data_(std::move(s)) {}

    // Prevent implicit conversion from numbers
    SafeString(int) = delete;
    SafeString(double) = delete;
    SafeString(char) = delete;      // single char is suspicious
    SafeString(bool) = delete;
    SafeString(std::nullptr_t) = delete;  // prevent null

    const std::string& str() const { return data_; }
};

int main() {
    UserId valid(42);                  // OK
    // UserId bad1(3.14);              // ERROR: deleted
    // UserId bad2(true);              // ERROR: deleted
    // UserId bad3('A');               // ERROR: deleted

    SafeString s("hello");              // OK: const char* -> string -> SafeString
    // SafeString bad4(42);             // ERROR: deleted
    // SafeString bad5(nullptr);        // ERROR: deleted

    std::cout << "UserId: " << valid.value() << '\n';
    std::cout << "String: " << s.str() << '\n';
}
// Expected output:
// UserId: 42
// String: hello
```

The reason you need to explicitly delete `bool`, `char`, `double`, and so on is that they can all implicitly convert to `int`. Without the deleted overloads, `UserId(true)` would compile successfully and silently produce `UserId(1)`.

### Q2: Use `= delete` on a template specialization to block specific types

The same technique works for function templates. You can delete a specific specialization to block one type, or use a constrained primary template to block an entire category of types.

```cpp
#include <iostream>
#include <type_traits>

// Generic serialization function
template<typename T>
void serialize(const T& value) {
    std::cout << "Serializing: " << value << '\n';
}

// Block specific types that shouldn't be serialized
template<> void serialize<bool>(const bool&) = delete;
// Reason: bool serialization is ambiguous (0/1 vs true/false)

template<> void serialize<void*>(void* const&) = delete;
// Reason: raw pointer serialization is meaningless

// For broader blocking, use a primary template with constraints
template<typename T>
requires std::is_pointer_v<T>
void serialize(const T&) = delete;  // block ALL pointer types

int main() {
    serialize(42);            // OK
    serialize(std::string("hello"));  // OK
    serialize(3.14);          // OK
    // serialize(true);       // ERROR: deleted specialization
    // int* p = nullptr;
    // serialize(p);          // ERROR: pointer serialization deleted
    std::cout << "Done\n";
}
// Expected output:
// Serializing: 42
// Serializing: hello
// Serializing: 3.14
// Done
```

The constrained `requires std::is_pointer_v<T>` version is more powerful than a single specialization because it covers every pointer type without having to enumerate them individually.

### Q3: Show how deleted overloads give better error messages than `static_assert`

Both `= delete` and `static_assert` can block undesired instantiations, but they fail at different points and produce different quality error messages. The deleted overload fails at overload resolution - before the function body is even examined - so the error points right at the call site. A `static_assert` fails inside the template body, which typically gives you a longer, more confusing error trace.

```cpp
#include <iostream>
#include <type_traits>

// Approach 1: static_assert - compiles body then fails
template<typename T>
void process_v1(T val) {
    static_assert(!std::is_same_v<T, bool>,
                  "bool not supported");
    std::cout << "Processing: " << val << '\n';
}

// Approach 2: deleted overload - fails at overload resolution
void process_v2(int val) {
    std::cout << "Processing int: " << val << '\n';
}
void process_v2(double val) {
    std::cout << "Processing double: " << val << '\n';
}
void process_v2(bool) = delete;  // clear error at call site

int main() {
    process_v2(42);        // OK: calls process_v2(int)
    process_v2(3.14);      // OK: calls process_v2(double)
    // process_v2(true);   // ERROR: "use of deleted function 'process_v2(bool)'"
    //                     // Error points to CALL SITE, not template body
    //                     // Much clearer than static_assert in template

    process_v1(42);        // OK
    // process_v1(true);   // Error in template body, harder to understand
}
// Expected output:
// Processing int: 42
// Processing double: 3.14
// Processing: 42
```

The table below summarizes when each technique has an advantage. In short: use `= delete` when you want a clean call-site error and SFINAE-friendliness; use `static_assert` when you want a custom diagnostic message.

**Comparison:**

| Feature | `= delete` | `static_assert` |
| --- | --- | --- |
| Error location | Call site | Template body |
| Error quality | "deleted function" - clear | Sometimes cryptic |
| Works in overload resolution | Yes (participates) | No (selected, then fails) |
| SFINAE-friendly | Yes | No |
| Can provide message | No (but error is clear) | Yes |

---

## Notes

- `= delete` participates in overload resolution - it's a better match than a conversion, so the compiler selects it and then reports the deletion rather than silently doing the conversion.
- The ranges library uses `void begin(auto&&) = delete;` as an ADL poison pill to prevent certain customization points from being found by accident.
- Combine with `explicit` constructors for maximum type safety.
- C++20 concepts can replace many poison pill patterns more elegantly: `requires std::integral<T>` expresses intent more clearly than a list of deleted overloads.
