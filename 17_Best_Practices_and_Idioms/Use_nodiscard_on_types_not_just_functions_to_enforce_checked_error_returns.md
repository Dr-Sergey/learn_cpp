# Use [[nodiscard]] on types (not just functions) to enforce checked error returns

**Category:** Best Practices and Idioms  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/nodiscard>  

---

## Topic Overview

You've probably seen `[[nodiscard]]` on functions, but putting it on a **type** is more powerful: every function that returns that type automatically inherits the warning. You don't have to remember to annotate each function individually - you annotate the type once, and the compiler enforces checking everywhere that type is returned.

```cpp
[[nodiscard]] struct Error { int code; };

Error do_work();    // automatically [[nodiscard]]!
Error validate();   // automatically [[nodiscard]]!
// Every function returning Error must have its result checked.
```

The nice part is that this scales: add one new function that returns `Error`, and the checking requirement comes along for free without any extra annotation on the function itself.

---

## Self-Assessment

### Q1: Mark a `Result<T,E>` type as `[[nodiscard]]` and show the warning

Here's a practical use of the pattern. The `Result<T,E>` type carries either a success value or an error, and discarding it silently is almost always a bug - you're throwing away information about whether the operation worked. Putting `[[nodiscard]]` on the type turns that class of bug into a compile-time warning.

```cpp
#include <iostream>
#include <string>
#include <variant>
#include <tuple>
#include <utility>

template<typename T, typename E>
[[nodiscard("Error result must be checked")]]
struct Result {
    std::variant<T, E> value;

    bool is_ok() const { return std::holds_alternative<T>(value); }
    T& unwrap() { return std::get<T>(value); }
    E& error() { return std::get<E>(value); }

    static Result ok(T val) { return Result{std::move(val)}; }
    static Result err(E e) { return Result{std::move(e)}; }
};

struct IoError {
    int code;
    std::string message;
};

Result<int, IoError> parse_number(const std::string& s) {
    try {
        return Result<int, IoError>::ok(std::stoi(s));
    } catch (...) {
        return Result<int, IoError>::err({-1, "parse failed: " + s});
    }
}

int main() {
    // GOOD: checking the result
    auto r1 = parse_number("42");
    if (r1.is_ok())
        std::cout << "Parsed: " << r1.unwrap() << '\n';

    auto r2 = parse_number("abc");
    if (!r2.is_ok())
        std::cout << "Error: " << r2.error().message << '\n';

    // BAD: this line would trigger a warning:
    // parse_number("99");  // warning: ignoring return value of type 'Result'
    //                      // with attribute 'nodiscard': Error result must be checked
}
// Expected output:
// Parsed: 42
// Error: parse failed: abc
```

That commented-out line at the bottom is the important one. Calling `parse_number` and throwing away the result is now a compiler warning - and with `-Werror` it becomes an error - without you having to do anything more than put `[[nodiscard]]` on the type definition.

### Q2: Explain why `[[nodiscard("reason")]]` is better than plain `[[nodiscard]]`

The plain `[[nodiscard]]` attribute generates a generic warning that just names the type. That's often enough to understand the problem, but when there are multiple `[[nodiscard]]` types in a codebase, a custom message tells the developer exactly why discarding is dangerous for this particular type.

```cpp
#include <iostream>

// Without message:
[[nodiscard]]
struct ErrorCode {
    int code;
};
// Warning: "ignoring return value of type 'ErrorCode'"
// Not very helpful. WHY must I check it?

// With message (C++20):
[[nodiscard("Ignoring error codes leads to silent failures")]]
struct StatusCode {
    int code;
    bool is_ok() const { return code == 0; }
};
// Warning: "ignoring return value of type 'StatusCode':
//           Ignoring error codes leads to silent failures"
// Much clearer!

StatusCode connect(const char* host) {
    return StatusCode{0};
}

[[nodiscard("Memory leak if allocation result is discarded")]]
struct Buffer {
    int* data;
    size_t size;
};

Buffer allocate(size_t n) {
    return Buffer{new int[n], n};
}

int main() {
    auto s = connect("localhost");  // OK: result used
    if (s.is_ok())
        std::cout << "Connected\n";

    auto b = allocate(100);  // OK: result used
    std::cout << "Allocated " << b.size << " ints\n";
    delete[] b.data;

    // connect("server");  // WARNING with custom message!
    // allocate(50);       // WARNING: Memory leak if discarded!
}
// Expected output:
// Connected
// Allocated 100 ints
```

The `Buffer` example shows a second reason to use a custom message: the warning text can tell you what the consequence of discarding actually is ("Memory leak"), which is much more actionable than "you ignored a return value."

### Q3: Show how to suppress a `[[nodiscard]]` warning with `(void)` cast

Sometimes you genuinely want to call a function and discard the result - for example, a cleanup function where you've already handled the important work and the return value is advisory. The standard way to express "I know about this return value, I've decided to ignore it" is to cast to `void`.

```cpp
#include <iostream>

[[nodiscard("Check the error code")]]
struct Status {
    int code;
};

Status initialize() { return {0}; }
Status cleanup() { return {0}; }

int main() {
    // Method 1: Use the result (preferred)
    auto s = initialize();
    if (s.code != 0) return 1;

    // Method 2: Intentional discard with (void) cast
    // "I know this can fail, but I don't care in this context"
    (void)cleanup();  // suppresses the warning

    // Method 3: [[maybe_unused]] on the variable
    [[maybe_unused]] auto s2 = cleanup();

    // Method 4: std::ignore (C++17, for structured bindings)
    // auto [code] = cleanup();

    std::cout << "Done\n";
}
// Expected output:
// Done
```

The `(void)` cast is the idiomatic choice. It's brief, universally understood, and serves as a signal to code reviewers that the discard is intentional rather than accidental. `[[maybe_unused]]` on a local variable is an alternative when you want to keep the value in scope for debugging purposes.

---

## Notes

- `[[nodiscard]]` on types is available since C++17; the `("reason")` form since C++20.
- Apply to: error types, RAII guards, allocation results, lock types - anything where silently discarding the return value is almost always a bug.
- The standard library uses it: `std::expected`, `std::error_code`, `std::unique_lock`.
- `[[nodiscard]]` on constructors (C++20) warns if a temporary is created and immediately discarded.
