# Understand [[fallthrough]] in switch statements and why it matters

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/fallthrough>  

---

## Topic Overview

In C++, when a `case` label in a `switch` statement doesn't end with `break`, execution **falls through** to the next case. This is often a bug. The C++17 `[[fallthrough]]` attribute tells the compiler (and readers) that the fallthrough is intentional.

### The Default Behavior Problem

Without any annotation, the compiler can't tell whether a missing `break` is a deliberate design choice or a typo:

```cpp
switch (x) {
    case 1:
        do_something();
        // No break! Falls through to case 2 -- is this a bug?
    case 2:
        do_other();
        break;
}
```

With `-Wextra` or `-Wimplicit-fallthrough`, the compiler warns about the missing `break`. But sometimes fallthrough is intentional.

### The [[fallthrough]] Attribute (C++17)

Add `[[fallthrough]];` right before the next case label to silence the warning and document the intent:

```cpp
switch (x) {
    case 1:
        do_something();
        [[fallthrough]];     // tells compiler: this is intentional
    case 2:
        do_other();
        break;
}
// No warning for the fallthrough from case 1 to case 2
```

### Rules for [[fallthrough]]

- Must appear as a **statement** immediately before a `case` or `default` label.
- Must be followed by a semicolon: `[[fallthrough]];`
- Cannot be used on the **last** case/default (nowhere to fall through to).
- Only suppresses the warning for **that specific fallthrough**.

### Common Pattern: Shared Processing

Fallthrough is particularly natural when higher-severity cases need to do everything a lower-severity case does, plus something more. Here is a logging example where `Fatal` funnels through `Error`, which funnels through `Warning`:

```cpp
enum class LogLevel { Debug, Info, Warning, Error, Fatal };

void handle_log(LogLevel level, const std::string& msg) {
    switch (level) {
        case LogLevel::Fatal:
            notify_admin(msg);
            [[fallthrough]];       // Fatal also does Error processing
        case LogLevel::Error:
            log_to_file(msg);
            [[fallthrough]];       // Error also does Warning processing
        case LogLevel::Warning:
            increment_warning_count();
            break;
        case LogLevel::Info:
        case LogLevel::Debug:      // empty cases don't need [[fallthrough]]
            std::cout << msg << "\n";
            break;
    }
}
```

Notice the two empty cases at the bottom - when a case has no code of its own before the next label, `[[fallthrough]]` is not needed.

---

## Self-Assessment

### Q1: Write a switch with intentional fallthrough and add [[fallthrough]] to suppress the warning

The permission model below is a textbook use of fallthrough: each level includes all the capabilities of the level below it:

```cpp
#include <iostream>
#include <string>

enum class Permission { None, Read, Write, Admin };

std::string describe_access(Permission p) {
    std::string abilities;

    switch (p) {
        case Permission::Admin:
            abilities += "manage users, ";
            [[fallthrough]];           // Admin can also write
        case Permission::Write:
            abilities += "write data, ";
            [[fallthrough]];           // Writers can also read
        case Permission::Read:
            abilities += "read data";
            break;
        case Permission::None:
            abilities = "no access";
            break;
    }

    return abilities;
}

int main() {
    std::cout << "Admin: " << describe_access(Permission::Admin) << "\n";
    // "manage users, write data, read data"

    std::cout << "Write: " << describe_access(Permission::Write) << "\n";
    // "write data, read data"

    std::cout << "Read:  " << describe_access(Permission::Read) << "\n";
    // "read data"

    std::cout << "None:  " << describe_access(Permission::None) << "\n";
    // "no access"
}
```

**How it works:**

- Each permission level accumulates the abilities of all lower levels via intentional fallthrough.
- `[[fallthrough]]` suppresses compiler warnings while documenting the intent.
- Compile with `-Wimplicit-fallthrough` to verify only un-annotated fallthroughs trigger warnings.

### Q2: Show a real-world case where accidental fallthrough introduced a bug

This is what unintentional fallthrough looks like in practice - the `Ok` case falls straight into the `NotFound` handler:

```cpp
#include <iostream>

// Bug: HTTP status code handler with accidental fallthrough
enum class HttpStatus { Ok = 200, NotFound = 404, ServerError = 500, Unknown = 0 };

void handle_response_BUGGY(HttpStatus status) {
    switch (status) {
        case HttpStatus::Ok:
            std::cout << "Success! Processing response body.\n";
            // BUG: forgot break! Falls through to NotFound!
        case HttpStatus::NotFound:
            std::cout << "Resource not found. Showing 404 page.\n";
            break;
        case HttpStatus::ServerError:
            std::cout << "Server error. Retrying...\n";
            break;
        default:
            std::cout << "Unknown status.\n";
            break;
    }
}

// Fixed version:
void handle_response_FIXED(HttpStatus status) {
    switch (status) {
        case HttpStatus::Ok:
            std::cout << "Success! Processing response body.\n";
            break;    // FIX: added break
        case HttpStatus::NotFound:
            std::cout << "Resource not found. Showing 404 page.\n";
            break;
        case HttpStatus::ServerError:
            std::cout << "Server error. Retrying...\n";
            break;
        default:
            std::cout << "Unknown status.\n";
            break;
    }
}

int main() {
    std::cout << "=== BUGGY ===\n";
    handle_response_BUGGY(HttpStatus::Ok);
    // Prints BOTH "Success!" AND "Resource not found" -- wrong!

    std::cout << "\n=== FIXED ===\n";
    handle_response_FIXED(HttpStatus::Ok);
    // Prints only "Success!" -- correct
}
```

**The real-world impact:**

- The Apple "goto fail" SSL bug (2014) was caused by a similar control-flow mistake.
- `-Wimplicit-fallthrough` (enabled by `-Wextra` in GCC/Clang) catches these bugs.
- Without the warning flag, this bug is silent and passes all simple tests (only triggers for `HttpStatus::Ok`).

### Q3: Compare [[fallthrough]] with a goto-based equivalent and explain readability tradeoffs

**Answer:**

```cpp
// Approach 1: [[fallthrough]] (recommended)
switch (cmd) {
    case Command::Save:
        save_file();
        [[fallthrough]];    // clear, localized, standard
    case Command::Validate:
        validate();
        break;
}

// Approach 2: goto (legacy C-style)
switch (cmd) {
    case Command::Save:
        save_file();
        goto validate_step;    // jumps out of switch context
    case Command::Validate:
    validate_step:
        validate();
        break;
}

// Approach 3: Refactored to avoid fallthrough entirely (often best)
if (cmd == Command::Save) {
    save_file();
}
if (cmd == Command::Save || cmd == Command::Validate) {
    validate();
}
```

The three approaches trade off clarity against flexibility. Here is a summary:

**Readability comparison:**

| Approach | Pros | Cons |
| --- | --- | --- |
| `[[fallthrough]]` | Standard, compiler-checked, local scope | Reader must understand fallthrough semantics |
| `goto` | Works in C and pre-C++17 | Non-local control flow, harder to reason about, `goto` stigma |
| Refactored `if` | No fallthrough needed, clearest intent | May duplicate conditions, less efficient for many cases |

**Recommendation:** Use `[[fallthrough]]` for intentional fallthrough in switch statements. Prefer refactoring to eliminate fallthrough when the logic is complex. Avoid `goto` in modern C++.

---

## Notes

- Empty case labels don't need `[[fallthrough]]`: `case 1: case 2: case 3: handle(); break;` is fine.
- GCC's `-Wimplicit-fallthrough=5` is the strictest level - only accepts the standard `[[fallthrough]]` attribute.
- MSVC uses `/W4` to enable the equivalent warning.
- Some coding standards (MISRA, CERT) prohibit fallthrough entirely - even with `[[fallthrough]]`.
